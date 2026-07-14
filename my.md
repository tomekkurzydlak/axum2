use std::collections::BTreeMap;
use std::fs::{self, File};
use std::io::Read;
use std::path::{Component, Path};

use anyhow::{Context, anyhow};
use image::ImageReader;
use serde::Serialize;
use zip::ZipArchive;

#[derive(Debug, Clone, Serialize, PartialEq, Eq)]
pub enum FileVerdict {
    Clean,
    CleanWithActiveContent,
    Rejected,
    ScanFailed,
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
        }
    }

    pub fn is_rejected(&self) -> bool {
        matches!(
            self.verdict,
            FileVerdict::Rejected | FileVerdict::ScanFailed
        )
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
}

impl Default for ValidationConfig {
    fn default() -> Self {
        Self {
            allow_missing_extension: false,
            max_archive_entries: 2_000,
            max_archive_entry_size: 200 * 1024 * 1024,
            max_archive_total_size: 500 * 1024 * 1024,
            max_archive_path_depth: 16,
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

    validate_magic(path, &mut report)?;
    if report.is_rejected() {
        return Ok(report);
    }

    let Some(ext) = extension.as_deref() else {
        report.allowed();
        return Ok(report);
    };

    validate_mime(ext, detected_mime, &mut report);
    if report.is_rejected() {
        return Ok(report);
    }

    match format_kind(ext) {
        Some(FormatKind::Ooxml(kind)) => validate_ooxml(path, kind, config, &mut report),
        Some(FormatKind::Pdf) => validate_pdf(path, &mut report),
        Some(FormatKind::Image) => validate_image(path, &mut report),
        Some(FormatKind::LegacyOffice) => validate_legacy_office(path, &mut report),
        Some(FormatKind::Text) => {}
        None => report.reject(ValidationReason::UnsupportedFormat),
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

    if is_pe(&header)
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

fn validate_pdf(path: &Path, report: &mut ValidationReport) {
    if lopdf::Document::load(path).is_err() {
        report.reject(ValidationReason::InvalidFileStructure);
        return;
    }

    if let Ok(bytes) = fs::read(path) {
        let markers = [
            b"/JavaScript".as_slice(),
            b"/JS".as_slice(),
            b"/OpenAction".as_slice(),
            b"/AA".as_slice(),
            b"/Launch".as_slice(),
            b"/EmbeddedFile".as_slice(),
            b"/RichMedia".as_slice(),
            b"/XFA".as_slice(),
            b"/AcroForm".as_slice(),
        ];
        if markers.iter().any(|marker| contains_bytes(&bytes, marker)) {
            report.flags.active_content = true;
            report.active(ValidationReason::ActiveContentPresent, 20);
        }
    }
}

fn validate_image(path: &Path, report: &mut ValidationReport) {
    let image = ImageReader::open(path)
        .and_then(|reader| reader.with_guessed_format())
        .and_then(|reader| reader.decode().map_err(std::io::Error::other));

    if image.is_err() {
        report.reject(ValidationReason::InvalidFileStructure);
    }
}

fn validate_legacy_office(path: &Path, report: &mut ValidationReport) {
    let Ok(header) = read_prefix(path, 8) else {
        report.scan_failed(ValidationReason::ParserFailure);
        return;
    };
    if !header.starts_with(b"\xD0\xCF\x11\xE0\xA1\xB1\x1A\xE1") {
        report.reject(ValidationReason::InvalidFileStructure);
        return;
    }

    if let Ok(bytes) = fs::read(path) {
        if contains_ascii_ci(&bytes, b"vba") || contains_ascii_ci(&bytes, b"_vba_project") {
            report.flags.macro_enabled = true;
            report.active(ValidationReason::MacroPresent, 20);
        }
    }
}

fn validate_ooxml(
    path: &Path,
    kind: OoxmlKind,
    config: &ValidationConfig,
    report: &mut ValidationReport,
) {
    let file = match File::open(path) {
        Ok(file) => file,
        Err(_) => {
            report.scan_failed(ValidationReason::ParserFailure);
            return;
        }
    };

    let mut archive = match ZipArchive::new(file) {
        Ok(archive) => archive,
        Err(_) => {
            report.reject(ValidationReason::InvalidFileStructure);
            return;
        }
    };

    if archive.len() > config.max_archive_entries {
        report.reject(ValidationReason::ArchiveLimitExceeded);
        return;
    }

    let mut total_size = 0u64;
    let mut has_content_types = false;
    let mut has_main_part = false;

    for idx in 0..archive.len() {
        let mut entry = match archive.by_index(idx) {
            Ok(entry) => entry,
            Err(_) => {
                report.scan_failed(ValidationReason::ParserFailure);
                return;
            }
        };

        if entry.encrypted() {
            report.flags.password_protected = true;
            report.reject(ValidationReason::PasswordProtectedContent);
            return;
        }

        let name = entry.name().replace('\\', "/");
        if !safe_zip_path(&name, config.max_archive_path_depth) {
            report.reject(ValidationReason::InvalidFileStructure);
            return;
        }

        total_size = total_size.saturating_add(entry.size());
        if entry.size() > config.max_archive_entry_size
            || total_size > config.max_archive_total_size
        {
            report.reject(ValidationReason::ArchiveLimitExceeded);
            return;
        }

        if name == "[Content_Types].xml" {
            has_content_types = true;
        }
        if name == kind.main_part() {
            has_main_part = true;
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
            return;
        }

        let mut sample = Vec::new();
        if entry.size() <= 1024 * 1024 {
            let _ = entry.by_ref().take(4096).read_to_end(&mut sample);
            if is_pe(&sample) || sample.starts_with(b"\x7FELF") || is_macho(&sample) {
                report.flags.embedded_executable = true;
                report.reject(ValidationReason::EmbeddedExecutable);
                return;
            }
        }
    }

    if !has_content_types || !has_main_part {
        report.reject(ValidationReason::InvalidFileStructure);
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

fn contains_bytes(haystack: &[u8], needle: &[u8]) -> bool {
    haystack
        .windows(needle.len())
        .any(|window| window == needle)
}

fn contains_ascii_ci(haystack: &[u8], needle: &[u8]) -> bool {
    if needle.is_empty() {
        return true;
    }
    haystack.windows(needle.len()).any(|window| {
        window
            .iter()
            .zip(needle)
            .all(|(a, b)| a.eq_ignore_ascii_case(b))
    })
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
    actual.eq_ignore_ascii_case(expected)
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
        "pptx" | "pptm" => Some(FormatKind::Ooxml(OoxmlKind::PowerPoint)),
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
        "bat"
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
