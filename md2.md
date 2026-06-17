def _mark_footnote_reference_marker(markdown: str, footnote_id: str, search_start: int = 0, search_end: int | None = None) -> str:
        search_end = len(markdown) if search_end is None else search_end
        search_area = markdown[search_start:search_end]
        marker = f"[{footnote_id}]"
        marker_pattern = re.compile(
            rf"(?<=[a-ząćęłńóśźż])\s+{re.escape(footnote_id)}(?=(?:\s*[,;:])?\s+[a-ząćęłńóśźż])",
            re.IGNORECASE,
        )
        for match in marker_pattern.finditer(search_area):
            prefix = search_area[max(0, match.start() - 20) : match.start()]
            if re.search(r"(?:Figure|Fig\.|Table|Section)\s*$", prefix, re.IGNORECASE):
                continue
            marker_start = search_start + match.start()
            return f"{markdown[:marker_start]} {marker}{markdown[search_start + match.end():]}"
        return markdown
                footnotes = self._extract_docling_footnotes(result)
                if input_path.lower().endswith(".docx"):
                    markdown, docx_footnotes = self._add_docx_footnote_references(input_path, markdown)
                    footnotes.extend(docx_footnotes)
                markdown = self._mark_footnote_references_as_superscript(markdown)
                markdown = self._normalize_footnote_blocks(markdown, footnotes)
                markdown = self._relocate_footnotes_to_reference_pages(markdown, footnotes)
                return DoclingConversionResult(markdown=markdown, raw_result=result)
            if hasattr(result, "export_to_markdown"):
                footnotes = self._extract_docling_footnotes(result)
                markdown = self._mark_footnote_references_as_superscript(result.export_to_markdown())
                markdown = self._normalize_footnote_blocks(markdown, footnotes)
                markdown = self._relocate_footnotes_to_reference_pages(markdown, footnotes)
                return DoclingConversionResult(markdown=markdown, raw_result=result)

            raise DoclingError("Docling conversion returned unsupported result format")
        except DoclingError:
            raise
        except Exception as exc:  # noqa: BLE001
            raise DoclingError(f"Docling conversion failed: {exc}") from exc

    def _create_document_converter(self):
        os.environ.setdefault("DOCLING_ARTIFACTS_PATH", self.artifacts_path)

        if InputFormat is None or PdfFormatOption is None or PdfPipelineOptions is None:
            return DocumentConverter()

        pipeline_options = PdfPipelineOptions()
        pipeline_options.do_ocr = self.ocr_enabled
        if self.ocr_enabled:
            pipeline_options.ocr_options = self._create_ocr_options()
        pipeline_options.accelerator_options.device = self._resolve_docling_device()

        return DocumentConverter(
            format_options={
                InputFormat.PDF: PdfFormatOption(pipeline_options=pipeline_options),
            }
        )

    def _resolve_docling_device(self) -> str:
        return "cuda" if self.device.strip().lower() == "gpu" else "cpu"

    def _create_ocr_options(self):
        ocr_engine = self.ocr_engine.strip().lower()
        ocr_langs = self._parse_ocr_langs()
        if ocr_engine == "rapidocr" and RapidOcrOptions is not None:
            return RapidOcrOptions(lang=ocr_langs, backend=self.rapidocr_backend.strip().lower())
        if ocr_engine == "tesseract" and TesseractCliOcrOptions is not None:
            return TesseractCliOcrOptions(lang=ocr_langs)
        if ocr_engine == "easyocr" and EasyOcrOptions is not None:
            return EasyOcrOptions(lang=self._to_easyocr_langs(ocr_langs), use_gpu=self._resolve_docling_device() == "cuda")
        raise DoclingError(f"Unsupported Docling OCR engine: {self.ocr_engine}")

    def _parse_ocr_langs(self) -> list[str]:
        langs = [lang.strip().lower() for lang in self.ocr_langs.split(",") if lang.strip()]
        return langs or ["pol", "eng"]

    @staticmethod
    def _to_easyocr_langs(ocr_langs: list[str]) -> list[str]:
        mapping = {"pol": "pl", "eng": "en", "deu": "de", "fra": "fr", "spa": "es"}
        return [mapping.get(lang, lang) for lang in ocr_langs]

    # Backward compatibility alias for older call sites.
    def convert_pdf_to_markdown(self, pdf_path: str) -> DoclingConversionResult:
        return self.convert_input_to_markdown(pdf_path)

    @staticmethod
    def _export_markdown_with_native_page_placeholders(result: object) -> str:
        document = getattr(result, "document", None)
        if document is None:
            raise DoclingError("Docling conversion result has no document")
        return document.export_to_markdown(page_break_placeholder="<!-- page -->")

    @staticmethod
    def _mark_footnote_references_as_superscript(markdown: str) -> str:
        markdown = re.sub(r"\[\^([A-Za-z0-9_-]+)\](?!:)", r"[\1]", markdown)
        return re.sub(r"<sup>([A-Za-z0-9_-]+)</sup>", r"[\1]", markdown)

    @staticmethod
    def _extract_docling_footnotes(result: object) -> list[tuple[str, str, list[str]]]:
        document = getattr(result, "document", None)
        if document is None or DocItemLabel is None or not hasattr(document, "iterate_items"):
            return []

        footnotes: list[tuple[str, str, list[str]]] = []
        for item, _level in document.iterate_items():
            if getattr(item, "label", None) != DocItemLabel.FOOTNOTE:
                continue

            text = (getattr(item, "text", "") or "").strip()
            match = re.match(r"^(\d{1,3})[\s.)\]:-]+(.+)$", text)
            if match is None:
                continue

            footnote_id = match.group(1)
            footnote_text = match.group(2).strip()
            hyperlink = getattr(item, "hyperlink", None)
            if hyperlink is not None and footnote_text == str(hyperlink):
                block = f"[{footnote_id}] [{footnote_text}]({hyperlink})"
            else:
                block = f"[{footnote_id}] {footnote_text}"

            candidates = [text, block]
            if hyperlink is not None:
                candidates.append(f"[{text}]({hyperlink})")
            footnotes.append((footnote_id, block, candidates))

        return footnotes

    @staticmethod
    def _normalize_footnote_blocks(markdown: str, footnotes: list[tuple[str, str, list[str]]]) -> str:
        result = markdown
        for _footnote_id, block, candidates in footnotes:
            for candidate in candidates:
                if not candidate or candidate == block:
                    continue
                pattern = re.compile(rf"(?m)^\s*{re.escape(candidate)}\s*$")
                result = pattern.sub(block, result, count=1)
        return result

    def _relocate_footnotes_to_reference_pages(self, markdown: str, footnotes: list[tuple[str, str, list[str]]] | None = None) -> str:
        if not self.relocate_footnotes:
            return markdown

        if footnotes:
            result = markdown
            for footnote_id, block, candidates in footnotes:
                candidate_position = self._find_footnote_block_position(result, candidates)
                search_start = 0
                search_end: int | None = None
                if candidate_position != -1:
                    page_start = result.rfind("<!-- page -->", 0, candidate_position)
                    search_start = 0 if page_start == -1 else page_start + len("<!-- page -->")
                    search_end = candidate_position
                result = self._remove_footnote_block_candidates(result, candidates)
                result = self._insert_footnote_after_reference_sentence(result, footnote_id, block, search_start, search_end)
            return result

        extracted_footnotes = self._extract_footnote_blocks(markdown)
        if not extracted_footnotes:
            return markdown

        footnotes_by_id: dict[str, list[tuple[str, int, int]]] = {}
        for footnote_id, block, start_line, end_line in extracted_footnotes:
            footnotes_by_id.setdefault(footnote_id, []).append((block, start_line, end_line))

        moved_ranges: list[tuple[int, int]] = []
        result = markdown

        for footnote_id, blocks in footnotes_by_id.items():
            # Move only unambiguous footnotes; duplicate definitions are left in their original place.
            if len(blocks) != 1:
                continue

            block, start_line, end_line = blocks[0]
            moved_ranges.append((start_line, end_line))
            result = self._insert_footnote_after_reference_sentence(result, footnote_id, block)

        if not moved_ranges:
            return markdown

        return self._remove_footnote_blocks(result, moved_ranges)

    @staticmethod
    def _remove_footnote_block_candidates(markdown: str, candidates: list[str]) -> str:
        result = markdown
        for candidate in candidates:
            if not candidate:
                continue
            pattern = re.compile(rf"(?m)^\s*{re.escape(candidate)}\s*$\n?")
            result = pattern.sub("", result, count=1)
        return result

    @staticmethod
    def _find_footnote_block_position(markdown: str, candidates: list[str]) -> int:
        positions = [position for candidate in candidates if candidate and (position := markdown.find(candidate)) != -1]
        return min(positions) if positions else -1

    @staticmethod
    def _insert_footnote_after_reference_sentence(
        markdown: str,
        footnote_id: str,
        block: str,
        search_start: int = 0,
        search_end: int | None = None,
    ) -> str:
        if block in markdown:
            return markdown

        search_end = len(markdown) if search_end is None else search_end
        search_area = markdown[search_start:search_end]
        marker = f"[{footnote_id}]"
        marker_pattern = re.compile(
            rf"(?<=[a-ząćęłńóśźż])\s+{re.escape(footnote_id)}(?=(?:\s*[,;:])?\s+[a-ząćęłńóśźż])",
            re.IGNORECASE,
        )
        marker_match = None
        for match in marker_pattern.finditer(search_area):
            prefix = search_area[max(0, match.start() - 20) : match.start()]
            if re.search(r"(?:Figure|Fig\.|Table|Section)\s*$", prefix, re.IGNORECASE):
                continue
            marker_match = match
            break

        if marker_match is not None:
            marker_start = search_start + marker_match.start()
            markdown = f"{markdown[:marker_start]} {marker}{markdown[search_start + marker_match.end():]}"
            marker_end = marker_start + len(marker) + 1
        else:
            bracket_match = re.search(rf"\[{re.escape(footnote_id)}\]", search_area)
            if bracket_match is None:
                return f"{markdown.rstrip()}\n\n{block}"
            prefix = search_area[max(0, bracket_match.start() - 20) : bracket_match.start()]
            if re.search(r"(?:Figure|Fig\.|Table|Section)\s*$", prefix, re.IGNORECASE):
                return f"{markdown.rstrip()}\n\n{block}"
            marker_end = search_start + bracket_match.end()

        sentence_end = re.search(r"[.!?](?:\s|$)", markdown[marker_end:])
        if sentence_end is None:
            insert_at = markdown.find("\n", marker_end)
            if insert_at == -1:
                insert_at = len(markdown)
        else:
            insert_at = marker_end + sentence_end.end()

        suffix = "" if insert_at >= len(markdown) or markdown[insert_at].isspace() else " "
        return f"{markdown[:insert_at].rstrip()} {block}{suffix}{markdown[insert_at:]}"

    @staticmethod
    def _add_docx_footnote_references(input_path: str, markdown: str) -> tuple[str, list[tuple[str, str, list[str]]]]:
        word_namespace = "{http://schemas.openxmlformats.org/wordprocessingml/2006/main}"
        paragraph_tag = f"{word_namespace}p"
        run_tag = f"{word_namespace}r"
        text_tag = f"{word_namespace}t"
        footnote_reference_tag = f"{word_namespace}footnoteReference"
        footnote_tag = f"{word_namespace}footnote"
        id_attr = f"{word_namespace}id"
        type_attr = f"{word_namespace}type"

        try:
            with zipfile.ZipFile(input_path) as docx_zip:
                document_xml = docx_zip.read("word/document.xml")
                footnotes_xml = docx_zip.read("word/footnotes.xml")
        except Exception:  # noqa: BLE001
            return markdown, []

        try:
            document_root = ElementTree.fromstring(document_xml)
            footnotes_root = ElementTree.fromstring(footnotes_xml)
        except ElementTree.ParseError:
            return markdown, []

        footnote_texts: dict[str, str] = {}
        for footnote in footnotes_root.findall(footnote_tag):
            footnote_id = footnote.attrib.get(id_attr)
            if not footnote_id or footnote.attrib.get(type_attr):
                continue
            text = "".join(text_node.text or "" for text_node in footnote.iter(text_tag)).strip()
            if text:
                footnote_texts[footnote_id] = text

        result = markdown
        footnotes: list[tuple[str, str, list[str]]] = []
        for paragraph in document_root.iter(paragraph_tag):
            before_parts: list[str] = []
            after_parts: list[str] = []
            paragraph_parts: list[str] = []
            current_footnote_id: str | None = None

            for run in paragraph.findall(run_tag):
                reference = run.find(footnote_reference_tag)
                if reference is not None:
                    current_footnote_id = reference.attrib.get(id_attr)
                    continue

                run_text = "".join(text_node.text or "" for text_node in run.iter(text_tag))
                paragraph_parts.append(run_text)
                if current_footnote_id is None:
                    before_parts.append(run_text)
                else:
                    after_parts.append(run_text)

            if current_footnote_id is None or current_footnote_id not in footnote_texts:
                continue

            paragraph_text = "".join(paragraph_parts).strip()
            before_text = "".join(before_parts).strip()
            if not paragraph_text or not before_text:
                continue

            marker = f"[{current_footnote_id}]"
            position = result.find(paragraph_text)
            if position == -1:
                continue

            reference_at = result.find(before_text, position)
            if reference_at == -1:
                continue

            insert_at = reference_at + len(before_text)
            if result[insert_at : insert_at + len(marker)] != marker:
                result = f"{result[:insert_at]}{marker}{result[insert_at:]}"

            block = f"{marker} {footnote_texts[current_footnote_id]}"
            footnotes.append((current_footnote_id, block, [block]))

        return result, footnotes

    @staticmethod
    def _extract_footnote_blocks(markdown: str) -> list[tuple[str, str, int, int]]:
        lines = markdown.splitlines()
        footnotes: list[tuple[str, str, int, int]] = []
        index = 0
        definition_pattern = re.compile(r"^\[\^([A-Za-z0-9_-]+)\]:\s*(.*)$|^(\d{1,3}):\s+(.+)$")

        while index < len(lines):
            match = definition_pattern.match(lines[index])
            if match is None:
                index += 1
                continue

            start_line = index
            footnote_id = match.group(1) or match.group(3)
            block_lines = [lines[index]]
            index += 1

            if match.group(1):
                while index < len(lines) and (lines[index].startswith((" ", "\t")) or not lines[index].strip()):
                    block_lines.append(lines[index])
                    index += 1

            footnotes.append((footnote_id, "\n".join(block_lines).rstrip(), start_line, index))

        return footnotes

    @staticmethod
    def _remove_footnote_blocks(markdown: str, ranges: list[tuple[int, int]]) -> str:
        lines = markdown.splitlines()
        removed_lines: set[int] = set()
        for start_line, end_line in ranges:
            removed_lines.update(range(start_line, end_line))
        return "\n".join(line for index, line in enumerate(lines) if index not in removed_lines)

    @staticmethod
    def _export_markdown_with_numbered_page_markers(result: object) -> str:
        document = getattr(result, "document", None)
        pages = getattr(result, "pages", None)
        if document is None:
            raise DoclingError("Docling conversion result has no document")

        # Future fallback: this function keeps explicit page numbering (<!-- page: N -->)
        # by exporting markdown page-by-page. It is currently not used in runtime flow.
        if isinstance(pages, list) and pages:
            chunks: list[str] = []
            for page in pages:
                page_no = getattr(page, "page_no", None)
                if not isinstance(page_no, int):
                    continue
                page_md = document.export_to_markdown(page_no=page_no).strip()
                chunks.append(f"<!-- page: {page_no} -->\n{page_md}" if page_md else f"<!-- page: {page_no} -->")
            if chunks:
                return "\n\n".join(chunks)

        return document.export_to_markdown(page_break_placeholder="<!-- page -->")
