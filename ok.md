let password_secret_ok = if let Ok(password_file) = std::env::var("") {
        std::fs::read_to_string(password_file)
            .ok()
            .map(|v| !v.trim().is_empty())
            .unwrap_or(false)
    } else if let Ok(password_env) = std::env::var("") {
        !password_env.trim().is_empty()
    } else {
        std::fs::read_to_string("")
            .ok()
            .map(|v| !v.trim().is_empty())
            .unwrap_or(false)
    };

    if !password_secret_ok {
        warn!("readiness failed: password file missing/empty");
        return HttpResponse::ServiceUnavailable().json(serde_json::json!({
            "status": "not_ready"
        }));
    }

    if !sqlproxy_ready(&state.cfg.pg_dsn).await {
        warn!("readiness failed: sqlproxy unavailable");
        return HttpResponse::ServiceUnavailable().json(serde_json::json!({
            "status": "not_ready"
        }));
    }

==

Ok(_) => HttpResponse::Ok().json(serde_json::json!({
                "status": "ok"
            })),
            Err(err) => {
                log_pg_error(&err);
                HttpResponse::ServiceUnavailable().json(serde_json::json!({
                    "status": "not_ready"
                }))
            }
        },

        Err(err) => {
            error!(error = ?err, "readiness failed: db pool checkout failed");
            HttpResponse::ServiceUnavailable().json(serde_json::json!({
                "status": "not_ready"
            }))
        }
    }
}

async fn sqlproxy_ready(pg_dsn: &str) -> bool {
    let addr = resolve_proxy_addr(pg_dsn);

    tokio::time::timeout(
        std::time::Duration::from_secs(2),
        tokio::net::TcpStream::connect(addr),
    )
    .await
    .ok()
    .and_then(Result::ok)
    .is_some()
}

fn resolve_proxy_addr(pg_dsn: &str) -> String {
    if let Ok(host) = std::env::var("PG_HOST") {
        let port = std::env::var("PG_PORT").unwrap_or_else(|_| "5432".to_string());
        return format!("{host}:{port}");
    }

    if let Ok(cfg) = pg_dsn.parse::<tokio_postgres::Config>() {
        if let Some(host) = cfg.get_hosts().first() {
            let port = cfg.get_ports().first().copied().unwrap_or(5432);
            if let tokio_postgres::config::Host::Tcp(host) = host {
                return format!("{host}:{port}");
            }
        }
    }

    "127.0.0.1:5432".to_string()
}

fn log_pg_error(e: &tokio_postgres::Error) {
    if let Some(db) = e.as_db_error() {
        error!(
            code = ?db.code(),
            message = %db.message(),
            detail = ?db.detail(),
            hint = ?db.hint(),
            schema = ?db.schema(),
            table = ?db.table(),
            column = ?db.column(),
            constraint = ?db.constraint(),
            "postgres db error"
        );
    } else {
        error!(error = ?e, "postgres client/transport error");
