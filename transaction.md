projection

use anyhow::Context;
use chrono::{DateTime, Utc};
use deadpool_postgres::Pool;
use serde_json::Value;
use std::collections::{BTreeSet, HashSet};
use tokio_postgres::Transaction;
use tracing::{debug, error, info};

use crate::config::AppConfig;
use crate::normalize::{build_normalized_token_stream, normalize_text};

pub async fn refresh_filters_full(pool: &Pool) -> anyhow::Result<()> {
    let mut client = pool.get().await?;
    let tx = client.transaction().await?;
    tx.execute(
        r#"
        WITH source_values AS (
            SELECT
                'status'::text AS filter_type,
                t.status AS filter_value,
                t.status AS display_value
            FROM schama_12.document t
            WHERE t.status IS NOT NULL AND btrim(t.status) <> ''
            GROUP BY t.status

            UNION ALL

            SELECT
                'business_area'::text AS filter_type,
                t.business_area AS filter_value,
                t.business_area AS display_value
            FROM schama_12.document t
            WHERE t.business_area IS NOT NULL AND btrim(t.business_area) <> ''
            GROUP BY t.business_area
        ),
        upserted AS (
            INSERT INTO schama_13.filters (
                filter_type, filter_value, display_value, document_count, updated_ts
            )
            SELECT
                s.filter_type,
                s.filter_value,
                s.display_value,
                NULL::bigint,
                now()
            FROM source_values s
            ON CONFLICT (filter_type, filter_value) DO UPDATE SET
                display_value = EXCLUDED.display_value,
                updated_ts = now()
            RETURNING 1
        )
        DELETE FROM schama_13.filters t
        WHERE t.filter_type IN ('status', 'business_area')
          AND NOT EXISTS (
            SELECT 1
            FROM source_values s
            WHERE s.filter_type = t.filter_type
              AND s.filter_value = t.filter_value
          )
        "#,
        &[],
    )
    .await?;
    tx.commit().await?;
    Ok(())
}

pub async fn run_projection(pool: &Pool, cfg: &AppConfig) -> anyhow::Result<()> {
    let start = std::time::Instant::now();
    let mut projected = 0usize;
    let mut failed = 0usize;

    loop {
        let mut client = pool.get().await?;
        let tx = client.transaction().await?;
        let batch_start = std::time::Instant::now();
        let pick_start = std::time::Instant::now();
        let rows = pick_batch(&tx, cfg.batch_size).await?;
        let pick_ms = pick_start.elapsed().as_millis() as u64;
        let count = rows.len();
        if count == 0 {
            tx.commit().await?;
            break;
        }
        info!(batch_size = count, "picked records for projection");
        debug!(batch_size = count, "batch processing started");

        let process_start = std::time::Instant::now();
        let mut batch_projected = 0usize;
        let mut batch_failed = 0usize;
        let mut hashes_for_recalc: HashSet<String> = HashSet::new();
        let mut projected_ids: Vec<i64> = Vec::with_capacity(count);
        let mut failed_ids: Vec<i64> = Vec::new();
        let mut status_filters: BTreeSet<String> = BTreeSet::new();
        let mut business_area_filters: BTreeSet<String> = BTreeSet::new();

        for (index, row) in rows.into_iter().enumerate() {
            let file_id: i64 = row.get("file_id");
            let file_meta: Value = row.get("file_meta");
            let control_conversion_status: Option<String> = row.get("control_conversion_status");
            debug!(file_id, "processing file");
            let savepoint = format!("sp_rec_{index}");
            tx.batch_execute(&format!("SAVEPOINT {savepoint}")).await?;

            match process_record(&tx, file_id, &file_meta, control_conversion_status.as_deref()).await {
                Ok(outcome) => {
                    projected += 1;
                    batch_projected += 1;
                    if let Some(hash) = outcome.hash_sha256 {
                        hashes_for_recalc.insert(hash);
                    }
                    if let Some(status) = outcome.status {
                        status_filters.insert(status);
                    }
                    if let Some(business_area) = outcome.business_area {
                        business_area_filters.insert(business_area);
                    }
                    projected_ids.push(file_id);
                    tx.batch_execute(&format!("RELEASE SAVEPOINT {savepoint}")).await?;
                }
                Err(err) => {
                    failed += 1;
                    batch_failed += 1;
                    error!(file_id, error = %err, "record projection failed");
                    tx.batch_execute(&format!("ROLLBACK TO SAVEPOINT {savepoint}")).await?;
                    failed_ids.push(file_id);
                    tx.batch_execute(&format!("RELEASE SAVEPOINT {savepoint}")).await?;
                }
            }
        }
        let process_ms = process_start.elapsed().as_millis() as u64;

        let duplicates_start = std::time::Instant::now();
        for (index, hash) in hashes_for_recalc.iter().enumerate() {
            let savepoint = format!("sp_dup_{index}");
            tx.batch_execute(&format!("SAVEPOINT {savepoint}")).await?;
            if let Err(err) = recalculate_duplicates_for_hash(&tx, hash).await {
                error!(hash, error = %err, "duplicate recalculation failed");
                tx.batch_execute(&format!("ROLLBACK TO SAVEPOINT {savepoint}")).await?;
            }
            tx.batch_execute(&format!("RELEASE SAVEPOINT {savepoint}")).await?;
        }
        let duplicates_ms = duplicates_start.elapsed().as_millis() as u64;

        let status_start = std::time::Instant::now();
        mark_status_projected_batch(&tx, &projected_ids).await?;
        mark_status_failed_batch(&tx, &failed_ids).await?;
        let status_update_ms = status_start.elapsed().as_millis() as u64;
        let status_filter_values: Vec<String> = status_filters.into_iter().collect();
        let business_area_filter_values: Vec<String> = business_area_filters.into_iter().collect();

        tx.commit().await?;
        let filters_start = std::time::Instant::now();
        if !status_filter_values.is_empty() || !business_area_filter_values.is_empty() {
            let filters_tx = client.transaction().await?;
            let filters_result = async {
                refresh_search_filters_batch(&filters_tx, "status", status_filter_values).await?;
                refresh_search_filters_batch(
                    &filters_tx,
                    "business_area",
                    business_area_filter_values,
                )
                .await?;
                filters_tx.commit().await?;
                Ok::<(), anyhow::Error>(())
            }
            .await;

            if let Err(err) = filters_result {
                error!(error = %err, "search filters refresh failed");
            }
        }
        let filters_update_ms = filters_start.elapsed().as_millis() as u64;
        let batch_duration_ms = batch_start.elapsed().as_millis() as u64;
        let records_per_sec = if batch_duration_ms > 0 {
            (count as f64) / (batch_duration_ms as f64 / 1000.0)
        } else {
            count as f64
        };
        debug!(
            batch_projected,
            batch_failed,
            unique_hashes = hashes_for_recalc.len(),
            duration_ms = batch_duration_ms,
            pick_ms,
            process_ms,
            duplicates_ms,
            status_update_ms,
            filters_update_ms,
            records_per_sec,
            "batch metrics"
        );
    }

    info!(
        projected,
        failed,
        duration_ms = start.elapsed().as_millis() as u64,
        "projection run finished"
    );
    Ok(())
}

async fn pick_batch(
    tx: &Transaction<'_>,
    batch_size: i64,
) -> anyhow::Result<Vec<tokio_postgres::Row>> {
    let rows = tx
        .query(
            r#"
            WITH picked AS (
                SELECT fm.file_id
                     , fc.status_cd AS control_conversion_status
                FROM schama_12.file_meta fm
                JOIN schama_11.file_control fc ON fc.file_id = fm.file_id
                WHERE fm.projection_status IN ('PENDING', 'FAILED')
                  AND fc.status_cd = 'COMPLETED'
                ORDER BY fm.tech_insert_ts ASC, fm.file_id ASC
                LIMIT $1
                FOR UPDATE SKIP LOCKED
            )
            SELECT fm.file_id, fm.file_meta, p.control_conversion_status
            FROM schama_12.file_meta fm
            JOIN picked p ON p.file_id = fm.file_id
            "#,
            &[&batch_size],
        )
        .await?;
    Ok(rows)
}

async fn process_record(
    tx: &Transaction<'_>,
    file_id: i64,
    file_meta: &Value,
    control_conversion_status: Option<&str>,
) -> anyhow::Result<ProcessOutcome> {
    let projection = map_projection(file_id, file_meta, control_conversion_status)?;
    upsert_projection(tx, &projection).await?;
    Ok(ProcessOutcome {
        hash_sha256: projection.hash_sha256.clone().filter(|v| !v.is_empty()),
        status: projection.status.clone().filter(|v| !v.is_empty()),
        business_area: projection.business_area.clone().filter(|v| !v.is_empty()),
    })
}

struct ProcessOutcome {
    hash_sha256: Option<String>,
    status: Option<String>,
    business_area: Option<String>,
}

struct ProjectionDocument {
    file_id: i64,
    source_system: Option<String>,
    schema_version: Option<String>,
    raw_file_uri: Option<String>,
    enriched_file_uri: Option<String>,
    source_file_url: Option<String>,
    source_file_name: Option<String>,
    source_file_version: Option<String>,
    source_folder_path: Option<String>,
    raw_folder_path: Option<String>,
    mime_type: Option<String>,
    size_bytes: Option<i64>,
    status: Option<String>,
    hash_sha256: Option<String>,
    creation_date_utc: Option<DateTime<Utc>>,
    modification_date_utc: Option<DateTime<Utc>>,
    effective_date: Option<chrono::NaiveDate>,
    expiration_date: Option<chrono::NaiveDate>,
    summary: Option<String>,
    keywords: Vec<String>,
    dqi_total: Option<f64>,
    dqi_description: Option<String>,
    perplexity_score: Option<f64>,
    conversion_quality_score: Option<f64>,
    is_converted: bool,
    conversion_provider: Option<String>,
    conversion_status: Option<String>,
    source_page_count: Option<i32>,
    page_count: Option<i32>,
    extraction_confidence: Option<f64>,
    is_deleted_from_source: bool,
    is_source_modified: bool,
    business_area: Option<String>,
    business_owner: Option<String>,
    business_category: Option<String>,
    keywords_normalized: Option<String>,
    summary_normalized: Option<String>,
    file_name_normalized: Option<String>,
    business_normalized: Option<String>,
    search_text_raw: Option<String>,
    search_text_normalized: Option<String>,
}

fn map_projection(
    file_id: i64,
    meta: &Value,
    control_conversion_status: Option<&str>,
) -> anyhow::Result<ProjectionDocument> {
    let source_file_url = json_str(meta, "/source_file/url");
    let raw_file_uri = json_str(meta, "/raw_file/uri");
    let source_file_name = json_str(meta, "/source_file/file_name")
        .or_else(|| json_str(meta, "/raw_file/file_name"));
    let keywords = json_str_array(meta, "/common_metadata/keywords");
    let summary = json_str(meta, "/common_metadata/summary");
    let business_area = json_str(meta, "/common_metadata/business_area");
    let business_owner = json_str(meta, "/common_metadata/business_owner");
    let business_category = json_str(meta, "/common_metadata/business_category");
    let ingestion_conversion_status = json_str(meta, "/ingestion_info/processing_status");
    let raw_status = json_str(meta, "/raw_file/status");

    let keywords_raw = keywords.join(" ");
    let summary_raw = summary.clone().unwrap_or_default();
    let business_raw = [business_area.clone(), business_owner.clone(), business_category.clone()]
        .into_iter()
        .flatten()
        .collect::<Vec<_>>()
        .join(" ");
    let file_name_raw = source_file_name.clone().unwrap_or_default();

    let keywords_normalized = maybe_normalized(&keywords_raw);
    let summary_normalized = maybe_normalized(&summary_raw);
    let file_name_normalized = maybe_normalized(&file_name_raw);
    let business_normalized = maybe_normalized(&business_raw);

    let search_text_raw = Some(
        [file_name_raw.as_str(), keywords_raw.as_str(), summary_raw.as_str(), business_raw.as_str()]
            .join(" ")
            .trim()
            .to_string(),
    );
    let search_text_normalized = search_text_raw
        .as_deref()
        .map(build_normalized_token_stream)
        .filter(|v| !v.is_empty());

    let is_converted_by_control = control_conversion_status == Some("COMPLETED");
    let _is_converted_by_ingestion = ingestion_conversion_status.as_deref() == Some("success");
    let is_converted = is_converted_by_control;
    // Alternatywnie (do decyzji): let is_converted = _is_converted_by_ingestion;

    Ok(ProjectionDocument {
        file_id,
        source_system: json_str(meta, "/source_file/source_system"),
        schema_version: json_str(meta, "/schema_version"),
        raw_file_uri: raw_file_uri.clone(),
        enriched_file_uri: json_str(meta, "/enriched_file/uri"),
        source_file_url: source_file_url.clone(),
        source_file_name,
        source_file_version: json_str(meta, "/source_file/version"),
        source_folder_path: source_file_url.as_deref().map(folder_path),
        raw_folder_path: raw_file_uri.as_deref().map(folder_path),
        mime_type: json_str(meta, "/raw_file/mime_type"),
        size_bytes: json_i64(meta, "/raw_file/size_bytes"),
        status: raw_status.clone(),
        hash_sha256: json_str(meta, "/raw_file/hash_sha256"),
        creation_date_utc: json_datetime(meta, "/raw_file/creation_date_utc"),
        modification_date_utc: json_datetime(meta, "/raw_file/modification_date_utc"),
        effective_date: json_date(meta, "/common_metadata/effective_date"),
        expiration_date: json_date(meta, "/common_metadata/expiration_date"),
        summary,
        keywords,
        dqi_total: json_f64(meta, "/common_metadata/quality_metadata/dqi_total"),
        dqi_description: json_str(meta, "/common_metadata/quality_metadata/description"),
        perplexity_score: json_f64(meta, "/common_metadata/quality_metadata/perplexity_score"),
        conversion_quality_score: json_f64(
            meta,
            "/common_metadata/quality_metadata/conversion_quality_score",
        ),
        is_converted,
        conversion_provider: json_str(meta, "/conversion/provider"),
        conversion_status: ingestion_conversion_status,
        source_page_count: json_i32(meta, "/conversion/source_page_count"),
        page_count: json_i32(meta, "/conversion/markdown_page_markers"),
        extraction_confidence: json_f64(meta, "/conversion/extraction_confidence"),
        is_deleted_from_source: raw_status.as_deref() == Some("Deleted"),
        is_source_modified: false,
        business_area,
        business_owner,
        business_category,
        keywords_normalized,
        summary_normalized,
        file_name_normalized,
        business_normalized,
        search_text_raw,
        search_text_normalized,
    })
}

fn maybe_normalized(input: &str) -> Option<String> {
    let normalized = normalize_text(input);
    if normalized.is_empty() {
        None
    } else {
        Some(normalized)
    }
}

async fn upsert_projection(tx: &Transaction<'_>, doc: &ProjectionDocument) -> anyhow::Result<()> {
    tx.execute(
        r#"
        INSERT INTO schama_12.document (
            file_id, source_system, schema_version, raw_file_uri, enriched_file_uri, source_file_url,
            source_file_name, source_file_version, source_folder_path, raw_folder_path, mime_type, size_bytes,
            status, hash_sha256, creation_date_utc, modification_date_utc, effective_date, expiration_date,
            summary, keywords, dqi_total, dqi_description, perplexity_score, conversion_quality_score,
            is_converted, conversion_provider, conversion_status, source_page_count, page_count, extraction_confidence,
            is_deleted_from_source, is_source_modified, business_area, business_owner, business_category,
            keywords_normalized, summary_normalized, file_name_normalized, business_normalized,
            search_text_raw, search_text_normalized, update_ts
        )
        VALUES (
            $1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13,$14,$15,$16,$17,$18,$19,$20,
            $21,$22,$23,$24,$25,$26,$27,$28,$29,$30,$31,$32,$33,$34,$35,$36,$37,$38,$39,$40,$41,now()
        )
        ON CONFLICT (file_id) DO UPDATE SET
            source_system = EXCLUDED.source_system,
            schema_version = EXCLUDED.schema_version,
            raw_file_uri = EXCLUDED.raw_file_uri,
            enriched_file_uri = EXCLUDED.enriched_file_uri,
            source_file_url = EXCLUDED.source_file_url,
            source_file_name = EXCLUDED.source_file_name,
            source_file_version = EXCLUDED.source_file_version,
            source_folder_path = EXCLUDED.source_folder_path,
            raw_folder_path = EXCLUDED.raw_folder_path,
            mime_type = EXCLUDED.mime_type,
            size_bytes = EXCLUDED.size_bytes,
            status = EXCLUDED.status,
            hash_sha256 = EXCLUDED.hash_sha256,
            creation_date_utc = EXCLUDED.creation_date_utc,
            modification_date_utc = EXCLUDED.modification_date_utc,
            effective_date = EXCLUDED.effective_date,
            expiration_date = EXCLUDED.expiration_date,
            summary = EXCLUDED.summary,
            keywords = EXCLUDED.keywords,
            dqi_total = EXCLUDED.dqi_total,
            dqi_description = EXCLUDED.dqi_description,
            perplexity_score = EXCLUDED.perplexity_score,
            conversion_quality_score = EXCLUDED.conversion_quality_score,
            is_converted = EXCLUDED.is_converted,
            conversion_provider = EXCLUDED.conversion_provider,
            conversion_status = EXCLUDED.conversion_status,
            source_page_count = EXCLUDED.source_page_count,
            page_count = EXCLUDED.page_count,
            extraction_confidence = EXCLUDED.extraction_confidence,
            is_deleted_from_source = EXCLUDED.is_deleted_from_source,
            is_source_modified = EXCLUDED.is_source_modified,
            business_area = EXCLUDED.business_area,
            business_owner = EXCLUDED.business_owner,
            business_category = EXCLUDED.business_category,
            keywords_normalized = EXCLUDED.keywords_normalized,
            summary_normalized = EXCLUDED.summary_normalized,
            file_name_normalized = EXCLUDED.file_name_normalized,
            business_normalized = EXCLUDED.business_normalized,
            search_text_raw = EXCLUDED.search_text_raw,
            search_text_normalized = EXCLUDED.search_text_normalized,
            update_ts = now()
        "#,
        &[
            &doc.file_id,
            &doc.source_system,
            &doc.schema_version,
            &doc.raw_file_uri,
            &doc.enriched_file_uri,
            &doc.source_file_url,
            &doc.source_file_name,
            &doc.source_file_version,
            &doc.source_folder_path,
            &doc.raw_folder_path,
            &doc.mime_type,
            &doc.size_bytes,
            &doc.status,
            &doc.hash_sha256,
            &doc.creation_date_utc,
            &doc.modification_date_utc,
            &doc.effective_date,
            &doc.expiration_date,
            &doc.summary,
            &doc.keywords,
            &doc.dqi_total,
            &doc.dqi_description,
            &doc.perplexity_score,
            &doc.conversion_quality_score,
            &doc.is_converted,
            &doc.conversion_provider,
            &doc.conversion_status,
            &doc.source_page_count,
            &doc.page_count,
            &doc.extraction_confidence,
            &doc.is_deleted_from_source,
            &doc.is_source_modified,
            &doc.business_area,
            &doc.business_owner,
            &doc.business_category,
            &doc.keywords_normalized,
            &doc.summary_normalized,
            &doc.file_name_normalized,
            &doc.business_normalized,
            &doc.search_text_raw,
            &doc.search_text_normalized,
        ],
    )
    .await
    .context("upsert t12006 failed")?;

    Ok(())
}

async fn recalculate_duplicates_for_hash(tx: &Transaction<'_>, hash_sha256: &str) -> anyhow::Result<()> {
    tx.execute(
        r#"
        WITH grp AS (
            SELECT file_id, creation_date_utc
            FROM schama_12.document
            WHERE hash_sha256 = $1
        ),
        oldest AS (
            SELECT file_id AS original_file_id
            FROM grp
            ORDER BY creation_date_utc NULLS LAST, file_id ASC
            LIMIT 1
        ),
        cnt AS (
            SELECT GREATEST(COUNT(*) - 1, 0) AS duplicate_count
            FROM grp
        )
        UPDATE schama_12.document t
        SET
            is_original = (t.file_id = (SELECT original_file_id FROM oldest)),
            original_file_id = (SELECT original_file_id FROM oldest),
            duplicate_count = (SELECT duplicate_count FROM cnt),
            similarity_percent = CASE WHEN (SELECT duplicate_count FROM cnt) > 0 THEN 100.00 ELSE NULL END
        WHERE t.hash_sha256 = $1
        "#,
        &[&hash_sha256],
    )
    .await?;
    Ok(())
}

async fn mark_status_projected_batch(tx: &Transaction<'_>, ids: &[i64]) -> anyhow::Result<()> {
    if ids.is_empty() {
        return Ok(());
    }
    tx.execute(
        r#"
        UPDATE schama_12.file_meta
        SET projection_status = 'REPLICATED'
        WHERE file_id = ANY($1::bigint[])
        "#,
        &[&ids],
    )
    .await?;
    Ok(())
}

async fn mark_status_failed_batch(tx: &Transaction<'_>, ids: &[i64]) -> anyhow::Result<()> {
    if ids.is_empty() {
        return Ok(());
    }
    tx.execute(
        r#"
        UPDATE schama_12.file_meta
        SET projection_status = 'FAILED'
        WHERE file_id = ANY($1::bigint[])
          AND projection_status = 'PENDING'
        "#,
        &[&ids],
    )
    .await?;
    Ok(())
}

fn json_str(v: &Value, ptr: &str) -> Option<String> {
    v.pointer(ptr)
        .and_then(Value::as_str)
        .map(str::trim)
        .filter(|s| !s.is_empty())
        .map(|s| s.to_string())
}

fn json_str_array(v: &Value, ptr: &str) -> Vec<String> {
    v.pointer(ptr)
        .and_then(Value::as_array)
        .map(|arr| {
            arr.iter()
                .filter_map(Value::as_str)
                .map(str::trim)
                .filter(|s| !s.is_empty())
                .map(ToString::to_string)
                .collect::<Vec<_>>()
        })
        .unwrap_or_default()
}

fn json_i64(v: &Value, ptr: &str) -> Option<i64> {
    v.pointer(ptr)
        .and_then(|x| x.as_i64().or_else(|| x.as_str()?.parse::<i64>().ok()))
}

fn json_i32(v: &Value, ptr: &str) -> Option<i32> {
    json_i64(v, ptr).and_then(|x| i32::try_from(x).ok())
}

fn json_f64(v: &Value, ptr: &str) -> Option<f64> {
    v.pointer(ptr)
        .and_then(|x| x.as_f64().or_else(|| x.as_str()?.parse::<f64>().ok()))
}

fn json_datetime(v: &Value, ptr: &str) -> Option<DateTime<Utc>> {
    v.pointer(ptr)
        .and_then(Value::as_str)
        .and_then(|s| chrono::DateTime::parse_from_rfc3339(s).ok())
        .map(|dt| dt.with_timezone(&Utc))
}

fn json_date(v: &Value, ptr: &str) -> Option<chrono::NaiveDate> {
    v.pointer(ptr)
        .and_then(Value::as_str)
        .and_then(|s| chrono::NaiveDate::parse_from_str(s, "%Y-%m-%d").ok())
}

fn folder_path(path: &str) -> String {
    match path.rsplit_once('/') {
        Some((left, _)) => left.to_string(),
        None => path.to_string(),schama_13.filters
    }
}

async fn refresh_search_filters_batch(
    tx: &Transaction<'_>,
    filter_type: &str,
    values: Vec<String>,
) -> anyhow::Result<()> {
    if values.is_empty() {
        return Ok(());
    }
    let display_values = values.clone();
    tx.execute(
        r#"
        INSERT INTO schama_13.filters (
            filter_type, filter_value, display_value, document_count, updated_ts
        )
        SELECT
            $1::text,
            src.filter_value,
            src.display_value,
            NULL::bigint,
            now()
        FROM unnest($2::text[], $3::text[]) AS src(filter_value, display_value)
        ON CONFLICT (filter_type, filter_value) DO UPDATE SET
            display_value = EXCLUDED.display_value,
            updated_ts = now()
        "#,
        &[&filter_type, &values, &display_values],
    )
    .await?;
    Ok(())
}

==

webapi

use actix_web::{HttpRequest, HttpResponse, Responder, get, post, web};
use serde::{Deserialize, Serialize};
use tracing::info;

use crate::AppState;
use crate::projection;
use crate::worker::wait_for_projection_idle;

#[derive(Deserialize)]
pub struct ProjectionStartRequest {
    pub app_name: String,
    pub command: String,
}

#[derive(Serialize)]
pub struct ProjectionStartResponse {
    pub accepted: bool,
    pub message: String,
}

#[derive(Serialize)]
pub struct RefreshFiltersResponse {
    pub ok: bool,
    pub message: String,
}

#[get("/healthz")]
async fn healthz() -> impl Responder {
    HttpResponse::Ok().finish()
}

#[get("/readyz")]
async fn readyz(state: web::Data<AppState>) -> impl Responder {
    match state.pool.get().await {
        Ok(client) => match client.simple_query("SELECT 1").await {
            Ok(_) => HttpResponse::Ok().finish(),
            Err(_) => HttpResponse::ServiceUnavailable().finish(),
        },
        Err(_) => HttpResponse::ServiceUnavailable().finish(),
    }
}

#[post("/v1/projection/start")]
async fn start_projection(
    req: HttpRequest,
    body: web::Json<ProjectionStartRequest>,
    state: web::Data<AppState>,
) -> impl Responder {
    if body.command != "start_projection" {
        return HttpResponse::BadRequest().json(ProjectionStartResponse {
            accepted: false,
            message: "invalid command".to_string(),
        });
    }

    if let Err(response) = authorize(&req, &body.app_name, &state) {
        return response;
    }

    info!(app_name = %body.app_name, "accepted start_projection request");
    state.worker.trigger().await;

    HttpResponse::Accepted().json(ProjectionStartResponse {
        accepted: true,
        message: "projection scheduled".to_string(),
    })
}

#[post("/v1/projection/refresh-filters")]
async fn refresh_filters(
    req: HttpRequest,
    body: web::Json<ProjectionStartRequest>,
    state: web::Data<AppState>,
) -> impl Responder {
    if body.command != "refresh_filters" {
        return HttpResponse::BadRequest().json(RefreshFiltersResponse {
            ok: false,
            message: "invalid command".to_string(),
        });
    }

    if let Err(response) = authorize(&req, &body.app_name, &state) {
        return response;
    }

    info!(app_name = %body.app_name, "accepted refresh_filters request");
    let _guard = state.full_refresh_lock.lock().await;
    wait_for_projection_idle(&state.worker).await;

    match projection::refresh_filters_full(&state.pool).await {
        Ok(_) => HttpResponse::Ok().json(RefreshFiltersResponse {
            ok: true,
            message: "filters refreshed".to_string(),
        }),
        Err(err) => HttpResponse::InternalServerError().json(RefreshFiltersResponse {
            ok: false,
            message: format!("refresh failed: {err}"),
        }),
    }
}

fn authorize(
    req: &HttpRequest,
    app_name: &str,
    state: &web::Data<AppState>,
) -> Result<(), HttpResponse> {
    let secret = req
        .headers()
        .get("X-Projection-Secret")
        .and_then(|h| h.to_str().ok())
        .unwrap_or_default();

    let Some(expected_secret) = state.cfg.allowed_callers.get(app_name) else {
        return Err(HttpResponse::Unauthorized().json(ProjectionStartResponse {
            accepted: false,
            message: "unknown caller".to_string(),
        }));
    };

    if secret != expected_secret {
        return Err(HttpResponse::Forbidden().json(ProjectionStartResponse {
            accepted: false,
            message: "invalid secret".to_string(),
        }));
    }
    Ok(())
}

pub fn routes(cfg: &mut web::ServiceConfig) {
    cfg.service(healthz)
        .service(readyz)
        .service(start_projection)
        .service(refresh_filters);
}


===

worker

use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};
use std::time::Duration;

use chrono::{Local, NaiveTime, TimeDelta};
use deadpool_postgres::Pool;
use tokio::sync::{Mutex, Notify};
use tracing::{error, info};

use crate::config::AppConfig;
use crate::projection::{refresh_filters_full, run_projection};

#[derive(Clone)]
pub struct WorkerHandle {
    notify: Arc<Notify>,
    pending: Arc<AtomicBool>,
    running: Arc<AtomicBool>,
}

impl WorkerHandle {
    pub async fn trigger(&self) {
        self.pending.store(true, Ordering::SeqCst);
        self.notify.notify_one();
    }

    pub fn is_projection_active_or_pending(&self) -> bool {
        self.pending.load(Ordering::SeqCst) || self.running.load(Ordering::SeqCst)
    }
}

pub struct ProjectionWorker;

impl ProjectionWorker {
    pub fn start(cfg: Arc<AppConfig>, pool: Pool) -> WorkerHandle {
        let notify = Arc::new(Notify::new());
        let pending = Arc::new(AtomicBool::new(false));
        let running = Arc::new(AtomicBool::new(false));

        let handle = WorkerHandle {
            notify: notify.clone(),
            pending: pending.clone(),
            running: running.clone(),
        };

        tokio::spawn(async move {
            loop {
                notify.notified().await;
                if !pending.load(Ordering::SeqCst) {
                    continue;
                }
                tokio::time::sleep(Duration::from_secs(cfg.debounce_seconds)).await;
                pending.store(false, Ordering::SeqCst);

                info!("starting projection run");
                running.store(true, Ordering::SeqCst);
                if let Err(err) = run_projection(&pool, &cfg).await {
                    error!(error = %err, "projection run failed");
                }
                running.store(false, Ordering::SeqCst);

                if pending.load(Ordering::SeqCst) {
                    tokio::time::sleep(Duration::from_secs(cfg.post_run_coalesce_seconds)).await;
                    notify.notify_one();
                }
            }
        });

        handle
    }
}

pub fn start_filter_refresh_scheduler(
    cfg: Arc<AppConfig>,
    pool: Pool,
    worker: WorkerHandle,
    full_refresh_lock: Arc<Mutex<()>>,
) {
    if !cfg.filter_refresh_enable {
        return;
    }
    let schedule_time = match NaiveTime::parse_from_str(&cfg.filter_refresh, "%H:%M") {
        Ok(time) => time,
        Err(err) => {
            error!(error = %err, value = %cfg.filter_refresh, "invalid FILTER_REFRESH format, expected HH:MM");
            return;
        }
    };
    info!(at = %cfg.filter_refresh, "daily filter refresh scheduler enabled");

    tokio::spawn(async move {
        loop {
            let sleep_for = duration_until_next(schedule_time);
            tokio::time::sleep(sleep_for).await;

            let _guard = full_refresh_lock.lock().await;
            wait_for_projection_idle(&worker).await;
            match refresh_filters_full(&pool).await {
                Ok(_) => info!("scheduled filters refresh completed"),
                Err(err) => error!(error = %err, "scheduled filters refresh failed"),
            }
        }
    });
}

pub async fn wait_for_projection_idle(worker: &WorkerHandle) {
    while worker.is_projection_active_or_pending() {
        tokio::time::sleep(Duration::from_secs(1)).await;
    }
}

fn duration_until_next(target: NaiveTime) -> Duration {
    let now = Local::now();
    let today = now.date_naive();
    let now_naive = now.naive_local();

    let mut next = today.and_time(target);
    if next <= now_naive {
        next += TimeDelta::days(1);
    }
    let diff = next - now_naive;
    Duration::from_secs(diff.num_seconds().max(1) as u64)
}

==

normalize

use regex::Regex;
use std::collections::BTreeSet;
use std::sync::OnceLock;
use unicode_normalization::UnicodeNormalization;

pub fn normalize_text(input: &str) -> String {
    let separated = split_compound_words(input);
    let lowered = separated.to_lowercase();
    let no_diacritics = remove_diacritics(&lowered);
    let cleaned = normalize_separators(&no_diacritics);
    cleaned.split_whitespace().collect::<Vec<_>>().join(" ")
}

pub fn build_normalized_token_stream(input: &str) -> String {
    let base = normalize_text(input);
    let mut out = BTreeSet::new();
    for token in base.split_whitespace() {
        out.insert(token.to_string());
        if let Some(lemma) = simple_lemma(token) {
            out.insert(lemma);
        }
        for variant in token_variants(token) {
            out.insert(variant);
        }
    }
    out.into_iter().collect::<Vec<_>>().join(" ")
}

fn split_compound_words(input: &str) -> String {
    static CAMEL_RE: OnceLock<Regex> = OnceLock::new();
    let camel = CAMEL_RE.get_or_init(|| Regex::new(r"([a-z0-9])([A-Z])").expect("regex"));
    let with_camel_split = camel.replace_all(input, "$1 $2").to_string();
    with_camel_split.replace(['_', '-'], " ")
}

fn remove_diacritics(input: &str) -> String {
    input
        .nfd()
        .filter(|c| !matches!(*c as u32, 0x0300..=0x036F))
        .collect::<String>()
}

fn normalize_separators(input: &str) -> String {
    static SEP_RE: OnceLock<Regex> = OnceLock::new();
    let regex = SEP_RE.get_or_init(|| Regex::new(r"[^\p{L}\p{N}\s]+").expect("regex"));
    regex.replace_all(input, " ").to_string()
}

fn simple_lemma(token: &str) -> Option<String> {
    let dict = [
        ("oplat", "oplata"),
        ("oplaty", "oplata"),
        ("prowizji", "prowizja"),
        ("prowizje", "prowizja"),
        ("rachunku", "rachunek"),
        ("rachunki", "rachunek"),
    ];
    if let Some((_, canonical)) = dict.iter().find(|(form, _)| *form == token) {
        return Some((*canonical).to_string());
    }
    for suffix in ["ami", "ach", "owego", "owej", "owie", "owiec", "ów", "ow", "ie", "y"] {
        if token.len() > suffix.len() + 2 && token.ends_with(suffix) {
            return Some(token.trim_end_matches(suffix).to_string());
        }
    }
    None
}

fn token_variants(token: &str) -> Vec<String> {
    let mut variants = Vec::new();
    if token.ends_with('a') && token.len() > 3 {
        variants.push(token.trim_end_matches('a').to_string());
    }
    if token.ends_with("ek") && token.len() > 4 {
        variants.push(token.trim_end_matches("ek").to_string());
    }
    variants
}

==
main
mod config;
mod db;
mod normalize;
mod projection;
mod web_api;
mod worker;

use actix_web::{App, HttpServer, web};
use anyhow::Context;
use std::sync::Arc;
use tokio::sync::Mutex;
use tracing::info;

use crate::config::AppConfig;
use crate::db::PgPool;
use crate::worker::{ProjectionWorker, WorkerHandle, start_filter_refresh_scheduler};

#[derive(Clone)]
pub struct AppState {
    pub cfg: Arc<AppConfig>,
    pub pool: PgPool,
    pub worker: WorkerHandle,
    pub full_refresh_lock: Arc<Mutex<()>>,
}

#[actix_web::main]
async fn main() -> anyhow::Result<()> {
    let cfg = Arc::new(AppConfig::from_env()?);
    init_tracing(&cfg)?;
    info!("starting projection service");
    info!(
        host = %cfg.server_host,
        port = cfg.server_port,
        pg_pool_max_size = 1,
        batch_size = cfg.batch_size,
        "configuration loaded"
    );

    let pool = db::connect_pg(&cfg.pg_dsn).await?;
    let worker = ProjectionWorker::start(cfg.clone(), pool.clone());
    let full_refresh_lock = Arc::new(Mutex::new(()));
    start_filter_refresh_scheduler(
        cfg.clone(),
        pool.clone(),
        worker.clone(),
        full_refresh_lock.clone(),
    );
    let state = AppState {
        cfg,
        pool,
        worker,
        full_refresh_lock,
    };
    let bind_host = state.cfg.server_host.clone();
    let bind_port = state.cfg.server_port;

    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(state.clone()))
            .configure(web_api::routes)
    })
    .bind((bind_host, bind_port))
    .with_context(|| "failed to bind HTTP server")?
    .run()
    .await
    .with_context(|| "http server failed")
}

fn init_tracing(cfg: &AppConfig) -> anyhow::Result<()> {
    let filter = tracing_subscriber::EnvFilter::try_new(cfg.log_level.clone())
        .unwrap_or_else(|_| tracing_subscriber::EnvFilter::new("info"));
    tracing_subscriber::fmt()
        .with_env_filter(filter)
        .try_init()
        .map_err(|err| anyhow::anyhow!(err.to_string()))?;
    Ok(())
}

==

db
use deadpool_postgres::{Manager, ManagerConfig, Pool, RecyclingMethod};
use tokio_postgres::NoTls;

pub type PgPool = Pool;

pub async fn connect_pg(pg_dsn: &str) -> anyhow::Result<PgPool> {
    let pg_cfg: tokio_postgres::Config = pg_dsn.parse()?;

    let mgr = Manager::from_config(
        pg_cfg,
        NoTls,
        ManagerConfig {
            recycling_method: RecyclingMethod::Fast,
        },
    );
    let pool = Pool::builder(mgr).max_size(1).build()?;
    Ok(pool)
}


==
config
use anyhow::{Context, anyhow};
use std::collections::HashMap;
use std::fs;

#[derive(Clone)]
pub struct AppConfig {
    pub server_host: String,
    pub server_port: u16,
    pub log_level: String,
    pub pg_dsn: String,
    pub allowed_callers: HashMap<String, String>,
    pub batch_size: i64,
    pub debounce_seconds: u64,
    pub post_run_coalesce_seconds: u64,
    pub filter_refresh_enable: bool,
    pub filter_refresh: String,
}

impl AppConfig {
    pub fn from_env() -> anyhow::Result<Self> {
        Ok(Self {
            server_host: env_or("SERVER_HOST", "0.0.0.0"),
            server_port: env_u16("SERVER_PORT", 8080)?,
            log_level: env_or("RUST_LOG", "info"),
            pg_dsn: build_pg_dsn()?,
            allowed_callers: parse_allowed_callers(&env_or("ALLOWED_CALLERS", ""))?,
            batch_size: env_i64("PROJECTION_BATCH_SIZE", 200)?,
            debounce_seconds: env_u64("PROJECTION_DEBOUNCE_SECONDS", 15)?,
            post_run_coalesce_seconds: env_u64("PROJECTION_POST_RUN_COALESCE_SECONDS", 15)?
                .min(60),
            filter_refresh_enable: env_bool("FILTER_REFRESH_ENABLE", false),
            filter_refresh: env_or("FILTER_REFRESH", "02:00"),
        })
    }
}

fn env_or(key: &str, default: &str) -> String {
    std::env::var(key).unwrap_or_else(|_| default.to_string())
}

fn env_u16(key: &str, default: u16) -> anyhow::Result<u16> {
    Ok(std::env::var(key)
        .ok()
        .and_then(|v| v.parse::<u16>().ok())
        .unwrap_or(default))
}

fn env_i64(key: &str, default: i64) -> anyhow::Result<i64> {
    Ok(std::env::var(key)
        .ok()
        .and_then(|v| v.parse::<i64>().ok())
        .unwrap_or(default))
}

fn env_u64(key: &str, default: u64) -> anyhow::Result<u64> {
    Ok(std::env::var(key)
        .ok()
        .and_then(|v| v.parse::<u64>().ok())
        .unwrap_or(default))
}

fn env_bool(key: &str, default: bool) -> bool {
    std::env::var(key)
        .ok()
        .map(|v| {
            let lowered = v.to_lowercase();
            lowered == "1" || lowered == "true" || lowered == "yes"
        })
        .unwrap_or(default)
}

fn parse_allowed_callers(raw: &str) -> anyhow::Result<HashMap<String, String>> {
    if raw.trim().is_empty() {
        return Ok(HashMap::new());
    }

    let mut result = HashMap::new();
    for pair in raw.split(',').map(str::trim).filter(|s| !s.is_empty()) {
        let (app, secret) = pair
            .split_once(':')
            .with_context(|| format!("invalid ALLOWED_CALLERS entry: {pair}"))?;
        if app.is_empty() || secret.is_empty() {
            return Err(anyhow!("empty app or secret in ALLOWED_CALLERS"));
        }
        result.insert(app.to_string(), secret.to_string());
    }
    Ok(result)
}

fn build_pg_dsn() -> anyhow::Result<String> {
    if is_true("ONPREM") {
        if let Some(dsn) = build_onprem_dsn()? {
            return Ok(dsn);
        }
        if let Ok(pg_dsn) = std::env::var("PG_DSN") {
            if !pg_dsn.trim().is_empty() {
                return Ok(pg_dsn);
            }
        }
        return Ok("host=localhost port=5432 user=postgres password=postgres dbname=postgres".to_string());
    }

    let host = env_or("PG_HOST", "127.0.0.1");
    let port = env_or("PG_PORT", "5432");
    let db = env_or("PG_DB", "postgres");
    let user = env_or("PG_USER", "postgres");
    let password = pg_password_or_file().unwrap_or_else(|| "postgres".to_string());
    Ok(format!(
        "host={host} port={port} user={user} password={password} dbname={db}"
    ))
}

fn build_onprem_dsn() -> anyhow::Result<Option<String>> {
    let pg_url = std::env::var("PG_URL").ok().filter(|v| !v.trim().is_empty());
    let pg_user = std::env::var("PG_USER").ok().filter(|v| !v.trim().is_empty());
    let password = pg_password_or_file();
    match (pg_url, pg_user, password) {
        (Some(url), Some(user), Some(password)) => {
            let encoded_password = urlencoding::encode(&password);
            if let Some(rest) = url.strip_prefix("postgresql://") {
                Ok(Some(format!(
                    "postgresql://{user}:{encoded_password}@{rest}"
                )))
            } else if let Some(rest) = url.strip_prefix("postgres://") {
                Ok(Some(format!("postgres://{user}:{encoded_password}@{rest}")))
            } else {
                Ok(Some(format!(
                    "postgresql://{user}:{encoded_password}@{url}"
                )))
            }
        }
        _ => Ok(None),
    }
}

fn pg_password_or_file() -> Option<String> {
    if let Ok(password) = std::env::var("PG_PASSWORD") {
        if !password.trim().is_empty() {
            return Some(password);
        }
    }

    let password_file = env_or("PG_PASSWORD_FILE", "/tmp/pgpasswd");
    fs::read_to_string(password_file)
        .ok()
        .map(|s| s.trim().to_string())
        .filter(|s| !s.is_empty())
}

fn is_true(name: &str) -> bool {
    std::env::var(name)
        .ok()
        .map(|v| {
            let lowered = v.to_lowercase();
            lowered == "1" || lowered == "true" || lowered == "yes"
        })
        .unwrap_or(false)
}


