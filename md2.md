def _find_docling_footnote_source_text(items: list[tuple[object, int]], footnote_item: object, footnote_id: str) -> str | None:
        footnote_prov = getattr(footnote_item, "prov", None) or []
        if not footnote_prov:
            return None

        footnote_page = footnote_prov[0].page_no
        footnote_top = footnote_prov[0].bbox.t
        source_pattern = re.compile(rf"[A-Za-zĄĆĘŁŃÓŚŹŻąćęłńóśźż]{re.escape(footnote_id)}(?:\s|$|[.,;:])")

        candidates: list[tuple[float, str]] = []
        for item, _level in items:
            if item is footnote_item or getattr(item, "label", None) == DocItemLabel.FOOTNOTE:
                continue

            prov = getattr(item, "prov", None) or []
            text = (getattr(item, "text", "") or "").strip()
            if not prov or not text or not source_pattern.search(text):
                continue
            if prov[0].page_no != footnote_page or prov[0].bbox.b <= footnote_top:
                continue

            candidates.append((prov[0].bbox.b, text))

        if not candidates:
            return None

        candidates.sort(key=lambda candidate: candidate[0])
        return candidates[0][1]
