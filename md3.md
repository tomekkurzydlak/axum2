def list_json_blobs(self, *, bucket_name: str, prefix: str) -> list[GcsJsonBlob]:
        try:
            bucket = self.client.bucket(bucket_name)
            return [
                GcsJsonBlob(blob_path=blob.name, generation=int(blob.generation) if blob.generation else None)
                for blob in bucket.list_blobs(prefix=prefix)
                if blob.name.endswith(".json")
            ]
        except Exception as exc:  # noqa: BLE001
            raise GcsError(f"Failed to list json blobs from GCS: {exc}") from exc

    def download_json_with_generation(self, *, input_bucket: str, input_blob_path: str) -> tuple[dict, int | None]:
        try:
            bucket = self.client.bucket(input_bucket)
            blob = bucket.blob(input_blob_path)
            if not blob.exists():
                raise ServiceError(f"Status file not found: gs://{input_bucket}/{input_blob_path}", status_code=404)
            content = blob.download_as_text()
            generation = int(blob.generation) if blob.generation else None
            return json.loads(content), generation
        except ServiceError:
            raise
        except NotFound as exc:
            raise ServiceError(f"Status file not found: gs://{input_bucket}/{input_blob_path}", status_code=404) from exc
        except Exception as exc:  # noqa: BLE001
            raise GcsError(f"Failed to download json from GCS: {exc}") from exc

            ==

async def submit_persistent(self, request: ConvertRequest) -> str:
        if not self.persistent_enabled:
            raise RuntimeError("Persistent conversion queue is not configured")
        job_id = uuid4().hex
        created_at = self._utc_now()
        group = "priority" if request.tenant_id in self._priority_tenant_ids else "standard"
        blob_path = self._job_blob_path(group=group, created_at=created_at, job_id=job_id)
        payload = {
            "job_id": job_id,
            "status": "queued",
            "created_at": created_at,
            "started_at": None,
            "lease_until": None,
            "lease_owner": None,
            "attempt": 0,
            "request": request.model_dump(mode="json"),
        }
        await asyncio.to_thread(
            self._gcs_service.upload_json,
            payload=payload,
            output_bucket=self._queue_bucket,
            output_blob_path=blob_path,
            if_generation_match=0,
        )
        return job_id

        ==
        return await self._claim_gcs_job("standard")

    async def _has_gcs_jobs(self, group: str) -> bool:
        prefix = f"{self._queue_prefix}/{group}/"
        blobs = await asyncio.to_thread(
            self._gcs_service.list_json_blobs,
            bucket_name=self._queue_bucket,
            prefix=prefix,
        )
        return bool(blobs)

    async def _claim_gcs_job(self, group: str) -> tuple[str, int | None, dict] | None:
        prefix = f"{self._queue_prefix}/{group}/"
        blobs = await asyncio.to_thread(
            self._gcs_service.list_json_blobs,
            bucket_name=self._queue_bucket,
            prefix=prefix,
        )
        for blob in sorted(blobs, key=lambda item: item.blob_path):
            try:
                payload, generation = await asyncio.to_thread(
                    self._gcs_service.download_json_with_generation,
                    input_bucket=self._queue_bucket,
                    input_blob_path=blob.blob_path,
                )
                if not self._can_claim(payload):
                    continue
                now = self._utc_now()
                payload["status"] = "processing"
                payload["started_at"] = payload.get("started_at") or now
                payload["lease_until"] = self._lease_until()
                payload["lease_owner"] = self._lease_owner
                payload["attempt"] = int(payload.get("attempt") or 0) + 1
                await asyncio.to_thread(
                    self._gcs_service.upload_json,
                    payload=payload,
                    output_bucket=self._queue_bucket,
                    output_blob_path=blob.blob_path,
                    if_generation_match=generation,
                )
                return blob.blob_path, generation, payload
            except Exception:  # noqa: BLE001
                logger.debug("Skipped queued conversion job %s during claim", blob.blob_path, exc_info=True)
        return None

    async def _heartbeat_job(self, blob_path: str) -> None:
        while True:
            await asyncio.sleep(self._heartbeat_seconds)
            try:
                payload, generation = await asyncio.to_thread(
                    self._gcs_service.download_json_with_generation,
                    input_bucket=self._queue_bucket,
                    input_blob_path=blob_path,
                )
                if payload.get("lease_owner") != self._lease_owner:
                    return
                payload["lease_until"] = self._lease_until()
                await asyncio.to_thread(
                    self._gcs_service.upload_json,
                    payload=payload,
                    output_bucket=self._queue_bucket,
                    output_blob_path=blob_path,
                    if_generation_match=generation,
                )
            except Exception:  # noqa: BLE001
                logger.exception("Failed to renew lease for queued conversion job %s", blob_path)

    async def _delete_owned_gcs_job(self, blob_path: str) -> None:
        try:
            payload, generation = await asyncio.to_thread(
                self._gcs_service.download_json_with_generation,
                input_bucket=self._queue_bucket,
                input_blob_path=blob_path,
            )
            if payload.get("lease_owner") != self._lease_owner:
                return
            await asyncio.to_thread(
                self._gcs_service.delete_blob,
                bucket_name=self._queue_bucket,
                blob_path=blob_path,
                if_generation_match=generation,
            )
        except Exception:  # noqa: BLE001
            logger.debug("Failed to delete queued conversion job %s", blob_path, exc_info=True)

    def _can_claim(self, payload: dict) -> bool:
        if payload.get("status") == "queued":
            return True
        if payload.get("status") != "processing":
            return False
        lease_until = payload.get("lease_until")
        if not lease_until:
            return True
        try:
            return datetime.fromisoformat(lease_until) < datetime.now(timezone.utc)
        except ValueError:
            return True

    async def _write_status(self, request: ConvertRequest, status: str) -> None:
        output_bucket = self._gcs_service.resolve_output_bucket(
            request_output_bucket=request.output_bucket,
            default_output_bucket=self._conversion_service.output_bucket_default,
        )
        output_prefix = request.output_path_prefix or self._conversion_service.output_path_prefix
        output_blob_path = self._gcs_service.build_output_blob_path(
            file_id=request.file_id,
            input_uri=request.input_uri,
            output_path_prefix=output_prefix,
        )
        status_blob_path = self._gcs_service.build_output_status_blob_path(output_blob_path=output_blob_path)
        payload = ConvertSavedResponse(
            file_id=request.file_id,
            tenant_id=request.tenant_id,
            input_uri=request.input_uri,
            status=status,
            response_mode=request.response_mode,
            processing_time_ms=0,
            output_uri=f"gs://{output_bucket}/{output_blob_path}",
            conversion=ConversionMetadataResponse(
                provider="docling",
                source_page_count=0,
                markdown_page_markers=None,
                extraction_confidence=None,
            ),
        ).model_dump(mode="json")
        await asyncio.to_thread(
            self._gcs_service.upload_json,
            payload=payload,
            output_bucket=output_bucket,
            output_blob_path=status_blob_path,
        )

    def _job_blob_path(self, *, group: str, created_at: str, job_id: str) -> str:
        safe_created_at = created_at.replace("-", "").replace(":", "").replace(".", "")
        return f"{self._queue_prefix}/{group}/{safe_created_at}_{job_id}.json"

    def _lease_until(self) -> str:
        return (datetime.now(timezone.utc) + timedelta(seconds=self._lease_seconds)).isoformat()

    @staticmethod
    def _utc_now() -> str:
        return datetime.now(timezone.utc).isoformat()
