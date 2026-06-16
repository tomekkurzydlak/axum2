@staticmethod
    def _add_docx_page_break_markers(input_path: str, markdown: str) -> str:
        if "<!-- page -->" in markdown or not input_path.lower().endswith(".docx"):
            return markdown

        anchors = DoclingService._extract_docx_page_break_anchors(input_path)
        if not anchors:
            return markdown

        result = markdown
        offset = 0
        marker = "\n\n<!-- page -->\n\n"
        for anchor in anchors:
            position = result.find(anchor, offset)
            if position == -1:
                continue
            insert_at = position + len(anchor)
            result = f"{result[:insert_at]}{marker}{result[insert_at:]}"
            offset = insert_at + len(marker)

        return result

    @staticmethod
    def _extract_docx_page_break_anchors(input_path: str) -> list[str]:
        word_namespace = "{http://schemas.openxmlformats.org/wordprocessingml/2006/main}"
        text_tag = f"{word_namespace}t"
        break_tag = f"{word_namespace}br"
        break_type_attr = f"{word_namespace}type"
        rendered_page_break_tag = f"{word_namespace}lastRenderedPageBreak"

        try:
            with zipfile.ZipFile(input_path) as docx_zip:
                document_xml = docx_zip.read("word/document.xml")
        except Exception:  # noqa: BLE001
            return []

        try:
            root = ElementTree.fromstring(document_xml)
        except ElementTree.ParseError:
            return []

        anchors: list[str] = []
        last_paragraph_text = ""
        paragraph_tag = f"{word_namespace}p"

        for paragraph in root.iter(paragraph_tag):
            paragraph_text_parts: list[str] = []
            text_before_break_parts: list[str] = []
            break_seen = False

            for element in paragraph.iter():
                if element.tag == text_tag and element.text:
                    paragraph_text_parts.append(element.text)
                    if not break_seen:
                        text_before_break_parts.append(element.text)
                elif element.tag == rendered_page_break_tag or (
                    element.tag == break_tag and element.attrib.get(break_type_attr) == "page"
                ):
                    anchor = "".join(text_before_break_parts).strip() or last_paragraph_text
                    if anchor:
                        anchors.append(anchor)
                    break_seen = True

            paragraph_text = "".join(paragraph_text_parts).strip()
            if paragraph_text:
                last_paragraph_text = paragraph_text

        return anchors
