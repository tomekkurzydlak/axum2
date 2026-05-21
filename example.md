use google_cloud_auth::credentials::Builder as AuthBuilder;
use google_cloud_auth::credentials::service_account::Builder as ServiceAccountBuilder;
use google_cloud_auth::signer::Signer;
use google_cloud_storage::builder::storage::SignedUrlBuilder;
use google_cloud_storage::http;
use serde_json::Value;
use std::env::var;
use std::fs;
use std::time::Duration as StdDuration;
use time::format_description::FormatItem;
use time::macros::format_description;
use time::{Duration, OffsetDateTime};

static RFC3339_FORMAT: &[FormatItem<'_>] =
    format_description!("[year]-[month]-[day]T[hour]:[minute]:[second]Z");

#[derive(Debug)]
pub struct GcsObject {
    pub bucket: String,
    pub object: String,
}

pub struct SignedUrl {
    pub url: String,
    pub expires_at: String,
}

pub fn parse_gcs_uri(raw_file_uri: &str) -> anyhow::Result<GcsObject> {
    let uri = raw_file_uri.trim();
    if let Some(path) = uri.strip_prefix("gs://") {
        return parse_bucket_object(path, "invalid gs uri");
    }

    if let Some(path) = uri.strip_prefix("https://storage.googleapis.com/") {
        return parse_bucket_object(path, "invalid storage.googleapis.com uri");
    }

    if let Some(path) = uri.strip_prefix("https://") {
        if let Some((bucket, object)) = path.split_once(".storage.googleapis.com/") {
            if !bucket.is_empty() && !object.is_empty() {
                return Ok(GcsObject {
                    bucket: bucket.to_string(),
                    object: object.to_string(),
                });
            }
        }
    }

    Err(anyhow::anyhow!("unsupported gcs uri"))
}

pub async fn generate_signed_url(raw_file_uri: &str) -> anyhow::Result<SignedUrl> {
    let ttl_secs = var("GCS_SIGNED_URL_TTL_SECONDS")
        .ok()
        .and_then(|value| value.parse::<u64>().ok())
        .filter(|value| *value > 0)
        .unwrap_or(300)
        .min(3600);

    let object = parse_gcs_uri(raw_file_uri)?;
    let signer = build_signer()?;
    let url = SignedUrlBuilder::for_object(
        format!("projects/_/buckets/{}", object.bucket),
        object.object,
    )
    .with_method(http::Method::GET)
    .with_expiration(StdDuration::from_secs(ttl_secs))
    .sign_with(&signer)
    .await?;

    Ok(SignedUrl {
        url,
        expires_at: (OffsetDateTime::now_utc() + Duration::seconds(ttl_secs as i64))
            .format(RFC3339_FORMAT)?,
    })
}

fn parse_bucket_object(path: &str, error: &str) -> anyhow::Result<GcsObject> {
    let (bucket, object) = path
        .split_once('/')
        .ok_or_else(|| anyhow::anyhow!(error.to_string()))?;
    if bucket.is_empty() || object.is_empty() {
        return Err(anyhow::anyhow!(error.to_string()));
    }
    Ok(GcsObject {
        bucket: bucket.to_string(),
        object: object.to_string(),
    })
}

fn build_signer() -> anyhow::Result<Signer> {
    if let Ok(key_json) = var("GCS_SERVICE_ACCOUNT_KEY_JSON") {
        let service_account_key: Value = serde_json::from_str(&key_json)?;
        return Ok(ServiceAccountBuilder::new(service_account_key).build_signer()?);
    }

    if let Ok(key_path) = var("GCS_SERVICE_ACCOUNT_KEY_FILE") {
        let service_account_key: Value = serde_json::from_str(&fs::read_to_string(key_path)?)?;
        return Ok(ServiceAccountBuilder::new(service_account_key).build_signer()?);
    }

    Ok(AuthBuilder::default().build_signer()?)
}

==
#[derive(Serialize, ToSchema)]
struct PreviewUrlRespDoc {
    file_id: i64,
    file_name: Option<String>,
    mime_type: Option<String>,
    url: String,
    expires_at: String,
}
#[utoipa::path(
    post,
    path = "/api/v1/files/{file_id}/preview-url",
    params(
        ("file_id" = i64, Path, description = "File id from search_document")
    ),
    security(("bearer_auth" = [])),
    responses(
        (status = 200, description = "Signed GCS preview URL", body = PreviewUrlRespDoc),
        (status = 401, description = "Unauthorized", body = ErrorResp),
        (status = 403, description = "Forbidden", body = ErrorResp),
        (status = 404, description = "File not found", body = ErrorResp),
        (status = 500, description = "Internal server error", body = ErrorResp)
    )
)]
fn preview_url_doc() {}
