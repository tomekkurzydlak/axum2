#[get("/api/v1/favorites")]
pub async fn favorites(
    pg: web::Data<tokio::sync::RwLock<Option<Pg>>>,
    req: HttpRequest,
    query: web::Query<FavoritesQuery>,
) -> actix_web::Result<HttpResponse> {
    let user_id = current_user_id(&req)?;
    let limit = query.limit.unwrap_or(50).clamp(1, 200);
    let offset = query.offset.unwrap_or(0).max(0);
    let pg = pg
        .read()
        .await
        .clone()
        .ok_or_else(|| actix_web::error::ErrorServiceUnavailable("database unavailable"))?;

    let rows = pg
        .query(
            r#"
            SELECT
              sd.file_id                                    AS file_id,
              fav.tech_insert_ts::text                      AS favorited_at,
              COALESCE(sd.source_file_uri, sd.raw_file_uri) AS source_uri,
              sd.source_file_name                           AS file_name,
              sd.mime_type                                  AS mime_type,
              sd.size_bytes                                 AS size_bytes,
              NULL::text                                    AS type_filesize,
              sd.last_modified_by                           AS last_modified_by,
              sd.enriched_file_uri                          AS processed_file_path,
              to_jsonb(sd) - 'search_vector' - 'keywords_vector' AS file_meta
            FROM schema1.tab_fav fav
            JOIN schema1.tab_search sd ON sd.file_id = fav.file_id
            WHERE fav.user_id = $1
            ORDER BY fav.tech_insert_ts DESC, sd.file_id ASC
            LIMIT $2 OFFSET $3
            "#,
            &[&user_id, &limit, &offset],
        )
        .await
        .map_err(|e| {
            if let Some(pg_err) = e.downcast_ref::<tokio_postgres::Error>() {
                log_pg_error(pg_err);
            } else {
                tracing::error!(error = ?e, "query failed");
            }
            actix_web::error::ErrorInternalServerError(e)
        })?;

    let mut hits = Vec::with_capacity(rows.len());
    for r in rows {
        hits.push(FavoriteFile {
            file_id: r.get::<_, i64>("file_id"),
            favorited_at: r.get::<_, String>("favorited_at"),
            source_uri: r.get::<_, Option<String>>("source_uri"),
            file_name: r.get::<_, Option<String>>("file_name"),
            mime_type: r.get::<_, Option<String>>("mime_type"),
            size_bytes: r.get::<_, Option<i64>>("size_bytes"),
            type_filesize: r.get::<_, Option<String>>("type_filesize"),
            last_modified_by: r.get::<_, Option<String>>("last_modified_by"),
            processed_file_path: r.get::<_, Option<String>>("processed_file_path"),
            file_meta: r.get::<_, serde_json::Value>("file_meta"),
        });
    }

    Ok(HttpResponse::Ok().json(FavoritesResp {
        count: hits.len(),
        hits,
    }))
}

#[post("/api/v1/files/{file_id}/favorite")]
pub async fn add_favorite(
    pg: web::Data<tokio::sync::RwLock<Option<Pg>>>,
    req: HttpRequest,
    path: web::Path<i64>,
) -> actix_web::Result<HttpResponse> {
    let user_id = current_user_id(&req)?;
    let file_id = path.into_inner();
    let pg = pg
        .read()
        .await
        .clone()
        .ok_or_else(|| actix_web::error::ErrorServiceUnavailable("database unavailable"))?;

    let rows = pg
        .query(
            r#"
            INSERT INTO schema1.tab_fav (
              user_id,
              file_id,
              tech_insert_ts,
              tech_update_ts
            )
            SELECT $1, $2, now(), now()
            WHERE EXISTS (
              SELECT 1
              FROM schema1.tab_search
              WHERE file_id = $2
            )
            ON CONFLICT (user_id, file_id)
            DO UPDATE SET tech_update_ts = now()
            RETURNING file_id
            "#,
            &[&user_id, &file_id],
        )
        .await
        .map_err(|e| {
            if let Some(pg_err) = e.downcast_ref::<tokio_postgres::Error>() {
                log_pg_error(pg_err);
            } else {
                tracing::error!(error = ?e, "query failed");
            }
            actix_web::error::ErrorInternalServerError(e)
        })?;

    rows.first()
        .ok_or_else(|| actix_web::error::ErrorNotFound("file not found"))?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "file_id": file_id,
        "is_favorite": true
    })))
}

#[delete("/api/v1/files/{file_id}/favorite")]
pub async fn delete_favorite(
    pg: web::Data<tokio::sync::RwLock<Option<Pg>>>,
    req: HttpRequest,
    path: web::Path<i64>,
) -> actix_web::Result<HttpResponse> {
    let user_id = current_user_id(&req)?;
    let file_id = path.into_inner();
    let pg = pg
        .read()
        .await
        .clone()
        .ok_or_else(|| actix_web::error::ErrorServiceUnavailable("database unavailable"))?;

    pg.query(
        r#"
        DELETE FROM schema1.tab_fav
        WHERE user_id = $1 AND file_id = $2
        "#,
        &[&user_id, &file_id],
    )
    .await
    .map_err(|e| {
        if let Some(pg_err) = e.downcast_ref::<tokio_postgres::Error>() {
            log_pg_error(pg_err);
        } else {
            tracing::error!(error = ?e, "query failed");
        }
        actix_web::error::ErrorInternalServerError(e)
    })?;

    Ok(HttpResponse::NoContent().finish())
}

====

struct FavoriteFileDoc {
    file_id: i64,
    favorited_at: String,
    source_uri: Option<String>,
    file_name: Option<String>,
    mime_type: Option<String>,
    size_bytes: Option<i64>,
    type_filesize: Option<String>,
    last_modified_by: Option<String>,
    processed_file_path: Option<String>,
    file_meta: serde_json::Value,
}

#[derive(Serialize, ToSchema)]
struct FavoritesRespDoc {
    count: usize,
    hits: Vec<FavoriteFileDoc>,
}

#[derive(Serialize, ToSchema)]
struct FavoriteStatusRespDoc {
    file_id: i64,
    is_favorite: bool,
}

#[derive(Serialize, ToSchema)]

==

    get,
    path = "/api/v1/favorites",
    params(
        ("limit" = Option<i64>, Query, description = "Maximum number of favorites"),
        ("offset" = Option<i64>, Query, description = "Pagination offset")
    ),
    security(("bearer_auth" = [])),
    responses(
        (status = 200, description = "Current user's favorite files", body = FavoritesRespDoc),
        (status = 401, description = "Unauthorized", body = ErrorResp),
        (status = 403, description = "Forbidden", body = ErrorResp),
        (status = 500, description = "Internal server error", body = ErrorResp)
    )
)]
fn favorites_doc() {}

#[utoipa::path(
    post,
    path = "/api/v1/files/{file_id}/favorite",
    params(
        ("file_id" = i64, Path, description = "File id from search_document")
    ),
    security(("bearer_auth" = [])),
    responses(
        (status = 200, description = "File marked as favorite", body = FavoriteStatusRespDoc),
        (status = 401, description = "Unauthorized", body = ErrorResp),
        (status = 403, description = "Forbidden", body = ErrorResp),
        (status = 404, description = "File not found", body = ErrorResp),
        (status = 500, description = "Internal server error", body = ErrorResp)
    )
)]
fn add_favorite_doc() {}

#[utoipa::path(
    delete,
    path = "/api/v1/files/{file_id}/favorite",
    params(
        ("file_id" = i64, Path, description = "File id from search_document")
    ),
    security(("bearer_auth" = [])),
    responses(
        (status = 204, description = "File removed from favorites"),
        (status = 401, description = "Unauthorized", body = ErrorResp),
        (status = 403, description = "Forbidden", body = ErrorResp),
        (status = 500, description = "Internal server error", body = ErrorResp)
    )
)]
fn delete_favorite_doc() {}

#[utoipa::path(
