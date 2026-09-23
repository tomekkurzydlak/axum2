if !token.is_empty() { req = req.bearer_auth(token); }
                let mut res = req.send().await.map_err(|_| GatewayFailure { kind: "network", retry_after: None })?;
                let status = res.status();
                if status.as_u16() == 401 && !refresh { continue; }
                let retry_after = res.headers().get("retry-after").and_then(|h| h.to_str().ok())
                    .and_then(|s| s.parse::<u64>().ok());
                info!(http_status = status.as_u16(), "AI response headers");
                let mut bytes = Vec::new();
                while let Some(part) = res.chunk().await.map_err(|_| GatewayFailure { kind: "network", retry_after: None })? {
                    if bytes.len() + part.len() > 1_048_576 { return Err(anyhow!("response_too_large")); }
                    bytes.extend_from_slice(&part);
                }
                if !status.is_success() {
                    let root: Value = serde_json::from_slice(&bytes).unwrap_or(Value::Null);
                    let code = root.pointer("/error/code").and_then(Value::as_str).unwrap_or("");
                    let kind = if status.as_u16() == 413 { "payload_too_large" }
                        else if matches!(code, "context_length_exceeded" | "input_too_long") { "context_exceeded" }
                        else if status.as_u16() == 429 { "rate_limit" }
                        else if matches!(status.as_u16(), 502 | 503 | 504) { "server_unavailable" }
                        else { "http_error" };
                    return Err(GatewayFailure { kind, retry_after }.into());
                }
                let root: Value = serde_json::from_slice(&bytes).map_err(|_| anyhow!("invalid_response_json"))?;
                let finish = root.pointer("/choices/0/finish_reason").and_then(Value::as_str).unwrap_or("unknown");
                info!(finish_reason = finish, "AI generation finished");
                if finish == "length" { return Err(anyhow!("output_truncated")); }
                let content = root.pointer("/choices/0/message/content").and_then(Value::as_str)
                    .ok_or_else(|| anyhow!("missing_model_content"))?;
                return serde_json::from_str(content).map_err(|_| anyhow!("invalid_model_json"));
            }
            Err(anyhow!("authentication_failed"))
        };
        tokio::time::timeout(std::time::Duration::from_secs(seconds), operation).await
            .map_err(|_| GatewayFailure { kind: "timeout", retry_after: None })?
    }
