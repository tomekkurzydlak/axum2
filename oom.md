pub struct AnalyzeFileStep;
#[async_trait]
impl Step for AnalyzeFileStep {
    fn name(&self) -> &'static str {
        "AnalyzeFileStep"
    }
    async fn execute(&self, ctx: &mut PipelineContext, clients: &AppClients) -> Result<()> {
        analyze_file(ctx, &clients.validation_semaphore).await
    }
}

async fn analyze_file(
    ctx: &mut PipelineContext,
    semaphore: &std::sync::Arc<tokio::sync::Semaphore>,
) -> Result<()> {
    let path = ctx
        .local_path
        .clone()
        .ok_or_else(|| anyhow!("Brak lokalnego pliku"))?;
    let name = ctx
        .file_name
        .as_deref()
        .filter(|name| !name.is_empty())
        .or(ctx.object_name.as_deref())
        .unwrap_or("")
        .to_string();
    let permit = semaphore.clone().acquire_owned().await?;
    // The permit lives inside the blocking task, including when its caller is cancelled.
    let result = tokio::task::spawn_blocking(move || -> anyhow::Result<_> {
        let _permit = permit;
        let detected_mime = mimetype_detector::detect_file(&path)?.to_string();
        let report = validate_file(&path, &name, &detected_mime, &ValidationConfig::default())?;
        let sha256 = if report.is_rejected() {
            None
        } else {
            use sha2::{Digest, Sha256};
            use std::io::Read;
            let mut file = fs::File::open(&path)?;
            let mut hasher = Sha256::new();
            let mut buf = [0u8; 65536];
            loop {
                let n = file.read(&mut buf)?;
                if n == 0 {
                    break;
                }
                hasher.update(&buf[..n]);
            }
            Some(hex::encode(hasher.finalize()))
        };
        Ok((report, sha256))
    })
    .await;
    let (report, sha256) = match result {
        Ok(Ok(result)) => result,
        result => {
            ctx.validation = Some(metadata_harvester::file_validation::ValidationReport::failed());
            return Err(anyhow!("Walidacja pliku nie powiodła się: {:?}", result));
        }
    };
    ctx.mime_type = report.verified_mime.clone();
    let rejected = report.is_rejected().then(|| rejected_message(&report));
    ctx.validation = Some(report);
    if let Some(error) = rejected {
        return Err(error);
    }
    ctx.sha256 = sha256;
    Ok(())
}

pub struct InsertFingerprintStep;
#[async_trait]
impl Step for InsertFingerprintStep {
    fn name(&self) -> &'static str {
        "InsertFingerprintStep"
    }
    async fn execute(&self, ctx: &mut PipelineContext, clients: &AppClients) -> Result<()> {
        let sha256 = ctx.sha256.as_ref().unwrap();
        let size = ctx.size_bytes.unwrap();
        clients
            .db
            .insert_fingerprint(ctx.file_id, &ctx.original_uri, sha256, size)
            .await?;
        info!("Fingerprint zapisany dla pliku: {}", ctx.file_id);
        Ok(())
    }
}

pub struct FetchSourceFileMetaStep;
#[async_trait]
impl Step for FetchSourceFileMetaStep {
    fn name(&self) -> &'static str {
        "FetchSourceFileMetaStep"
    }
    async fn execute(&self, ctx: &mut PipelineContext, clients: &AppClients) -> Result<()> {
        let dir = ctx.dir_path.as_ref().unwrap();
        let name = ctx.file_name.as_ref().unwrap();
        let meta = clients.db.get_source_file_meta(dir, name).await?;
        ctx.source_meta = Some(meta);
        Ok(())
    }
}

pub struct BuildAndSaveMetadataStep;
#[async_trait]
impl Step for BuildAndSaveMetadataStep {
    fn name(&self) -> &'static str {
        "BuildAndSaveMetadataStep"
    }
    async fn execute(&self, ctx: &mut PipelineContext, clients: &AppClients) -> Result<()> {
        let meta = build_metadata(ctx, &clients.cli);
