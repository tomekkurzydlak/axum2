 def _relocate_footnotes_to_reference_pages(self, markdown: str) -> str:
        if not self.relocate_footnotes or "<!-- page -->" not in markdown:
            return markdown

        footnotes = self._extract_footnote_blocks(markdown)
        if not footnotes:
            return markdown

        footnotes_by_id: dict[str, list[tuple[str, int, int]]] = {}
        for footnote_id, block, start_line, end_line in footnotes:
            footnotes_by_id.setdefault(footnote_id, []).append((block, start_line, end_line))

        pages = markdown.split("<!-- page -->")
        page_footnotes: dict[int, list[str]] = {}
        moved_ranges: list[tuple[int, int]] = []

        for footnote_id, blocks in footnotes_by_id.items():
            # Move only unambiguous footnotes; duplicate definitions are left in their original place.
            if len(blocks) != 1:
                continue

            referenced_pages = [
                page_index
                for page_index, page in enumerate(pages)
                if re.search(rf"<sup>{re.escape(footnote_id)}</sup>", page)
            ]
            if len(referenced_pages) != 1:
                continue

            block, start_line, end_line = blocks[0]
            page_footnotes.setdefault(referenced_pages[0], []).append(block)
            moved_ranges.append((start_line, end_line))

        if not moved_ranges:
            return markdown

        markdown_without_moved_footnotes = self._remove_footnote_blocks(markdown, moved_ranges)
        pages = markdown_without_moved_footnotes.split("<!-- page -->")
        for page_index, blocks in page_footnotes.items():
            pages[page_index] = f"{pages[page_index].rstrip()}\n\n" + "\n".join(blocks)

        return "\n\n<!-- page -->\n".join(pages)

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
