use std::collections::BTreeMap;
use std::fs::File;
use std::io::Read;
use std::path::{Component, Path};

use anyhow::{Context, anyhow};
use encoding_rs::{Encoding, UTF_8, WINDOWS_1250};
use image::ImageReader;
use quick_xml::{Reader, events::Event};
use serde::Serialize;
use zip::ZipArchive;

#[derive(Debug, Clone, Serialize, PartialEq, Eq)]
pub enum FileVerdict {
    Clean,
    CleanWithActiveContent,
    Rejected,
    ScanFailed,
    Unrecognized,
}

#[derive(Debug, Clone, Serialize, PartialEq, Eq)]
pub enum ValidationReason {
    Allowed,
    ExtensionNotAllowed,
    MissingExtension,
    DoubleExtension,
    MimeMismatch,
    ExecutableDetected,
    ScriptDetected,
    InvalidFileStructure,
    MacroPresent,
    ActiveContentPresent,
    EmbeddedExecutable,
    UnsupportedFormat,
    ArchiveNotAllowed,
    ArchiveLimitExceeded,
    PasswordProtectedContent,
    ParserFailure,
}

#[derive(Debug, Clone, Default, Serialize, PartialEq, Eq)]
pub struct ValidationFlags {
    pub macro_enabled: bool,
    pub active_content: bool,
    pub embedded_file: bool,
    pub embedded_executable: bool,
    pub password_protected: bool,
}

#[derive(Debug, Clone, Serialize, PartialEq, Eq)]
pub struct ValidationReport {
    pub verdict: FileVerdict,
    pub score: i32,
    pub reasons: Vec<ValidationReason>,
    pub flags: ValidationFlags,
    pub detected_mime: String,
    pub normalized_extension: Option<String>,
    pub verified_mime: Option<String>,
}

impl ValidationReport {
    fn new(detected_mime: &str, extension: Option<String>) -> Self {
        Self {
            verdict: FileVerdict::Clean,
            score: 0,
            reasons: Vec::new(),
            flags: ValidationFlags::default(),
            detected_mime: detected_mime.to_string(),
            normalized_extension: extension,
            verified_mime: None,
        }
    }

    pub fn is_rejected(&self) -> bool {
        matches!(
            self.verdict,
            FileVerdict::Rejected | FileVerdict::ScanFailed | FileVerdict::Unrecognized
        )
    }

    pub fn failed() -> Self {
        let mut report = Self::new("", None);
        report.scan_failed(ValidationReason::ParserFailure);
        report
    }

    pub fn status(&self) -> &'static str {
        match self.verdict {
            FileVerdict::Clean | FileVerdict::CleanWithActiveContent => "ACCEPTED",
            FileVerdict::Rejected => "REJECTED",
            FileVerdict::Unrecognized => "UNRECOGNIZED",
            FileVerdict::ScanFailed => "VALIDATION_ERROR",
        }
    }

    fn unrecognized(&mut self) {
        self.reason(ValidationReason::UnsupportedFormat);
        self.verdict = FileVerdict::Unrecognized;
    }

    fn reason(&mut self, reason: ValidationReason) {
        if !self.reasons.contains(&reason) {
            self.reasons.push(reason);
        }
    }

    fn reject(&mut self, reason: ValidationReason) {
        self.reason(reason);
        self.verdict = FileVerdict::Rejected;
    }

    fn scan_failed(&mut self, reason: ValidationReason) {
        self.reason(reason);
        self.verdict = FileVerdict::ScanFailed;
    }

    fn active(&mut self, reason: ValidationReason, score: i32) {
        self.reason(reason);
        self.score += score;
        if self.verdict == FileVerdict::Clean {
            self.verdict = FileVerdict::CleanWithActiveContent;
        }
    }

    fn allowed(&mut self) {
        if self.reasons.is_empty() {
            self.reason(ValidationReason::Allowed);
        }
    }
}

#[derive(Debug, Clone)]
pub struct ValidationConfig {
    pub allow_missing_extension: bool,
    pub max_archive_entries: usize,
    pub max_archive_entry_size: u64,
    pub max_archive_total_size: u64,
    pub max_archive_path_depth: usize,
    pub max_image_pixels: u64,
    pub max_xml_size: u64,
}

impl Default for ValidationConfig {
    fn default() -> Self {
        Self {
            allow_missing_extension: false,
            max_archive_entries: 2_000,
            max_archive_entry_size: 200 * 1024 * 1024,
            max_archive_total_size: 500 * 1024 * 1024,
            max_archive_path_depth: 16,
            max_image_pixels: 100_000_000,
            max_xml_size: 1024 * 1024,
        }
    }
}

pub fn validate_file(
    path: &Path,
    file_name: &str,
    detected_mime: &str,
    config: &ValidationConfig,
) -> anyhow::Result<ValidationReport> {
    let extension = normalized_extension(file_name);
    let mut report = ValidationReport::new(detected_mime, extension.clone());

    validate_name(file_name, extension.as_deref(), config, &mut report);
    if report.is_rejected() {
        return Ok(report);
    }

    if validate_magic(path, &mut report).is_err() {
        report.scan_failed(ValidationReason::ParserFailure);
    }
    if report.is_rejected() {
        return Ok(report);
    }

    let Some(ext) = extension.as_deref() else {
        report.unrecognized();
        return Ok(report);
    };
    let detected = detected_mime.split(';').next().unwrap_or("").trim();
    let kind = format_kind(ext);
    let generic = detected.eq_ignore_ascii_case("application/octet-stream")
        || detected.eq_ignore_ascii_case("application/zip")
            && matches!(kind, Some(FormatKind::Ooxml(_)))
        || ["application/x-ole-storage", "application/x-cfb"]
            .iter()
            .any(|mime| detected.eq_ignore_ascii_case(mime))
            && matches!(kind, Some(FormatKind::LegacyOffice));
    if !generic && !matches!(kind, Some(FormatKind::Text)) {
        validate_mime(ext, detected, &mut report);
        if report.is_rejected() {
            return Ok(report);
        }
    }
    // An octet-stream/container result needs structural verification first.
    let mime = match format_kind(ext) {
        Some(FormatKind::Ooxml(kind)) => validate_ooxml(path, kind, config, &mut report),
        Some(FormatKind::Pdf) => validate_pdf(path, &mut report),
        Some(FormatKind::Image) => validate_image(path, config, &mut report),
        Some(FormatKind::LegacyOffice) => validate_legacy_office(path, config, &mut report),
        Some(FormatKind::Text) => validate_text(path, ext, detected_mime, &mut report),
        None => None,
    };
    if !report.is_rejected() {
        if let Some(mime) = mime {
            validate_mime(ext, &mime, &mut report);
            if !report.is_rejected() {
                report.verified_mime = Some(mime);
            }
        } else {
            report.unrecognized();
        }
    }

    report.allowed();
    Ok(report)
}

fn validate_name(
    file_name: &str,
    extension: Option<&str>,
    config: &ValidationConfig,
    report: &mut ValidationReport,
) {
    let parts = file_name.rsplit('/').next().unwrap_or(file_name);
    let extensions: Vec<String> = parts
        .split('.')
        .skip(1)
        .map(|p| p.to_ascii_lowercase())
        .collect();

    if extension.is_none() && !config.allow_missing_extension {
        report.reject(ValidationReason::MissingExtension);
        return;
    }

    if let Some(ext) = extension {
        if blocked_archive_extension(ext) {
            report.reject(ValidationReason::ArchiveNotAllowed);
            return;
        }
        if executable_extension(ext) {
            report.reject(ValidationReason::ExecutableDetected);
            return;
        }
        if script_extension(ext) {
            report.reject(ValidationReason::ScriptDetected);
            return;
        }
        if format_kind(ext).is_none() {
            report.reject(ValidationReason::ExtensionNotAllowed);
            return;
        }
    }

    if extensions.len() > 1
        && extensions.iter().take(extensions.len() - 1).any(|ext| {
            executable_extension(ext) || script_extension(ext) || blocked_archive_extension(ext)
        })
    {
        report.reject(ValidationReason::DoubleExtension);
    }
}

fn validate_magic(path: &Path, report: &mut ValidationReport) -> anyhow::Result<()> {
    let header = read_prefix(path, 4096)?;

    if header.starts_with(b"MZ")
        || is_pe(&header)
        || starts_with_any(
            &header,
            &[b"\x7FELF", b"\xCA\xFE\xBA\xBE", b"dex\n", b"\0asm", b"ITSF"],
        )
        || is_macho(&header)
        || is_lnk(&header)
    {
        report.reject(ValidationReason::ExecutableDetected);
    }

    Ok(())
}

fn validate_mime(ext: &str, detected_mime: &str, report: &mut ValidationReport) {
    if allowed_mimes()
        .get(ext)
        .is_some_and(|mimes| !mimes.iter().any(|m| mime_matches(detected_mime, m)))
    {
        report.reject(ValidationReason::MimeMismatch);
    }
}

fn validate_pdf(path: &Path, report: &mut ValidationReport) -> Option<String> {
    let document = match lopdf::Document::load(path) {
        Ok(document) => document,
        Err(_) => {
            report.reject(ValidationReason::InvalidFileStructure);
            return None;
        }
    };
    if document.is_encrypted() {
        report.flags.password_protected = true;
        report.reject(ValidationReason::PasswordProtectedContent);
        return None;
    }
    if document.catalog().is_err() || document.get_pages().is_empty() {
        report.reject(ValidationReason::InvalidFileStructure);
        return None;
    }
    let markers: &[&[u8]] = &[
        b"JavaScript",
        b"JS",
        b"OpenAction",
        b"AA",
        b"Launch",
        b"EmbeddedFile",
        b"RichMedia",
        b"XFA",
        b"AcroForm",
    ];
    for object in document.objects.values() {
        let dict = match object {
            lopdf::Object::Dictionary(dict) => dict,
            lopdf::Object::Stream(stream) => &stream.dict,
            _ => continue,
        };
        if dict.iter().any(|(key, value)| {
            markers.contains(&key.as_slice())
                || value.as_name().is_ok_and(|name| markers.contains(&name))
        }) {
            report.flags.active_content = true;
            report.active(ValidationReason::ActiveContentPresent, 20);
            break;
        }
    }
    Some("application/pdf".into())
}

fn validate_image(
    path: &Path,
    config: &ValidationConfig,
    report: &mut ValidationReport,
) -> Option<String> {
    let result = (|| -> anyhow::Result<_> {
        let reader = ImageReader::open(path)?.with_guessed_format()?;
        let format = reader
            .format()
            .ok_or_else(|| anyhow!("Unknown image format"))?;
        let (width, height) = reader.into_dimensions()?;
        Ok((format.to_mime_type(), width as u64 * height as u64))
    })();
    match result {
        Ok((mime, pixels)) if pixels > 0 && pixels <= config.max_image_pixels => Some(mime.into()),
        _ => {
            report.reject(ValidationReason::InvalidFileStructure);
            None
        }
    }
}

fn validate_legacy_office(
    path: &Path,
    config: &ValidationConfig,
    report: &mut ValidationReport,
) -> Option<String> {
    let mut compound = match cfb::open(path) {
        Ok(compound) => compound,
        Err(_) => {
            report.reject(ValidationReason::InvalidFileStructure);
            return None;
        }
    };
    let entries: Vec<_> = compound
        .walk()
        .take(config.max_archive_entries + 1)
        .map(|entry| (entry.path().to_path_buf(), entry.is_stream(), entry.len()))
        .collect();
    if entries.len() > config.max_archive_entries {
        report.reject(ValidationReason::ArchiveLimitExceeded);
        return None;
    }
    let mut mimes = Vec::new();
    for (path, is_stream, size) in &entries {
        let name = path.file_name().and_then(|s| s.to_str()).unwrap_or("");
        if name.eq_ignore_ascii_case("EncryptedPackage")
            || name.eq_ignore_ascii_case("EncryptionInfo")
        {
            report.flags.password_protected = true;
            report.reject(ValidationReason::PasswordProtectedContent);
            return None;
        }
        if name.eq_ignore_ascii_case("VBA") || name.eq_ignore_ascii_case("_VBA_PROJECT") {
            report.flags.macro_enabled = true;
            report.active(ValidationReason::MacroPresent, 20);
        }
        if !is_stream {
            continue;
        }
        if *size > config.max_archive_entry_size {
            report.reject(ValidationReason::ArchiveLimitExceeded);
            return None;
        }
        let mut header = [0u8; 8];
        let n = match compound
            .open_stream(path)
            .and_then(|mut stream| stream.read(&mut header))
        {
            Ok(n) => n,
            Err(_) => {
                report.scan_failed(ValidationReason::ParserFailure);
                return None;
            }
        };
        let has_stream = |expected: &str| {
            entries.iter().any(|(path, stream, _)| {
                *stream
                    && path.parent() == Some(Path::new("/"))
                    && path.file_name().is_some_and(|v| v == expected)
            })
        };
        // Identify only root document streams, not embedded documents.
        if path.parent() != Some(Path::new("/")) {
            continue;
        }
        let mime = match name {
            "WordDocument"
                if n >= 2
                    && header[..2] == [0xec, 0xa5]
                    && (has_stream("0Table") || has_stream("1Table")) =>
            {
                Some("application/msword")
            }
            "Workbook" | "Book"
                if n >= 4 && header[0] == 0x09 && [0x00, 0x02, 0x04, 0x08].contains(&header[1]) =>
            {
                Some("application/vnd.ms-excel")
            }
            "PowerPoint Document" if n >= 8 && has_stream("Current User") => {
                Some("application/vnd.ms-powerpoint")
            }
            _ => None,
        };
        if let Some(mime) = mime {
            mimes.push(mime);
        }
    }
    mimes.sort_unstable();
    mimes.dedup();
    if mimes.len() == 1 {
        Some(mimes[0].into())
    } else {
        report.unrecognized();
        None
    }
}

// Streaming decode: retain only a bounded sample for conservative Markdown hints.
fn text_sample(path: &Path, encoding: &'static Encoding) -> anyhow::Result<Option<String>> {
    let mut file = File::open(path)?;
    let mut decoder = encoding.new_decoder();
    let mut input = [0u8; 8192];
    let mut output = [0u8; 32768];
    let mut sample = String::new();
    loop {
        let n = file.read(&mut input)?;
        let mut offset = 0;
        loop {
            let (result, read, written, errors) =
                decoder.decode_to_utf8(&input[offset..n], &mut output, n == 0);
            if errors {
                return Ok(None);
            }
            let text = std::str::from_utf8(&output[..written])?;
            if text
                .chars()
                .any(|c| c.is_control() && !matches!(c, '\n' | '\r' | '\t' | '\u{000c}'))
            {
                return Ok(None);
            }
            if sample.len() < 65536 {
                let available = 65536 - sample.len();
                sample.extend(text.chars().scan(0usize, |count, c| {
                    *count += c.len_utf8();
                    (*count <= available).then_some(c)
                }));
            }
            offset += read;
            if result == encoding_rs::CoderResult::InputEmpty {
                break;
            }
        }
        if n == 0 {
            return Ok(Some(sample));
        }
    }
}

fn validate_text(
    path: &Path,
    ext: &str,
    detected_mime: &str,
    report: &mut ValidationReport,
) -> Option<String> {
    let result = (|| -> anyhow::Result<Option<String>> {
        let header = read_prefix(path, 4)?;
        let encoding = Encoding::for_bom(&header)
            .map(|(encoding, _)| encoding)
            .unwrap_or(UTF_8);
        let mut sample = text_sample(path, encoding)?;
        // Legacy Polish text without a BOM: explicit Windows-1250 fallback.
        if sample.is_none() && encoding == UTF_8 && !header.starts_with(&[0xef, 0xbb, 0xbf]) {
            sample = text_sample(path, WINDOWS_1250)?;
        }
        Ok(sample)
    })();
    match result {
        Ok(Some(sample)) => {
            let detected = detected_mime.split(';').next().unwrap_or("").trim();
            let mime = if !["text/plain", "text/markdown", "text/csv"]
                .iter()
                .any(|mime| detected.eq_ignore_ascii_case(mime))
            {
                if ext == "md"
                    && sample.lines().any(|line| {
                        let line = line.trim_start();
                        line.starts_with("# ")
                            || line.starts_with("## ")
                            || line.starts_with("### ")
                            || line.starts_with("```")
                            || line.starts_with("~~~")
                            || line.contains("](") && line.contains('[')
                    })
                {
                    "text/markdown"
                } else {
                    "text/plain"
                }
            } else {
                detected
            };
            Some(mime.to_string())
        }
        Ok(None) => {
            report.unrecognized();
            None
        }
        Err(_) => {
            report.scan_failed(ValidationReason::ParserFailure);
            None
        }
    }
}

fn validate_ooxml(
    path: &Path,
    kind: OoxmlKind,
    config: &ValidationConfig,
    report: &mut ValidationReport,
) -> Option<String> {
    let file = match File::open(path) {
        Ok(file) => file,
        Err(_) => {
            report.scan_failed(ValidationReason::ParserFailure);
            return None;
        }
    };
    let mut archive = match ZipArchive::new(file) {
        Ok(archive) => archive,
        Err(_) => {
            report.reject(ValidationReason::InvalidFileStructure);
            return None;
        }
    };
    if archive.len() > config.max_archive_entries {
        report.reject(ValidationReason::ArchiveLimitExceeded);
        return None;
    }
    let mut total_size = 0u64;
    let mut main_parts = 0;
    let mut content_types = None;
    let mut seen = std::collections::HashSet::new();
    for idx in 0..archive.len() {
        let mut entry = match archive.by_index(idx) {
            Ok(entry) => entry,
            Err(_) => {
                report.scan_failed(ValidationReason::ParserFailure);
                return None;
            }
        };
        if entry.encrypted() {
            report.flags.password_protected = true;
            report.reject(ValidationReason::PasswordProtectedContent);
            return None;
        }
        let name = entry.name().replace('\\', "/");
        if !safe_zip_path(&name, config.max_archive_path_depth) || !seen.insert(name.clone()) {
            report.reject(ValidationReason::InvalidFileStructure);
            return None;
        }
        total_size = total_size.saturating_add(entry.size());
        if entry.size() > config.max_archive_entry_size
            || total_size > config.max_archive_total_size
        {
            report.reject(ValidationReason::ArchiveLimitExceeded);
            return None;
        }
        if [
            "word/document.xml",
            "xl/workbook.xml",
            "ppt/presentation.xml",
        ]
        .contains(&name.as_str())
        {
            main_parts += 1;
        }
        if name == "[Content_Types].xml" {
            let mut xml = Vec::new();
            if entry
                .by_ref()
                .take(config.max_xml_size + 1)
                .read_to_end(&mut xml)
                .is_err()
            {
                report.scan_failed(ValidationReason::ParserFailure);
                return None;
            }
            if xml.len() as u64 > config.max_xml_size {
                report.reject(ValidationReason::ArchiveLimitExceeded);
                return None;
            }
            content_types = Some(xml);
        }
        if name.ends_with("vbaProject.bin") {
            report.flags.macro_enabled = true;
            report.active(ValidationReason::MacroPresent, 20);
        }
        if name.contains("/activeX/") || name.contains("/embeddings/") || name.contains("oleObject")
        {
            report.flags.embedded_file = true;
            report.active(ValidationReason::ActiveContentPresent, 15);
        }
        if suspicious_embedded_extension(&name) {
            report.flags.embedded_executable = true;
            report.reject(ValidationReason::EmbeddedExecutable);
            return None;
        }
        // Inspect a prefix even for large entries; never inflate a whole entry.
        let mut sample = Vec::new();
        if entry.by_ref().take(4096).read_to_end(&mut sample).is_err() {
            report.scan_failed(ValidationReason::ParserFailure);
            return None;
        }
        if name == kind.main_part() {
            let mut reader = Reader::from_reader(sample.as_slice());
            let root = loop {
                match reader.read_event() {
                    Ok(Event::Start(element) | Event::Empty(element)) => {
                        break Some(element.local_name().as_ref().to_vec());
                    }
                    Ok(Event::Decl(_) | Event::Comment(_) | Event::Text(_)) => continue,
                    _ => break None,
                }
            };
            let expected: &[u8] = match kind {
                OoxmlKind::Word => b"document",
                OoxmlKind::Excel => b"workbook",
                OoxmlKind::PowerPoint => b"presentation",
            };
            if root.as_deref() != Some(expected) {
                report.reject(ValidationReason::InvalidFileStructure);
                return None;
            }
        }
        if is_pe(&sample)
            || sample.starts_with(b"MZ")
            || sample.starts_with(b"\x7FELF")
            || is_macho(&sample)
            || is_lnk(&sample)
        {
            report.flags.embedded_executable = true;
            report.reject(ValidationReason::EmbeddedExecutable);
            return None;
        }
    }
    if main_parts != 1 || !seen.contains(kind.main_part()) {
        report.reject(ValidationReason::InvalidFileStructure);
        return None;
    }
    let Some(xml) = content_types else {
        report.reject(ValidationReason::InvalidFileStructure);
        return None;
    };
    let mime = ooxml_mime(&xml, kind);
    if mime.is_none() {
        report.unrecognized();
    }
    mime
}

fn ooxml_mime(xml: &[u8], kind: OoxmlKind) -> Option<String> {
    let mut reader = Reader::from_reader(xml);
    let mut mime = None;
    let mut root = false;
    let mut depth = 0usize;
    let mut closed = false;
    loop {
        let event = reader.read_event().ok()?;
        match event {
            Event::Start(ref element) | Event::Empty(ref element) => {
                if closed {
                    return None;
                }
                if matches!(event, Event::Start(_)) {
                    depth += 1;
                }
                if !root {
                    if element.local_name().as_ref() != b"Types" {
                        return None;
                    }
                    let name = element.name();
                    let mut namespace_key = b"xmlns".to_vec();
                    if let Some(colon) = name.as_ref().iter().position(|&b| b == b':') {
                        namespace_key.push(b':');
                        namespace_key.extend_from_slice(&name.as_ref()[..colon]);
                    }
                    let namespace = element.attributes().find_map(|attr| {
                        let attr = attr.ok()?;
                        (attr.key.as_ref() == namespace_key).then_some(attr.value.into_owned())
                    });
                    if namespace.as_deref()
                        != Some(
                            b"http://schemas.openxmlformats.org/package/2006/content-types"
                                .as_slice(),
                        )
                    {
                        return None;
                    }
                    if matches!(event, Event::Empty(_)) {
                        return None;
                    }
                    root = true;
                }
                if element.local_name().as_ref() == b"Override" {
                    let mut part = None;
                    let mut content = None;
                    for attr in element.attributes() {
                        let attr = attr.ok()?;
                        let value = attr
                            .decode_and_unescape_value(reader.decoder())
                            .ok()?
                            .into_owned();
                        match attr.key.as_ref() {
                            b"PartName" => part = Some(value),
                            b"ContentType" => content = Some(value),
                            _ => {}
                        }
                    }
                    if part.as_deref() == Some(&format!("/{}", kind.main_part())) {
                        if mime.is_some() {
                            return None;
                        }
                        mime = Some(match content.as_deref()? {
                            "application/vnd.openxmlformats-officedocument.wordprocessingml.document.main+xml" => "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
                            "application/vnd.ms-word.document.macroEnabled.main+xml" => "application/vnd.ms-word.document.macroenabled.12",
                            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet.main+xml" => "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                            "application/vnd.ms-excel.sheet.macroEnabled.main+xml" => "application/vnd.ms-excel.sheet.macroenabled.12",
                            "application/vnd.openxmlformats-officedocument.presentationml.presentation.main+xml" => "application/vnd.openxmlformats-officedocument.presentationml.presentation",
                            "application/vnd.ms-powerpoint.presentation.macroEnabled.main+xml" => "application/vnd.ms-powerpoint.presentation.macroenabled.12",
                            "application/vnd.openxmlformats-officedocument.presentationml.slideshow.main+xml" => "application/vnd.openxmlformats-officedocument.presentationml.slideshow",
                            "application/vnd.ms-powerpoint.slideshow.macroEnabled.main+xml" => "application/vnd.ms-powerpoint.slideshow.macroenabled.12",
                            _ => return None,
                        }.to_string());
                    }
                }
            }
            Event::Text(text)
                if depth == 0 && text.as_ref().iter().any(|b| !b.is_ascii_whitespace()) =>
            {
                return None;
            }
            Event::End(_) => {
                depth = depth.checked_sub(1)?;
                if depth == 0 {
                    closed = true;
                }
            }
            Event::DocType(_) => return None,
            Event::Eof => return mime.filter(|_| root && depth == 0),
            _ => {}
        }
    }
}

fn normalized_extension(file_name: &str) -> Option<String> {
    Path::new(file_name)
        .extension()
        .and_then(|ext| ext.to_str())
        .map(|ext| ext.trim().to_ascii_lowercase())
        .filter(|ext| !ext.is_empty())
}

fn read_prefix(path: &Path, len: usize) -> anyhow::Result<Vec<u8>> {
    let mut file = File::open(path).with_context(|| format!("cannot open {}", path.display()))?;
    let mut buf = vec![0u8; len];
    let size = file.read(&mut buf)?;
    buf.truncate(size);
    Ok(buf)
}

fn is_pe(bytes: &[u8]) -> bool {
    if bytes.len() < 0x40 || !bytes.starts_with(b"MZ") {
        return false;
    }
    let offset = u32::from_le_bytes([bytes[0x3c], bytes[0x3d], bytes[0x3e], bytes[0x3f]]) as usize;
    offset.checked_add(4).is_some_and(|end| end <= bytes.len())
        && &bytes[offset..offset + 4] == b"PE\0\0"
}

fn is_macho(bytes: &[u8]) -> bool {
    starts_with_any(
        bytes,
        &[
            b"\xFE\xED\xFA\xCE",
            b"\xFE\xED\xFA\xCF",
            b"\xCE\xFA\xED\xFE",
            b"\xCF\xFA\xED\xFE",
            b"\xCA\xFE\xBA\xBE",
            b"\xCA\xFE\xBA\xBF",
        ],
    )
}

fn is_lnk(bytes: &[u8]) -> bool {
    bytes.starts_with(b"\x4C\x00\x00\x00\x01\x14\x02\x00")
}

fn starts_with_any(bytes: &[u8], patterns: &[&[u8]]) -> bool {
    patterns.iter().any(|pattern| bytes.starts_with(pattern))
}

fn safe_zip_path(name: &str, max_depth: usize) -> bool {
    let path = Path::new(name);
    if path.is_absolute() || name.starts_with('/') {
        return false;
    }
    let mut depth = 0usize;
    for component in path.components() {
        match component {
            Component::Normal(_) => depth += 1,
            Component::CurDir => {}
            _ => return false,
        }
    }
    depth <= max_depth
}

fn suspicious_embedded_extension(name: &str) -> bool {
    Path::new(name)
        .extension()
        .and_then(|ext| ext.to_str())
        .map(|ext| {
            let ext = ext.to_ascii_lowercase();
            executable_extension(&ext) || script_extension(&ext) || blocked_archive_extension(&ext)
        })
        .unwrap_or(false)
}

fn mime_matches(actual: &str, expected: &str) -> bool {
    actual
        .split(';')
        .next()
        .unwrap_or("")
        .trim()
        .eq_ignore_ascii_case(expected)
}

#[derive(Debug, Clone, Copy)]
enum FormatKind {
    Pdf,
    Image,
    Text,
    LegacyOffice,
    Ooxml(OoxmlKind),
}

#[derive(Debug, Clone, Copy)]
enum OoxmlKind {
    Word,
    Excel,
    PowerPoint,
}

impl OoxmlKind {
    fn main_part(self) -> &'static str {
        match self {
            Self::Word => "word/document.xml",
            Self::Excel => "xl/workbook.xml",
            Self::PowerPoint => "ppt/presentation.xml",
        }
    }
}

fn format_kind(ext: &str) -> Option<FormatKind> {
    match ext {
        "pdf" => Some(FormatKind::Pdf),
        "jpg" | "jpeg" | "png" | "gif" | "bmp" | "tif" | "tiff" | "webp" => Some(FormatKind::Image),
        "txt" | "csv" | "md" => Some(FormatKind::Text),
        "doc" | "xls" | "ppt" => Some(FormatKind::LegacyOffice),
        "docx" | "docm" => Some(FormatKind::Ooxml(OoxmlKind::Word)),
        "xlsx" | "xlsm" => Some(FormatKind::Ooxml(OoxmlKind::Excel)),
        "pptx" | "pptm" | "ppsx" | "ppsm" => Some(FormatKind::Ooxml(OoxmlKind::PowerPoint)),
        _ => None,
    }
}

fn executable_extension(ext: &str) -> bool {
    matches!(
        ext,
        "exe"
            | "dll"
            | "sys"
            | "scr"
            | "com"
            | "msi"
            | "msp"
            | "drv"
            | "elf"
            | "so"
            | "dylib"
            | "app"
            | "dmg"
            | "jar"
            | "war"
            | "ear"
            | "apk"
            | "dex"
            | "wasm"
            | "class"
            | "lnk"
            | "chm"
    )
}

fn script_extension(ext: &str) -> bool {
    matches!(
        ext,
        "py" | "pyw"
            | "mjs"
            | "cjs"
            | "jsx"
            | "ts"
            | "tsx"
            | "bash"
            | "zsh"
            | "pl"
            | "rb"
            | "php"
            | "lua"
            | "r"
            | "bat"
            | "cmd"
            | "ps1"
            | "psm1"
            | "vbs"
            | "vbe"
            | "js"
            | "jse"
            | "hta"
            | "wsf"
            | "wsh"
            | "sh"
    )
}

fn blocked_archive_extension(ext: &str) -> bool {
    matches!(
        ext,
        "zip" | "rar" | "7z" | "tar" | "gz" | "tgz" | "bz2" | "xz" | "cab" | "iso"
    )
}

fn allowed_mimes() -> BTreeMap<&'static str, &'static [&'static str]> {
    BTreeMap::from([
        (
            "ppsx",
            &["application/vnd.openxmlformats-officedocument.presentationml.slideshow"][..],
        ),
        (
            "ppsm",
            &["application/vnd.ms-powerpoint.slideshow.macroenabled.12"][..],
        ),
        ("pdf", &["application/pdf"][..]),
        ("jpg", &["image/jpeg"][..]),
        ("jpeg", &["image/jpeg"][..]),
        ("png", &["image/png"][..]),
        ("gif", &["image/gif"][..]),
        ("bmp", &["image/bmp", "image/x-ms-bmp"][..]),
        ("tif", &["image/tiff"][..]),
        ("tiff", &["image/tiff"][..]),
        ("webp", &["image/webp"][..]),
        ("txt", &["text/plain", "application/octet-stream"][..]),
        (
            "csv",
            &[
                "text/csv",
                "text/plain",
                "application/vnd.ms-excel",
                "application/octet-stream",
            ][..],
        ),
        (
            "md",
            &["text/markdown", "text/plain", "application/octet-stream"][..],
        ),
        (
            "doc",
            &["application/msword", "application/octet-stream"][..],
        ),
        (
            "xls",
            &["application/vnd.ms-excel", "application/octet-stream"][..],
        ),
        (
            "ppt",
            &["application/vnd.ms-powerpoint", "application/octet-stream"][..],
        ),
        (
            "docx",
            &[
                "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
                "application/zip",
            ][..],
        ),
        (
            "docm",
            &[
                "application/vnd.ms-word.document.macroenabled.12",
                "application/zip",
            ][..],
        ),
        (
            "xlsx",
            &[
                "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                "application/zip",
            ][..],
        ),
        (
            "xlsm",
            &[
                "application/vnd.ms-excel.sheet.macroenabled.12",
                "application/zip",
            ][..],
        ),
        (
            "pptx",
            &[
                "application/vnd.openxmlformats-officedocument.presentationml.presentation",
                "application/zip",
            ][..],
        ),
        (
            "pptm",
            &[
                "application/vnd.ms-powerpoint.presentation.macroenabled.12",
                "application/zip",
            ][..],
        ),
    ])
}

pub fn rejected_message(report: &ValidationReport) -> anyhow::Error {
    anyhow!(
        "Walidacja pliku zakończona werdyktem {:?}: {:?}",
        report.verdict,
        report.reasons
    )
}
