def _convert_with_pdf_fallback(self, input_path: str):
        try:
            return self.converter.convert(input_path)
        except ConversionError:
            if not input_path.lower().endswith(".pdf"):
                raise
            logger.warning("PDF conversion failed for %s; retrying after Ghostscript normalization", input_path, exc_info=True)

            stage = "normalization"
            try:
                with tempfile.TemporaryDirectory(prefix="docling_pdf_") as tmp:
                    normalized_path = os.path.join(tmp, "normalized.pdf")
                    completed = subprocess.run(
                        [
                            "gs", "-dSAFER", "-dBATCH", "-dNOPAUSE",
                            "-sDEVICE=pdfwrite", "-dCompatibilityLevel=1.7",
                            f"-sOutputFile={normalized_path}", input_path,
                        ],
                        capture_output=True,
                        text=True,
                        timeout=120,
                    )
                    if completed.returncode != 0:
                        raise DoclingError(
                            f"Ghostscript failed with code {completed.returncode}: "
                            f"{(completed.stderr or completed.stdout)[-4000:]}"
                        )
                    if not os.path.isfile(normalized_path) or os.path.getsize(normalized_path) == 0:
                        raise DoclingError("Ghostscript produced no output PDF")
                    stage = "conversion"
                    result = self.converter.convert(normalized_path)
                    logger.info("PDF conversion succeeded after Ghostscript normalization for %s", input_path)
                    return result
            except Exception as exc:  # noqa: BLE001
                raise DoclingError(f"PDF fallback {stage} failed for {input_path}: {exc}") from exc
