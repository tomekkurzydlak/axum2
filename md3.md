async def _gcs_worker_loop(self, worker_id: int) -> None:
        while True:
            try:
                claimed = await self._claim_next_gcs_job()
                if claimed is None:
                    await asyncio.sleep(self._poll_seconds)
                    continue
                blob_path, _generation, payload = claimed
                heartbeat = asyncio.create_task(self._heartbeat_job(blob_path))
                try:
                    request = ConvertRequest(**payload["request"])
                    await self._write_status(request, "processing")
                    await self._conversion_service.convert(request, persist_status_json=True)
                    await self._delete_owned_gcs_job(blob_path)
                except Exception as exc:  # noqa: BLE001
                    logger.exception("Persistent worker %s failed to process conversion", worker_id)
                    try:
                        request = ConvertRequest(**payload["request"])
                        await self._write_status(request, "error")
                    finally:
                        await self._delete_owned_gcs_job(blob_path)
                finally:
                    heartbeat.cancel()
                    await asyncio.gather(heartbeat, return_exceptions=True)
            except asyncio.CancelledError:
                raise
            except Exception:  # noqa: BLE001
                logger.exception("Persistent worker %s failed to poll conversion queue", worker_id)
                await asyncio.sleep(self._poll_seconds)

    async def _claim_next_gcs_job(self) -> tuple[str, int | None, dict] | None:
        claimed = await self._claim_gcs_job("priority")
        if claimed is not None:
            return claimed
        if await self._has_gcs_jobs("priority"):
            return None
        return await self._claim_gcs_job("standard")
