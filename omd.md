let result = tokio::time::timeout(duration, async {
        match process(&mut run, text).await {
            Ok(v) => Ok(v),
            Err(e) => {
                tracing::warn!(analysis_id = %id, error = %format!("{e:#}"), "Recovering intermediate analysis");
                run.incomplete.store(1, Ordering::SeqCst);
                let (summary, keywords) = recovered_notes(&run)?;
                let system = format!("{}\nInput contains PARTIAL intermediate notes. Summarize only available information; do not infer missing parts.", client.system_prompt());
                let user = serde_json::json!({"summary": summary, "keywords": keywords}).to_string();
                if run.fits(&system, &user, client.output_limit()) {
                    if let Ok(meta) = run.metadata(&system, &user).await { return Ok(meta); }
                }
                recovered_notes(&run)
            }
        }
    }).await;
    let result = match result {
        Ok(Ok(v)) => Ok(v),
        other => {
            run.incomplete.store(1, Ordering::SeqCst);
            recovered_notes(&run).or_else(|_| match other {
                Ok(Err(e)) => Err(e),
                Err(_) => Err(anyhow::anyhow!("analysis_budget_exceeded: deadline")),
                _ => unreachable!(),
            })
        }
    };
    let result = result.map(|(summary, keywords)| {
        if run.incomplete.load(Ordering::SeqCst) > 0 {
            (format!("[Analiza częściowa] {summary}"), keywords)
        } else {
            (summary, keywords)
        }
    });
    tracing::info!(file_id, analysis_id = %id, partial = run.incomplete.load(Ordering::SeqCst) > 0,
        success = result.is_ok(), "Document analysis finished");
    result.with_context(|| format!("AI analysis {} file_id {}", id, file_id))
}

// Bounded fallback uses only successfully parsed notes, even after a deadline.
fn recovered_notes(run: &Run<'_>) -> Result<(String, Vec<String>)> {
    let mut records = run.saved.lock().unwrap().clone();
    records.sort_by_key(|r| r["source_start"].as_u64());
    let mut summary = String::new();
    let mut keywords = Vec::<String>::new();
    for record in records {
        if let Some(s) = record["notes"]["summary"].as_str() {
            if summary.chars().count() < 2000 {
                if !summary.is_empty() {
                    summary.push(' ');
                }
                summary.extend(s.chars().take(2000 - summary.chars().count()));
            }
        }
        if let Some(items) = record["notes"]["keywords"].as_array() {
            for k in items.iter().filter_map(Value::as_str) {
                if keywords.len() < run.cfg.keywords
                    && !keywords.iter().any(|v| v.eq_ignore_ascii_case(k))
                {
                    keywords.push(k.to_owned());
                }
            }
        }
    }
    ensure!(!summary.trim().is_empty(), "no_intermediate_analysis");
    Ok((summary, keywords))
}
