Example
n 164

 @staticmethod
    def build_output_blob_path(*, file_id: str, input_uri: str, output_path_prefix: str) -> str:
        parsed = parse_gs_uri(input_uri)
        input_name = PurePosixPath(parsed.blob_path).name
        name_parts = input_name.split(".")
        if len(name_parts) >= 2:
            if len(name_parts) == 2:
                output_name = f"{name_parts[0]}.md"
            else:
                output_name = f"{name_parts[0]}.md.{name_parts[-1]}"
        else:
            output_name = f"{input_name}.md"
        prefix = output_path_prefix.strip("/")
        if prefix:
            return f"{prefix}/{output_name}"
        return output_name



n 112

    def download_input_to_tempfile(self, input_uri: str, mime_type: str) -> str:
        parsed = parse_gs_uri(input_uri)
        normalized_mime_type = mime_type.strip().lower()
        suffix = SUPPORTED_INPUT_MIME_TO_SUFFIX.get(normalized_mime_type)
        if suffix is None:
            supported = ", ".join(sorted(SUPPORTED_INPUT_MIME_TO_SUFFIX.keys()))
            raise BadRequestError(f"Unsupported mime_type for Docling input. Supported: {supported}")

        input_name = PurePosixPath(parsed.blob_path).name
        safe_prefix = re.sub(r"[^A-Za-z0-9._-]+", "_", input_name)[:64] or "input"
        tmp = tempfile.NamedTemporaryFile(delete=False, prefix=f"{safe_prefix}_", suffix=suffix)
        tmp_path = tmp.name
        tmp.close()
