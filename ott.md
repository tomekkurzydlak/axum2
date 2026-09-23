pub mod budget;
pub mod chunking;
pub mod config;
mod contracts;
mod prompts;

use crate::clients::ai_gateway::{AiGatewayClient, GatewayFailure};
use anyhow::{bail, ensure, Context, Result};
use config::Config;
use contracts::{final_meta, Notes};
use serde_json::Value;
use std::{
    collections::VecDeque,
    time::{Duration, Instant},
};

struct Run<'a> {
    client: &'a AiGatewayClient,
    cfg: Config,
    calls: usize,
    input_units: usize,
    id: String,
}
impl Run<'_> {
    fn fits(&self, system: &str, user: &str, output: u32) -> bool {
        let n = budget::units(system, user);
        n <= self.cfg.input_limit && n.saturating_add(output as usize) <= self.cfg.context_limit
    }
    async fn request(&mut self, system: &str, user: &str, output: u32) -> Result<Value> {
        ensure!(self.fits(system, user, output), "input_budget_exceeded");
        for attempt in 0..3u64 {
            ensure!(
                self.calls < self.cfg.max_calls,
                "analysis_budget_exceeded: max calls"
            );
            self.input_units = self.input_units.saturating_add(budget::units(system, user));
            ensure!(
                self.input_units <= self.cfg.max_input_units,
                "analysis_budget_exceeded: cumulative input"
            );
            self.calls += 1;
            let start = Instant::now();
            tracing::info!(analysis_id = %self.id, call = self.calls, attempt,
                model = self.client.model(), input_units = budget::units(system, user),
                counting = "utf8_bytes_conservative_not_exact_tokens", "AI call started");
            let result = self
                .client
                .request_json(
                    system,
                    user,
                    output,
                    self.cfg.max_body_bytes,
                    self.cfg.request_secs,
                )
                .await;
            tracing::info!(analysis_id = %self.id, call = self.calls,
                elapsed_ms = start.elapsed().as_millis() as u64, success = result.is_ok(), "AI call finished");
            match result {
                Ok(v) => return Ok(v),
                Err(e) => {
                    tracing::warn!(analysis_id = %self.id, call = self.calls, error = %e, "AI call failed");
                    let failure = e.downcast_ref::<GatewayFailure>();
                    let transient = failure
                        .map(|e| {
                            matches!(
                                e.kind,
                                "network" | "timeout" | "rate_limit" | "server_unavailable"
                            )
                        })
                        .unwrap_or(false);
                    if !transient || attempt == 2 {
                        return Err(e);
                    }
                    let delay = failure.and_then(|e| e.retry_after).unwrap_or(1 << attempt);
                    let jitter = (uuid::Uuid::new_v4().as_u128() % 500) as u64;
                    tokio::time::sleep(
                        Duration::from_secs(delay.min(self.cfg.total_secs))
                            + Duration::from_millis(jitter),
                    )
                    .await;
                }
            }
        }
        unreachable!()
    }
    async fn notes(&mut self, system: &str, user: &str) -> Result<Notes> {
        for attempt in 0..2 {
            let result = self
                .request(system, user, self.cfg.output)
                .await
                .and_then(Notes::parse);
            match result {
                Ok(n) => return Ok(n),
                Err(e) if attempt == 0 && e.downcast_ref::<GatewayFailure>().is_none() => continue,
                Err(e) => return Err(e),
            }
        }
        unreachable!()
    }
}
fn size_error(e: &anyhow::Error) -> bool {
    e.downcast_ref::<GatewayFailure>()
        .map(|e| matches!(e.kind, "payload_too_large" | "context_exceeded"))
        .unwrap_or(false)
}

pub async fn analyze(
    client: &AiGatewayClient,
    text: &str,
    file_id: i64,
) -> Result<(String, Vec<String>)> {
    let cfg = Config::from_env()?;
    if !cfg.enabled {
        return client.get_ai_meta(text).await;
    }
    ensure!(!text.trim().is_empty(), "empty_document");
    let id = uuid::Uuid::new_v4().to_string();
    tracing::info!(file_id, analysis_id = %id, bytes = text.len(), "Document analysis started");
    let duration = Duration::from_secs(cfg.total_secs);
    let mut run = Run {
        client,
        cfg,
        calls: 0,
        input_units: 0,
        id: id.clone(),
    };
    let result = tokio::time::timeout(duration, process(&mut run, text))
        .await
        .map_err(|_| {
            anyhow::anyhow!(
                "analysis_budget_exceeded: deadline analysis {} file_id {}",
                id,
                file_id
            )
        })?;
    result.with_context(|| format!("AI analysis {} file_id {}", id, file_id))
}

async fn process(run: &mut Run<'_>, text: &str) -> Result<(String, Vec<String>)> {
    let system = run.client.system_prompt().to_owned();
    // Avoid a full-document copy merely to decide which path to use.
    if text.len().saturating_add(system.len()).saturating_add(540) <= run.cfg.input_limit {
        let user = format!("Document (Markdown):\n\n{}", text);
        if run.fits(&system, &user, run.client.output_limit()) {
            match run.request(&system, &user, run.client.output_limit()).await {
                Ok(v) => return final_meta(v),
                Err(e) if size_error(&e) => {}
                Err(e) => return Err(e),
            }
        }
    }
    let reserve = prompts::MAP.len() + budget::OVERHEAD + 1200;
    ensure!(
        run.cfg.target > reserve + 256,
        "invalid_configuration: map budget"
    );
    let limit = run.cfg.target - reserve;
    let mut pending: VecDeque<_> = chunking::ranges(text, limit).into_iter().collect();
    let mut records = Vec::new();
    while let Some(range) = pending.pop_front() {
        let context = chunking::context_before(text, range.start, 1024);
        let user = format!(
            "{}\nSource bytes {}..{}:\n{}",
            context,
            range.start,
            range.end,
            &text[range.clone()]
        );
        match run.notes(prompts::MAP, &user).await {
            Ok(notes) => records.push(serde_json::json!({"source_start":range.start,"source_end":range.end,"notes":notes})),
            Err(e) if size_error(&e) && range.len() > 512 => {
                let sub = chunking::ranges(&text[range.clone()], range.len() / 2);
                for r in sub.into_iter().rev() { pending.push_front(range.start+r.start..range.start+r.end); }
            },
            Err(e) => return Err(e).context(format!("map source {}..{}", range.start, range.end)),
        }
    }
    let final_system = format!("{}{}", system, prompts::FINAL);
    let mut group_limit = run.cfg.target;
    for level in 0..run.cfg.max_depth {
        let all = serde_json::to_string(&records)?;
        if run.fits(&final_system, &all, run.client.output_limit()) {
            match run
                .request(&final_system, &all, run.client.output_limit())
                .await
            {
                Ok(v) => return final_meta(v),
                Err(e) if size_error(&e) => {
                    group_limit /= 2;
                }
                Err(e) => return Err(e).context("final synthesis"),
            }
        }
        ensure!(
            group_limit > prompts::REDUCE.len() + budget::OVERHEAD + 256,
            "reduction_budget_too_small"
        );
        let mut next = Vec::new();
        let mut start = 0;
        while start < records.len() {
            let mut end = start + 1;
            let first = serde_json::to_string(&records[start..end])?;
            ensure!(
                budget::units(prompts::REDUCE, &first) <= group_limit,
                "intermediate_result_too_large"
            );
            while end < records.len() {
                let candidate = serde_json::to_string(&records[start..end + 1])?;
                if budget::units(prompts::REDUCE, &candidate) > group_limit {
                    break;
                }
                end += 1;
            }
            let user = serde_json::to_string(&records[start..end])?;
            match run.notes(prompts::REDUCE, &user).await {
                Ok(notes) => next.push(
                    serde_json::json!({"source_start":records[start]["source_start"],
                    "source_end":records[end-1]["source_end"],"notes":notes}),
                ),
                Err(e) if size_error(&e) => {
                    group_limit /= 2;
                    continue;
                }
                Err(e) => return Err(e).context(format!("reduce level {}", level)),
            }
            start = end;
        }
        ensure!(
            serde_json::to_vec(&next)?.len() < all.len(),
            "reduction_no_progress"
        );
        records = next;
    }
    bail!("analysis_budget_exceeded: reduction depth")
}

==
chunking
==
use std::ops::Range;

/// Source ranges cover every byte exactly once. Prefer paragraph/line boundaries;
/// oversized rows are split safely without allocating the entire document again.
pub fn ranges(text: &str, limit: usize) -> Vec<Range<usize>> {
    assert!(limit >= 4);
    let mut result = Vec::new();
    let mut start = 0;
    while start < text.len() {
        let mut end = start.saturating_add(limit).min(text.len());
        while !text.is_char_boundary(end) { end -= 1; }
        if end < text.len() {
            let slice = &text[start..end];
            let boundary = slice.rfind("\n\n").map(|p| p + 2)
                .or_else(|| slice.rfind('\n').map(|p| p + 1));
            if let Some(p) = boundary { if p >= slice.len() / 2 { end = start + p; } }
        }
        result.push(start..end);
        start = end;
    }
    result
}

/// Context is bounded separately by the caller. Table columns are preserved as
/// raw Markdown: no naive splitting on pipes (which may be escaped or in code).
pub fn context_before(text: &str, start: usize, max: usize) -> String {
    let mut headings = Vec::new();
    let mut previous = "";
    let mut table = String::new();
    let mut fenced = false;
    for line in text[..start].lines() {
        let trimmed = line.trim();
        if trimmed.starts_with("```") || trimmed.starts_with("~~~") { fenced = !fenced; }
        if !fenced {
            if trimmed.starts_with('#') && trimmed.contains(' ') {
                let level = trimmed.chars().take_while(|c| *c == '#').count();
                if level <= 6 {
                    headings.retain(|(l, _): &(usize, &str)| *l < level);
                    headings.push((level, line));
                }
            }
            if trimmed.contains('|') && trimmed.contains('-')
                && trimmed.chars().all(|c| matches!(c, '|' | '-' | ':' | ' ' | '\t'))
                && previous.contains('|') {
                table = format!("{}\n{}", previous, line);
            } else if trimmed.is_empty() || !trimmed.contains('|') { table.clear(); }
        }
        previous = line;
    }
    let value = format!("Repeated context only:\n{}\n{}\nContinuation at source byte {} (a row may continue).\n",
        headings.iter().map(|(_, s)| *s).collect::<Vec<_>>().join("\n"), table, start);
    if value.len() <= max { value } else {
        // Do not truncate a table header into misleading column names.
        format!("Continuation at source byte {}; preceding header exceeds context budget.\n", start)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn lossless_large_unicode_rows() {
        let text = format!("# Tabela\n| a | b |\n|---|---|\n| {} | x\\|y |\n", "żółć🙂".repeat(1000));
        let parts = ranges(&text, 128);
        assert!(parts.iter().all(|r| r.len() <= 128));
        assert_eq!(parts.iter().map(|r| &text[r.clone()]).collect::<String>(), text);
        assert!(context_before(&text, parts[1].start, 512).contains("a | b"));
    }
    #[test]
    fn empty_and_exact() {
        assert!(ranges("", 4).is_empty());
        assert_eq!(ranges("abcd", 4), vec![0..4]);
    }
}

==
config
==
use anyhow::{bail, Context, Result};
use std::{env, str::FromStr};

#[derive(Clone, Debug)]
pub struct Config {
    pub enabled: bool,
    pub input_limit: usize,
    pub target: usize,
    pub context_limit: usize,
    pub output: u32,
    pub max_calls: usize,
    pub max_input_units: usize,
    pub max_depth: usize,
    pub request_secs: u64,
    pub total_secs: u64,
    pub max_body_bytes: usize,
}

fn setting<T: FromStr>(key: &str, default: T) -> Result<T> {
    match env::var(key) {
        Ok(value) => value.parse().map_err(|_| anyhow::anyhow!("invalid configuration: {}", key)),
        Err(env::VarError::NotPresent) => Ok(default),
        Err(e) => Err(e).with_context(|| format!("invalid configuration: {}", key)),
    }
}

impl Config {
    pub fn from_env() -> Result<Self> {
        let c = Self {
            enabled: setting("AI_DOCUMENT_ENABLED", true)?,
            input_limit: setting("AI_DOCUMENT_INPUT_LIMIT", 24_000)?,
            target: setting("AI_DOCUMENT_TARGET", 20_000)?,
            context_limit: setting("AI_DOCUMENT_CONTEXT_LIMIT", 32_768)?,
            output: setting("AI_DOCUMENT_OUTPUT_TOKENS", 2_000)?,
            max_calls: setting("AI_DOCUMENT_MAX_CALLS", 512)?,
            max_input_units: setting("AI_DOCUMENT_MAX_INPUT_UNITS", 12_000_000)?,
            max_depth: setting("AI_DOCUMENT_MAX_DEPTH", 12)?,
            request_secs: setting("AI_DOCUMENT_REQUEST_SECS", 180)?,
            total_secs: setting("AI_DOCUMENT_TOTAL_SECS", 1800)?,
            max_body_bytes: setting("AI_DOCUMENT_MAX_BODY_BYTES", 256_000)?,
        };
        if c.target < 4096 || c.target > c.input_limit || c.input_limit > 1_000_000
            || c.context_limit > 10_000_000 || c.context_limit <= c.input_limit
            || c.output == 0 || c.max_calls == 0 || c.max_depth == 0 || c.max_input_units == 0
            || c.input_limit.saturating_add(c.output as usize) > c.context_limit
            || c.max_depth > 32 || c.request_secs == 0 || c.total_secs == 0
            || c.max_body_bytes < 4096
        { bail!("invalid document analysis budgets"); }
        Ok(c)
    }
}
