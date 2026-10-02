tokio::time::timeout(Duration::from_secs(30), async {
            let req = self.client.get(&endpoint).query(&[
                ("file_id", item.file_id.as_str()),
                ("input_uri", item.input_uri.as_str()),
                ("output_bucket", item.output_bucket.as_str()),
                ("output_path_prefix", item.output_path_prefix.as_str()),
            ]);
            let status = send_authenticated_request(req, &self.cli).await?
                .json::<DoclingStatusResponse>().await?;
            anyhow::ensure!(status.file_id == item.file_id && status.input_uri == item.input_uri,
                "Docling status belongs to another file");
            Ok(status)
        }).await.map_err(|_| anyhow!("Docling status request timed out"))?
    }

    pub async fn resume_or_submit_batch(
        &self,
        items: &[DoclingBatchItem],
    ) -> Result<HashMap<String, DoclingStatusResponse>> {
        let mut statuses = HashMap::new();
        let mut submit = Vec::new();
        let mut pending = Vec::new();
        for item in items {
            if !self.cli.process_docling_from_beginning {
                match self.get_status(item).await {
                    Ok(s) if s.status.eq_ignore_ascii_case("success")
                        && s.error.is_none()
                        && s.output_uri.as_deref().map(|u| !u.trim().is_empty()).unwrap_or(false) => {
                        info!(file_id = %item.file_id, "Docling: reusing completed conversion");
                        statuses.insert(item.file_id.clone(), s);
                        continue;
                    }
                    Ok(s) if matches!(s.status.to_ascii_lowercase().as_str(), "queued" | "processing") => {
                        info!(file_id = %item.file_id, status = %s.status, "Docling: resuming polling");
                        pending.push(item.clone());
                        continue;
                    }
                    Ok(s) => warn!(file_id = %item.file_id, status = %s.status, "Docling: submitting again"),
                    Err(e) => warn!(file_id = %item.file_id, error = %e, "Docling status unavailable; submitting again"),
                }
            }
            submit.push(item.clone());
            pending.push(item.clone());
        }
        info!(reused = statuses.len(), submit = submit.len(), pending = pending.len(), "Docling resume plan");
        if !submit.is_empty() {
            self.submit_batch_async(&submit).await?;
        }
        if !pending.is_empty() {
            statuses.extend(self.poll_batch_status(&pending).await?);
        }
        Ok(statuses)
    }

    pub async fn poll_batch_status(
        &self,
        items: &[DoclingBatchItem],
    ) -> Result<HashMap<String, DoclingStatusResponse>> {
