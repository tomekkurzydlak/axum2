use axum::{
    extract::{Path, Query, State},
    http::{header::AUTHORIZATION, HeaderMap, StatusCode},
    response::{
        sse::{Event, KeepAlive, Sse},
        IntoResponse, Response,
    },
    routing::{get, post},
    Json, Router,
};
use chrono::{DateTime, Utc};
use devtray_shared::{
    AgentConfig, AgentUpdateInfo, CreateNotificationRequest, DeviceHeartbeatRequest,
    DeviceRegistrationRequest, NotificationMessage, NotificationSeverity,
};
use serde::{Deserialize, Serialize};
use sha2::{Digest, Sha256};
use sqlx::FromRow;
use std::{convert::Infallible, time::Duration};
use tracing::error;
use uuid::Uuid;

use crate::state::AppState;

const POLL_LIMIT: i64 = 50;

#[derive(Debug, Serialize)]
struct StatusResponse {
    status: &'static str,
}

#[derive(Debug, Serialize)]
struct ErrorResponse {
    error: String,
}

#[derive(Debug, Serialize)]
struct StoredConfigResponse {
    version: u64,
    checksum: String,
}

#[derive(Debug, Serialize)]
struct DeviceRegistrationResponse {
    device_id: String,
    device_token: String,
}

#[derive(Debug, Serialize)]
struct AdminDeviceResponse {
    id: String,
    hostname: String,
    username: String,
    agent_version: String,
    enabled: bool,
    created_at: String,
    last_seen_at: Option<String>,
}

#[derive(Debug, Serialize)]
struct PollNotificationsResponse {
    notifications: Vec<NotificationMessage>,
}

#[derive(Debug, Serialize)]
struct DictionarySnapshotResponse {
    fetched_at: String,
    payload: serde_json::Value,
}

#[derive(Debug, Deserialize)]
struct PollQuery {
    since_id: Option<String>,
}

#[derive(Debug, Deserialize)]
struct StreamQuery {
    since_id: Option<String>,
}

#[derive(Debug, Deserialize, FromRow)]
struct ConfigRow {
    version: i64,
    config_json: String,
}

#[derive(Debug, Deserialize, FromRow)]
struct DeviceAuthRow {
    id: String,
    enabled: i64,
}

#[derive(Debug, Deserialize, FromRow)]
struct AdminDeviceRow {
    id: String,
    hostname: String,
    username: String,
    agent_version: String,
    enabled: i64,
    created_at: String,
    last_seen_at: Option<String>,
}

#[derive(Debug, Deserialize, FromRow)]
struct NotificationRow {
    id: String,
    title: String,
    body: String,
    severity: String,
    created_at: String,
    expires_at: Option<String>,
}

#[derive(Debug, Clone)]
struct AuthenticatedDevice {
    id: String,
}

pub fn router() -> Router<AppState> {
    Router::new()
        .route("/health", get(health))
        .route("/ready", get(ready))
        .route("/api/v1/config", get(get_latest_config))
        .route("/api/v1/dictionaries", get(get_dictionaries))
        .route("/api/v1/agent/update/latest", get(get_agent_update_latest))
        .route("/api/v1/admin/config", post(post_admin_config))
        .route("/api/v1/devices/register", post(post_device_register))
        .route("/api/v1/devices/heartbeat", post(post_device_heartbeat))
        .route("/api/v1/admin/devices", get(get_admin_devices))
        .route(
            "/api/v1/admin/notifications",
            post(post_admin_notifications),
        )
        .route("/api/v1/notifications/poll", get(get_notifications_poll))
        .route(
            "/api/v1/notifications/stream",
            get(get_notifications_stream),
        )
        .route(
            "/api/v1/notifications/:id/ack",
            post(post_notifications_ack),
        )
}

async fn health() -> Json<StatusResponse> {
    Json(StatusResponse { status: "ok" })
}

async fn ready(State(state): State<AppState>) -> Response {
    match sqlx::query_scalar::<_, i64>("SELECT 1")
        .fetch_one(&state.pool)
        .await
    {
        Ok(_) => (StatusCode::OK, Json(StatusResponse { status: "ready" })).into_response(),
        Err(err) => {
            error!(error = %err, "database readiness check failed");
            json_error(StatusCode::SERVICE_UNAVAILABLE, "database is not ready")
        }
    }
}

async fn get_latest_config(State(state): State<AppState>) -> Response {
    match sqlx::query_as::<_, ConfigRow>(
        "SELECT version, config_json FROM config_versions ORDER BY version DESC, id DESC LIMIT 1",
    )
    .fetch_optional(&state.pool)
    .await
    {
        Ok(Some(row)) => match serde_json::from_str::<AgentConfig>(&row.config_json) {
            Ok(config) => Json(config).into_response(),
            Err(err) => {
                error!(error = %err, version = row.version, "invalid config JSON in database");
                json_error(
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "stored configuration is invalid",
                )
            }
        },
        Ok(None) => json_error(
            StatusCode::NOT_FOUND,
            "configuration not found; admin must publish one first",
        ),
        Err(err) => {
            error!(error = %err, "failed to query latest config");
            json_error(
                StatusCode::INTERNAL_SERVER_ERROR,
                "failed to read configuration",
            )
        }
    }
}

async fn get_agent_update_latest(State(state): State<AppState>) -> Response {
    match &state.config.agent_latest_update {
        Some(update) => Json::<AgentUpdateInfo>(update.clone()).into_response(),
        None => json_error(
            StatusCode::NOT_FOUND,
            "latest agent update is not configured",
        ),
    }
}

async fn get_dictionaries(State(state): State<AppState>, headers: HeaderMap) -> Response {
    if let Err(response) = authenticate_device(&headers, &state).await {
        return response;
    }

    let row = match sqlx::query_as::<_, (String, String)>(
        "SELECT payload_json, fetched_at FROM dictionary_snapshots WHERE key = 'default' LIMIT 1",
    )
    .fetch_optional(&state.pool)
    .await
    {
        Ok(Some(value)) => value,
        Ok(None) => {
            return json_error(
                StatusCode::SERVICE_UNAVAILABLE,
                "dictionary snapshot is not ready yet",
            );
        }
        Err(err) => {
            error!(error = %err, "failed to read dictionary snapshot");
            return json_error(
                StatusCode::INTERNAL_SERVER_ERROR,
                "failed to read dictionary snapshot",
            );
        }
    };

    let payload = match serde_json::from_str::<serde_json::Value>(&row.0) {
        Ok(value) => value,
        Err(err) => {
            error!(error = %err, "invalid dictionary snapshot JSON");
            return json_error(
                StatusCode::INTERNAL_SERVER_ERROR,
                "stored dictionary snapshot is invalid",
            );
        }
    };

    Json(DictionarySnapshotResponse {
        fetched_at: row.1,
        payload,
    })
    .into_response()
}

async fn post_admin_config(
    State(state): State<AppState>,
    headers: HeaderMap,
    Json(payload): Json<AgentConfig>,
) -> Response {
    if !is_admin_authorized(&headers, &state.config.admin_token) {
        return json_error(StatusCode::UNAUTHORIZED, "missing or invalid admin token");
    }

    if let Err(validation_error) = payload.validate() {
        return json_error(
            StatusCode::BAD_REQUEST,
            format!("invalid configuration: {}", validation_error),
        );
    }

    let latest_version =
        match sqlx::query_scalar::<_, Option<i64>>("SELECT MAX(version) FROM config_versions")
            .fetch_one(&state.pool)
            .await
        {
            Ok(value) => value,
            Err(err) => {
                error!(error = %err, "failed to query latest config version");
                return json_error(
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "failed to persist configuration",
                );
            }
        };

    if let Some(current) = latest_version {
        if payload.config_version <= current as u64 {
            return json_error(
                StatusCode::CONFLICT,
                format!(
                    "config_version must be greater than current version {}",
                    current
                ),
            );
        }
    }

    let config_json = match serde_json::to_string(&payload) {
        Ok(value) => value,
        Err(err) => {
            error!(error = %err, "failed to serialize configuration");
            return json_error(
                StatusCode::INTERNAL_SERVER_ERROR,
                "failed to persist configuration",
            );
        }
    };

    let checksum = sha256_hex(config_json.as_bytes());
    let created_at = Utc::now().to_rfc3339();

    if let Err(err) = sqlx::query(
        "INSERT INTO config_versions (version, config_json, checksum, created_at) VALUES (?1, ?2, ?3, ?4)",
    )
    .bind(payload.config_version as i64)
    .bind(&config_json)
    .bind(&checksum)
    .bind(&created_at)
    .execute(&state.pool)
    .await
    {
        error!(error = %err, "failed to insert config version");
        return json_error(
            StatusCode::INTERNAL_SERVER_ERROR,
            "failed to persist configuration",
        );
    }

    (
        StatusCode::CREATED,
        Json(StoredConfigResponse {
            version: payload.config_version,
            checksum,
        }),
    )
        .into_response()
}

async fn post_device_register(
    State(state): State<AppState>,
    Json(payload): Json<DeviceRegistrationRequest>,
) -> Response {
    if let Err(validation_error) = payload.validate() {
        return json_error(
            StatusCode::BAD_REQUEST,
            format!("invalid registration request: {}", validation_error),
        );
    }

    let device_id = Uuid::new_v4().to_string();
    let device_token = format!("dt_{}{}", Uuid::new_v4().simple(), Uuid::new_v4().simple());
    let token_hash = sha256_hex(device_token.as_bytes());
    let created_at = Utc::now().to_rfc3339();

    if let Err(err) = sqlx::query(
        "INSERT INTO devices (id, hostname, username, agent_version, token_hash, enabled, created_at, last_seen_at) VALUES (?1, ?2, ?3, ?4, ?5, 1, ?6, NULL)",
    )
    .bind(&device_id)
    .bind(&payload.device.hostname)
    .bind(&payload.device.username)
    .bind(&payload.device.agent_version)
    .bind(&token_hash)
    .bind(&created_at)
    .execute(&state.pool)
    .await
    {
        error!(error = %err, "failed to register device");
        return json_error(
            StatusCode::INTERNAL_SERVER_ERROR,
            "failed to register device",
        );
    }

    (
        StatusCode::CREATED,
        Json(DeviceRegistrationResponse {
            device_id,
            device_token,
        }),
    )
        .into_response()
}

async fn post_device_heartbeat(
    State(state): State<AppState>,
    headers: HeaderMap,
    Json(payload): Json<DeviceHeartbeatRequest>,
) -> Response {
    if let Err(validation_error) = payload.validate() {
        return json_error(
            StatusCode::BAD_REQUEST,
            format!("invalid heartbeat request: {}", validation_error),
        );
    }

    let device = match authenticate_device(&headers, &state).await {
        Ok(device) => device,
        Err(response) => return response,
    };

    if payload.device_id != device.id {
        return json_error(
            StatusCode::UNAUTHORIZED,
            "device token does not match device_id",
        );
    }

    let last_seen_at = Utc::now().to_rfc3339();

    if let Err(err) = sqlx::query(
        "UPDATE devices SET last_seen_at = ?1, agent_version = ?2, hostname = ?3, username = ?4 WHERE id = ?5",
    )
    .bind(&last_seen_at)
    .bind(&payload.agent_version)
    .bind(&payload.hostname)
    .bind(&payload.username)
    .bind(&payload.device_id)
    .execute(&state.pool)
    .await
    {
        error!(error = %err, device_id = %payload.device_id, "failed to update heartbeat");
        return json_error(StatusCode::INTERNAL_SERVER_ERROR, "failed to process heartbeat");
    }

    Json(StatusResponse { status: "ok" }).into_response()
}

async fn get_admin_devices(State(state): State<AppState>, headers: HeaderMap) -> Response {
    if !is_admin_authorized(&headers, &state.config.admin_token) {
        return json_error(StatusCode::UNAUTHORIZED, "missing or invalid admin token");
    }

    let rows = match sqlx::query_as::<_, AdminDeviceRow>(
        "SELECT id, hostname, username, agent_version, enabled, created_at, last_seen_at FROM devices ORDER BY created_at DESC",
    )
    .fetch_all(&state.pool)
    .await
    {
        Ok(value) => value,
        Err(err) => {
            error!(error = %err, "failed to query admin devices");
            return json_error(StatusCode::INTERNAL_SERVER_ERROR, "failed to read devices");
        }
    };

    let response: Vec<AdminDeviceResponse> = rows
        .into_iter()
        .map(|row| AdminDeviceResponse {
            id: row.id,
            hostname: row.hostname,
            username: row.username,
            agent_version: row.agent_version,
            enabled: row.enabled == 1,
            created_at: row.created_at,
            last_seen_at: row.last_seen_at,
        })
        .collect();

    Json(response).into_response()
}

async fn post_admin_notifications(
    State(state): State<AppState>,
    headers: HeaderMap,
    Json(payload): Json<CreateNotificationRequest>,
) -> Response {
    if !is_admin_authorized(&headers, &state.config.admin_token) {
        return json_error(StatusCode::UNAUTHORIZED, "missing or invalid admin token");
    }

    if let Err(validation_error) = payload.validate() {
        return json_error(
            StatusCode::BAD_REQUEST,
            format!("invalid notification request: {}", validation_error),
        );
    }

    let message = NotificationMessage {
        id: Uuid::new_v4().to_string(),
        title: payload.title,
        body: payload.body,
        severity: payload.severity,
        created_at: Utc::now(),
        expires_at: payload.expires_at,
    };

    if let Err(validation_error) = message.validate() {
        return json_error(
            StatusCode::BAD_REQUEST,
            format!("invalid notification: {}", validation_error),
        );
    }

    if let Err(err) = sqlx::query(
        "INSERT INTO notifications (id, title, body, severity, target_type, target_value, created_by, created_at, expires_at) VALUES (?1, ?2, ?3, ?4, 'all', NULL, 'admin', ?5, ?6)",
    )
    .bind(&message.id)
    .bind(&message.title)
    .bind(&message.body)
    .bind(notification_severity_to_str(&message.severity))
    .bind(message.created_at.to_rfc3339())
    .bind(message.expires_at.map(|value| value.to_rfc3339()))
    .execute(&state.pool)
    .await
    {
        error!(error = %err, "failed to create notification");
        return json_error(
            StatusCode::INTERNAL_SERVER_ERROR,
            "failed to create notification",
        );
    }

    (StatusCode::CREATED, Json(message)).into_response()
}

async fn get_notifications_poll(
    State(state): State<AppState>,
    headers: HeaderMap,
    Query(params): Query<PollQuery>,
) -> Response {
    let device = match authenticate_device(&headers, &state).await {
        Ok(device) => device,
        Err(response) => return response,
    };

    let now = Utc::now().to_rfc3339();

    let rows = if let Some(since_id) = params.since_id {
        let since_created_at = match sqlx::query_scalar::<_, Option<String>>(
            "SELECT created_at FROM notifications WHERE id = ?1",
        )
        .bind(&since_id)
        .fetch_one(&state.pool)
        .await
        {
            Ok(Some(value)) => value,
            Ok(None) => {
                return json_error(
                    StatusCode::BAD_REQUEST,
                    "since_id does not reference existing notification",
                )
            }
            Err(err) => {
                error!(error = %err, "failed to resolve since_id");
                return json_error(
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "failed to poll notifications",
                );
            }
        };

        match sqlx::query_as::<_, NotificationRow>(
            "SELECT id, title, body, severity, created_at, expires_at FROM notifications WHERE target_type = 'all' AND (expires_at IS NULL OR expires_at > ?1) AND (created_at > ?2 OR (created_at = ?2 AND id > ?3)) ORDER BY created_at ASC, id ASC LIMIT ?4",
        )
        .bind(&now)
        .bind(&since_created_at)
        .bind(&since_id)
        .bind(POLL_LIMIT)
        .fetch_all(&state.pool)
        .await
        {
            Ok(value) => value,
            Err(err) => {
                error!(error = %err, "failed to poll notifications with since_id");
                return json_error(
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "failed to poll notifications",
                );
            }
        }
    } else {
        let mut rows = match sqlx::query_as::<_, NotificationRow>(
            "SELECT id, title, body, severity, created_at, expires_at FROM notifications WHERE target_type = 'all' AND (expires_at IS NULL OR expires_at > ?1) ORDER BY created_at DESC, id DESC LIMIT ?2",
        )
        .bind(&now)
        .bind(POLL_LIMIT)
        .fetch_all(&state.pool)
        .await
        {
            Ok(value) => value,
            Err(err) => {
                error!(error = %err, "failed to poll latest notifications");
                return json_error(
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "failed to poll notifications",
                );
            }
        };

        rows.reverse();
        rows
    };

    let mut notifications = Vec::with_capacity(rows.len());
    for row in rows {
        match notification_from_row(row) {
            Ok(notification) => notifications.push(notification),
            Err(err_msg) => {
                error!(error = %err_msg, "failed to decode stored notification");
                return json_error(
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "stored notification is invalid",
                );
            }
        }
    }

    if let Err(err) = mark_deliveries_shown(&state.pool, &device.id, &notifications).await {
        error!(error = %err, device_id = %device.id, "failed to mark deliveries for poll");
        return json_error(
            StatusCode::INTERNAL_SERVER_ERROR,
            "failed to poll notifications",
        );
    }

    Json(PollNotificationsResponse { notifications }).into_response()
}

async fn get_notifications_stream(
    State(state): State<AppState>,
    headers: HeaderMap,
    Query(params): Query<StreamQuery>,
) -> Response {
    let device = match authenticate_device(&headers, &state).await {
        Ok(device) => device,
        Err(response) => return response,
    };

    if let Some(since_id) = &params.since_id {
        let exists = match sqlx::query_scalar::<_, Option<String>>(
            "SELECT id FROM notifications WHERE id = ?1",
        )
        .bind(since_id)
        .fetch_one(&state.pool)
        .await
        {
            Ok(value) => value.is_some(),
            Err(err) => {
                error!(error = %err, "failed to validate stream since_id");
                return json_error(
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "failed to open notifications stream",
                );
            }
        };

        if !exists {
            return json_error(
                StatusCode::BAD_REQUEST,
                "since_id does not reference existing notification",
            );
        }
    }

    let pool = state.pool.clone();
    let device_id = device.id;
    let stream = async_stream::stream! {
        let mut cursor = params.since_id;
        let mut ticker = tokio::time::interval(Duration::from_secs(2));

        loop {
            ticker.tick().await;

            let notifications = match fetch_notifications_since(&pool, cursor.as_deref()).await {
                Ok(items) => items,
                Err(err) => {
                    error!(error = %err, device_id = %device_id, "sse stream fetch failed");
                    break;
                }
            };

            if notifications.is_empty() {
                continue;
            }

            if let Err(err) = mark_deliveries_shown(&pool, &device_id, &notifications).await {
                error!(error = %err, device_id = %device_id, "failed to mark deliveries for sse");
                break;
            }

            for notification in notifications {
                let payload = match serde_json::to_string(&notification) {
                    Ok(value) => value,
                    Err(err) => {
                        error!(error = %err, "failed to serialize notification for sse");
                        continue;
                    }
                };

                cursor = Some(notification.id.clone());
                yield Ok::<Event, Infallible>(
                    Event::default()
                        .event("notification")
                        .id(notification.id)
                        .data(payload)
                );
            }
        }
    };

    Sse::new(stream)
        .keep_alive(
            KeepAlive::new()
                .interval(Duration::from_secs(15))
                .text("keep-alive"),
        )
        .into_response()
}

async fn post_notifications_ack(
    State(state): State<AppState>,
    headers: HeaderMap,
    Path(notification_id): Path<String>,
) -> Response {
    let device = match authenticate_device(&headers, &state).await {
        Ok(device) => device,
        Err(response) => return response,
    };

    let exists = match sqlx::query_scalar::<_, i64>(
        "SELECT COUNT(1) FROM notifications WHERE id = ?1 AND target_type = 'all'",
    )
    .bind(&notification_id)
    .fetch_one(&state.pool)
    .await
    {
        Ok(value) => value > 0,
        Err(err) => {
            error!(error = %err, notification_id = %notification_id, "failed to check notification existence");
            return json_error(
                StatusCode::INTERNAL_SERVER_ERROR,
                "failed to acknowledge notification",
            );
        }
    };

    if !exists {
        return json_error(StatusCode::NOT_FOUND, "notification not found");
    }

    let acknowledged_at = Utc::now().to_rfc3339();

    let result = sqlx::query(
        "INSERT INTO notification_deliveries (id, notification_id, device_id, status, shown_at, acknowledged_at, created_at) VALUES (?1, ?2, ?3, 'acknowledged', ?4, ?4, ?4) ON CONFLICT(notification_id, device_id) DO UPDATE SET status = 'acknowledged', acknowledged_at = excluded.acknowledged_at, shown_at = COALESCE(notification_deliveries.shown_at, excluded.shown_at)",
    )
    .bind(Uuid::new_v4().to_string())
    .bind(&notification_id)
    .bind(&device.id)
    .bind(&acknowledged_at)
    .execute(&state.pool)
    .await;

    if let Err(err) = result {
        error!(error = %err, notification_id = %notification_id, device_id = %device.id, "failed to acknowledge notification");
        return json_error(
            StatusCode::INTERNAL_SERVER_ERROR,
            "failed to acknowledge notification",
        );
    }

    Json(StatusResponse { status: "ok" }).into_response()
}

fn is_admin_authorized(headers: &HeaderMap, expected_token: &str) -> bool {
    matches!(bearer_token(headers), Some(token) if token == expected_token)
}

async fn authenticate_device(
    headers: &HeaderMap,
    state: &AppState,
) -> Result<AuthenticatedDevice, Response> {
    let Some(token) = bearer_token(headers) else {
        return Err(json_error(
            StatusCode::UNAUTHORIZED,
            "missing or invalid device token",
        ));
    };

    let token_hash = sha256_hex(token.as_bytes());

    let row = match sqlx::query_as::<_, DeviceAuthRow>(
        "SELECT id, enabled FROM devices WHERE token_hash = ?1 LIMIT 1",
    )
    .bind(&token_hash)
    .fetch_optional(&state.pool)
    .await
    {
        Ok(value) => value,
        Err(err) => {
            error!(error = %err, "failed to authenticate device");
            return Err(json_error(
                StatusCode::INTERNAL_SERVER_ERROR,
                "failed to authenticate device",
            ));
        }
    };

    let Some(row) = row else {
        return Err(json_error(
            StatusCode::UNAUTHORIZED,
            "missing or invalid device token",
        ));
    };

    if row.enabled != 1 {
        return Err(json_error(StatusCode::FORBIDDEN, "device is disabled"));
    }

    Ok(AuthenticatedDevice { id: row.id })
}

async fn fetch_notifications_since(
    pool: &sqlx::SqlitePool,
    since_id: Option<&str>,
) -> Result<Vec<NotificationMessage>, String> {
    let now = Utc::now().to_rfc3339();

    let rows = if let Some(since_id) = since_id {
        let since_created_at = sqlx::query_scalar::<_, Option<String>>(
            "SELECT created_at FROM notifications WHERE id = ?1",
        )
        .bind(since_id)
        .fetch_one(pool)
        .await
        .map_err(|e| format!("failed to resolve since_id: {}", e))?
        .ok_or_else(|| "since_id does not reference existing notification".to_string())?;

        sqlx::query_as::<_, NotificationRow>(
            "SELECT id, title, body, severity, created_at, expires_at FROM notifications WHERE target_type = 'all' AND (expires_at IS NULL OR expires_at > ?1) AND (created_at > ?2 OR (created_at = ?2 AND id > ?3)) ORDER BY created_at ASC, id ASC LIMIT ?4",
        )
        .bind(&now)
        .bind(&since_created_at)
        .bind(since_id)
        .bind(POLL_LIMIT)
        .fetch_all(pool)
        .await
        .map_err(|e| format!("failed to fetch notifications since_id: {}", e))?
    } else {
        let mut rows = sqlx::query_as::<_, NotificationRow>(
            "SELECT id, title, body, severity, created_at, expires_at FROM notifications WHERE target_type = 'all' AND (expires_at IS NULL OR expires_at > ?1) ORDER BY created_at DESC, id DESC LIMIT ?2",
        )
        .bind(&now)
        .bind(POLL_LIMIT)
        .fetch_all(pool)
        .await
        .map_err(|e| format!("failed to fetch latest notifications: {}", e))?;
        rows.reverse();
        rows
    };

    rows.into_iter()
        .map(notification_from_row)
        .collect::<Result<Vec<_>, _>>()
}

async fn mark_deliveries_shown(
    pool: &sqlx::SqlitePool,
    device_id: &str,
    notifications: &[NotificationMessage],
) -> Result<(), String> {
    let delivery_created_at = Utc::now().to_rfc3339();

    for notification in notifications {
        sqlx::query(
            "INSERT OR IGNORE INTO notification_deliveries (id, notification_id, device_id, status, shown_at, acknowledged_at, created_at) VALUES (?1, ?2, ?3, 'shown', ?4, NULL, ?5)",
        )
        .bind(Uuid::new_v4().to_string())
        .bind(&notification.id)
        .bind(device_id)
        .bind(&delivery_created_at)
        .bind(&delivery_created_at)
        .execute(pool)
        .await
        .map_err(|e| format!("insert delivery failed for {}: {}", notification.id, e))?;

        sqlx::query(
            "UPDATE notification_deliveries SET shown_at = COALESCE(shown_at, ?1), status = CASE WHEN status = 'acknowledged' THEN status ELSE 'shown' END WHERE notification_id = ?2 AND device_id = ?3",
        )
        .bind(&delivery_created_at)
        .bind(&notification.id)
        .bind(device_id)
        .execute(pool)
        .await
        .map_err(|e| format!("update delivery failed for {}: {}", notification.id, e))?;
    }

    Ok(())
}

fn notification_from_row(row: NotificationRow) -> Result<NotificationMessage, String> {
    let created_at = DateTime::parse_from_rfc3339(&row.created_at)
        .map_err(|e| format!("created_at parse error: {}", e))?
        .with_timezone(&Utc);

    let expires_at = match row.expires_at {
        Some(value) => Some(
            DateTime::parse_from_rfc3339(&value)
                .map_err(|e| format!("expires_at parse error: {}", e))?
                .with_timezone(&Utc),
        ),
        None => None,
    };

    let severity = notification_severity_from_str(&row.severity)
        .ok_or_else(|| format!("unknown notification severity: {}", row.severity))?;

    Ok(NotificationMessage {
        id: row.id,
        title: row.title,
        body: row.body,
        severity,
        created_at,
        expires_at,
    })
}

fn notification_severity_to_str(severity: &NotificationSeverity) -> &'static str {
    match severity {
        NotificationSeverity::Info => "Info",
        NotificationSeverity::Warning => "Warning",
        NotificationSeverity::Critical => "Critical",
    }
}

fn notification_severity_from_str(value: &str) -> Option<NotificationSeverity> {
    match value {
        "Info" => Some(NotificationSeverity::Info),
        "Warning" => Some(NotificationSeverity::Warning),
        "Critical" => Some(NotificationSeverity::Critical),
        _ => None,
    }
}

fn bearer_token(headers: &HeaderMap) -> Option<&str> {
    let value = headers.get(AUTHORIZATION)?;
    let value = value.to_str().ok()?;
    value.strip_prefix("Bearer ")
}

fn sha256_hex(input: &[u8]) -> String {
    format!("{:x}", Sha256::digest(input))
}

fn json_error(status: StatusCode, message: impl Into<String>) -> Response {
    (
        status,
        Json(ErrorResponse {
            error: message.into(),
        }),
    )
        .into_response()
}


==

use std::env;

use chrono::{DateTime, Utc};
use devtray_shared::AgentUpdateInfo;
use thiserror::Error;

#[derive(Debug, Clone)]
pub struct Config {
    pub http_bind_host: String,
    pub http_bind_port: u16,
    pub public_base_url: Option<String>,
    pub database_url: String,
    pub admin_token: String,
    pub agent_latest_update: Option<AgentUpdateInfo>,
    pub dictionary_sync_enabled: bool,
    pub dictionary_sync_interval_seconds: u64,
    pub dictionary_source_database_url: Option<String>,
    pub dictionary_source_queries_json: Option<String>,
}

impl Config {
    pub fn from_env() -> Result<Self, ConfigError> {
        let http_bind_host = env::var("HTTP_BIND_HOST").unwrap_or_else(|_| "127.0.0.1".to_string());

        let http_bind_port = match env::var("HTTP_BIND_PORT") {
            Ok(value) => value
                .parse::<u16>()
                .map_err(|_| ConfigError::InvalidPort(value))?,
            Err(_) => 8787,
        };

        let public_base_url = env::var("PUBLIC_BASE_URL").ok().and_then(|value| {
            if value.trim().is_empty() {
                None
            } else {
                Some(value)
            }
        });

        let database_url =
            env::var("DATABASE_URL").unwrap_or_else(|_| "sqlite://devnotify.db".to_string());

        let admin_token = env::var("ADMIN_TOKEN").map_err(|_| ConfigError::MissingAdminToken)?;
        if admin_token.trim().is_empty() {
            return Err(ConfigError::MissingAdminToken);
        }

        let agent_latest_update = match env::var("AGENT_LATEST_VERSION") {
            Ok(version) if !version.trim().is_empty() => {
                let notes = env::var("AGENT_LATEST_NOTES")
                    .map_err(|_| ConfigError::MissingAgentUpdateField("AGENT_LATEST_NOTES"))?;
                let download_url = env::var("AGENT_LATEST_DOWNLOAD_URL").map_err(|_| {
                    ConfigError::MissingAgentUpdateField("AGENT_LATEST_DOWNLOAD_URL")
                })?;
                let published_at_raw = env::var("AGENT_LATEST_PUBLISHED_AT").map_err(|_| {
                    ConfigError::MissingAgentUpdateField("AGENT_LATEST_PUBLISHED_AT")
                })?;
                let sha256 = env::var("AGENT_LATEST_SHA256")
                    .map_err(|_| ConfigError::MissingAgentUpdateField("AGENT_LATEST_SHA256"))?;

                let published_at = DateTime::parse_from_rfc3339(&published_at_raw)
                    .map_err(|_| ConfigError::InvalidAgentUpdatePublishedAt(published_at_raw))?
                    .with_timezone(&Utc);

                let update = AgentUpdateInfo {
                    version,
                    notes,
                    download_url,
                    published_at,
                    sha256,
                };
                update
                    .validate()
                    .map_err(ConfigError::InvalidAgentUpdatePayload)?;
                Some(update)
            }
            _ => None,
        };

        let dictionary_sync_enabled = env::var("DICTIONARY_SYNC_ENABLED")
            .ok()
            .map(|value| matches!(value.as_str(), "1" | "true" | "TRUE" | "yes" | "YES"))
            .unwrap_or(false);

        let dictionary_sync_interval_seconds = match env::var("DICTIONARY_SYNC_INTERVAL_SECONDS") {
            Ok(value) => value
                .parse::<u64>()
                .map_err(|_| ConfigError::InvalidDictionarySyncInterval(value))?,
            Err(_) => 300,
        };

        let dictionary_source_database_url = env::var("DICTIONARY_SOURCE_DATABASE_URL")
            .ok()
            .and_then(|value| {
                if value.trim().is_empty() {
                    None
                } else {
                    Some(value)
                }
            });

        let dictionary_source_queries_json = env::var("DICTIONARY_SOURCE_QUERIES_JSON")
            .ok()
            .and_then(|value| {
                if value.trim().is_empty() {
                    None
                } else {
                    Some(value)
                }
            });

        if dictionary_sync_enabled {
            if dictionary_source_database_url.is_none() {
                return Err(ConfigError::MissingDictionarySourceDatabaseUrl);
            }
            if dictionary_source_queries_json.is_none() {
                return Err(ConfigError::MissingDictionarySourceQueriesJson);
            }
        }

        Ok(Self {
            http_bind_host,
            http_bind_port,
            public_base_url,
            database_url,
            admin_token,
            agent_latest_update,
            dictionary_sync_enabled,
            dictionary_sync_interval_seconds,
            dictionary_source_database_url,
            dictionary_source_queries_json,
        })
    }

    pub fn bind_addr(&self) -> String {
        format!("{}:{}", self.http_bind_host, self.http_bind_port)
    }
}

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("HTTP_BIND_PORT has invalid value: {0}")]
    InvalidPort(String),
    #[error("ADMIN_TOKEN is required")]
    MissingAdminToken,
    #[error("missing required agent update env field: {0}")]
    MissingAgentUpdateField(&'static str),
    #[error("AGENT_LATEST_PUBLISHED_AT must be RFC3339, got: {0}")]
    InvalidAgentUpdatePublishedAt(String),
    #[error("invalid latest agent update payload: {0}")]
    InvalidAgentUpdatePayload(String),
    #[error("DICTIONARY_SYNC_INTERVAL_SECONDS has invalid value: {0}")]
    InvalidDictionarySyncInterval(String),
    #[error("DICTIONARY_SOURCE_DATABASE_URL is required when DICTIONARY_SYNC_ENABLED=true")]
    MissingDictionarySourceDatabaseUrl,
    #[error("DICTIONARY_SOURCE_QUERIES_JSON is required when DICTIONARY_SYNC_ENABLED=true")]
    MissingDictionarySourceQueriesJson,
}

==

use clap::{Parser, Subcommand, ValueEnum};
use devtray_shared::{AgentConfig, CreateNotificationRequest, NotificationSeverity};
use reqwest::blocking::{Client, Response};
use serde::Deserialize;
use std::{fs, process::ExitCode, time::Duration};

#[derive(Debug, Parser)]
#[command(name = "devnotify-admin")]
#[command(about = "CLI for DevNotify admin operations")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Debug, Subcommand)]
enum Commands {
    SendNotification {
        #[arg(long)]
        server_url: String,
        #[arg(long)]
        admin_token: String,
        #[arg(long)]
        title: String,
        #[arg(long)]
        body: String,
        #[arg(long, value_enum)]
        severity: SeverityArg,
    },
    UploadConfig {
        #[arg(long)]
        server_url: String,
        #[arg(long)]
        admin_token: String,
        #[arg(long)]
        file: String,
    },
    ListDevices {
        #[arg(long)]
        server_url: String,
        #[arg(long)]
        admin_token: String,
    },
}

#[derive(Debug, Clone, ValueEnum)]
enum SeverityArg {
    Info,
    Warning,
    Critical,
}

impl From<SeverityArg> for NotificationSeverity {
    fn from(value: SeverityArg) -> Self {
        match value {
            SeverityArg::Info => NotificationSeverity::Info,
            SeverityArg::Warning => NotificationSeverity::Warning,
            SeverityArg::Critical => NotificationSeverity::Critical,
        }
    }
}

#[derive(Debug, Deserialize)]
struct AdminDevice {
    id: String,
    hostname: String,
    username: String,
    agent_version: String,
    enabled: bool,
    created_at: String,
    last_seen_at: Option<String>,
}

type CliResult<T> = Result<T, String>;

fn main() -> ExitCode {
    let cli = Cli::parse();

    let result = match cli.command {
        Commands::SendNotification {
            server_url,
            admin_token,
            title,
            body,
            severity,
        } => send_notification(&server_url, &admin_token, title, body, severity.into()),
        Commands::UploadConfig {
            server_url,
            admin_token,
            file,
        } => upload_config(&server_url, &admin_token, &file),
        Commands::ListDevices {
            server_url,
            admin_token,
        } => list_devices(&server_url, &admin_token),
    };

    match result {
        Ok(()) => ExitCode::SUCCESS,
        Err(err) => {
            eprintln!("Error: {err}");
            ExitCode::FAILURE
        }
    }
}

fn send_notification(
    server_url: &str,
    admin_token: &str,
    title: String,
    body: String,
    severity: NotificationSeverity,
) -> CliResult<()> {
    let client = http_client()?;
    let url = format!(
        "{}/api/v1/admin/notifications",
        trim_trailing_slash(server_url)
    );

    let payload = CreateNotificationRequest {
        title,
        body,
        severity,
        expires_at: None,
    };

    let response = client
        .post(&url)
        .bearer_auth(admin_token)
        .json(&payload)
        .send()
        .map_err(|e| format!("request failed for {}: {}", url, e))?;

    ensure_success(response, "send-notification")?;
    println!("Notification sent successfully.");
    Ok(())
}

fn upload_config(server_url: &str, admin_token: &str, file: &str) -> CliResult<()> {
    let client = http_client()?;
    let url = format!("{}/api/v1/admin/config", trim_trailing_slash(server_url));

    let raw = fs::read_to_string(file)
        .map_err(|e| format!("failed to read config file {}: {}", file, e))?;

    let config: AgentConfig = serde_yaml::from_str(&raw)
        .map_err(|e| format!("failed to parse YAML config {}: {}", file, e))?;

    config
        .validate()
        .map_err(|e| format!("config validation failed for {}: {}", file, e))?;

    let response = client
        .post(&url)
        .bearer_auth(admin_token)
        .json(&config)
        .send()
        .map_err(|e| format!("request failed for {}: {}", url, e))?;

    ensure_success(response, "upload-config")?;
    println!(
        "Config uploaded successfully. Version: {}",
        config.config_version
    );
    Ok(())
}

fn list_devices(server_url: &str, admin_token: &str) -> CliResult<()> {
    let client = http_client()?;
    let url = format!("{}/api/v1/admin/devices", trim_trailing_slash(server_url));

    let response = client
        .get(&url)
        .bearer_auth(admin_token)
        .send()
        .map_err(|e| format!("request failed for {}: {}", url, e))?;

    let response = ensure_success(response, "list-devices")?;
    let devices = response
        .json::<Vec<AdminDevice>>()
        .map_err(|e| format!("failed to decode devices response: {}", e))?;

    if devices.is_empty() {
        println!("No registered devices.");
        return Ok(());
    }

    for device in devices {
        println!(
            "id={} hostname={} username={} version={} enabled={} created_at={} last_seen_at={}",
            device.id,
            device.hostname,
            device.username,
            device.agent_version,
            device.enabled,
            device.created_at,
            device.last_seen_at.unwrap_or_else(|| "-".to_string())
        );
    }

    Ok(())
}

fn http_client() -> CliResult<Client> {
    Client::builder()
        .timeout(Duration::from_secs(20))
        .build()
        .map_err(|e| format!("failed to initialize HTTP client: {}", e))
}

fn trim_trailing_slash(input: &str) -> &str {
    input.trim_end_matches('/')
}

fn ensure_success(response: Response, action: &str) -> CliResult<Response> {
    if response.status().is_success() {
        return Ok(response);
    }

    let status = response.status().as_u16();
    let body = response
        .text()
        .unwrap_or_else(|_| "<unable to read response body>".to_string());
    Err(format!(
        "{} failed with HTTP {}. Response: {}",
        action, status, body
    ))
}
