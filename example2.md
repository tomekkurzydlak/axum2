#[derive(Debug, Serialize)]
pub struct SearchFilter {
    pub filter_type: String,
    pub filter_value: String,
    pub display_value: Option<String>,
    pub document_count: Option<i64>,
}

#[derive(Debug, Serialize)]
pub struct SearchFiltersResp {
    pub filters: Vec<SearchFilter>,
}

#[derive(Clone, Copy)]
enum SearchScope {
    All,
    Keywords,
}

impl SearchScope {
    fn parse(value: Option<&str>) -> actix_web::Result<Self> {
        match value.unwrap_or("all").to_ascii_lowercase().as_str() {
            "all" | "search_vector" => Ok(Self::All),
            "keywords" | "keywords_vector" => Ok(Self::Keywords),
            _ => Err(actix_web::error::ErrorBadRequest(
                "scope must be all or keywords",
            )),
        }
    }

    fn vector_column(self) -> &'static str {
        match self {
            Self::All => "search_vector",
            Self::Keywords => "keywords_vector",
        }
    }
}

#[derive(Clone, Copy)]
enum SearchMode {
    And,
    Or,
}

impl SearchMode {
    fn parse(value: Option<&str>) -> actix_web::Result<Self> {
        match value.unwrap_or("and").to_ascii_lowercase().as_str() {
            "and" => Ok(Self::And),
            "or" => Ok(Self::Or),
            _ => Err(actix_web::error::ErrorBadRequest("mode must be and or or")),
        }
    }

    fn operator(self) -> &'static str {
        match self {
            Self::And => " & ",
            Self::Or => " | ",
        }
    }
}

fn build_tsquery(q: &str, mode: SearchMode) -> actix_web::Result<String> {
    let normalized = build_normalized_token_stream(q);
    let tokens = normalized
        .split_whitespace()
        .filter(|token| token.len() > 1)
        .collect::<Vec<_>>();

    if tokens.is_empty() {
        return Err(actix_web::error::ErrorBadRequest(
            "q has no searchable tokens",
        ));
    }

    Ok(tokens.join(mode.operator()))
}
==
run_search(
        pg,
        body.q.clone(),
        body.limit,
        body.offset,
        body.scope.as_deref(),
        body.mode.as_deref(),
        body.status.as_deref(),
        body.business_area.as_deref(),
    )
    .await
}

#[get("/api/v1/search")]
pub async fn search_get(
    pg: web::Data<tokio::sync::RwLock<Option<Pg>>>,
    query: web::Query<SearchQuery>,
) -> actix_web::Result<HttpResponse> {
    run_search(
        pg,
        query.q.clone(),
        query.limit,
        query.offset,
        query.scope.as_deref(),
        query.mode.as_deref(),
        query.status.as_deref(),
        query.business_area.as_deref(),
    )
    .await
}

async fn run_search(
    pg: web::Data<tokio::sync::RwLock<Option<Pg>>>,
    raw_q: String,
    limit: Option<i64>,
    offset: Option<i64>,
    scope: Option<&str>,
    mode: Option<&str>,
    status: Option<&str>,
    business_area: Option<&str>,
) -> actix_web::Result<HttpResponse> {
===
let limit = limit.unwrap_or(50).clamp(1, 200);
    let offset = offset.unwrap_or(0).max(0);
    let scope = SearchScope::parse(scope)?;
    let mode = SearchMode::parse(mode)?;
    let tsquery = build_tsquery(&q, mode)?;
    let vector_column = scope.vector_column();
===
#[utoipa::path(
    get,
    path = "/api/v1/search",
    params(
        ("q" = String, Query, description = "Search phrase"),
        ("scope" = Option<String>, Query, description = "all/search_vector or keywords/keywords_vector"),
        ("mode" = Option<String>, Query, description = "and or or"),
        ("status" = Option<String>, Query, description = "Status filter"),
        ("business_area" = Option<String>, Query, description = "Business area filter"),
        ("limit" = Option<i64>, Query, description = "Page size"),
        ("offset" = Option<i64>, Query, description = "Page offset")
    ),
    security(("bearer_auth" = [])),
    responses(
        (status = 200, description = "Search results", body = SearchRespDoc),
        (status = 400, description = "Bad request", body = ErrorResp),
        (status = 401, description = "Unauthorized", body = ErrorResp),
        (status = 403, description = "Forbidden", body = ErrorResp),
        (status = 500, description = "Internal server error", body = ErrorResp)
    )
)]
fn search_get_doc() {}

#[utoipa::path(
    get,
    path = "/api/v1/search/filters",
    security(("bearer_auth" = [])),
    responses(
        (status = 200, description = "Search filters", body = SearchFiltersRespDoc),
        (status = 401, description = "Unauthorized", body = ErrorResp),
        (status = 403, description = "Forbidden", body = ErrorResp),
        (status = 500, description = "Internal server error", body = ErrorResp)
    )
)]
fn search_filters_doc() {}
