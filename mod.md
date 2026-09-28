mod

pub mod budget;
pub mod chunking;
pub mod config;
mod contracts;
pub(crate) mod prompts;
mod reduction;

use crate::clients::ai_gateway::{AiGatewayClient, GatewayFailure};
use anyhow::{Context, Result, bail, ensure};
use config::Config;
use contracts::{Notes, final_meta};
use futures::{StreamExt, TryStreamExt, stream};
use serde_json::Value;
use std::{
    collections::VecDeque,
    ops::Range,
    sync::atomic::{AtomicUsize, Ordering},
    time::{Duration, Instant},
};

struct Run<'a> {
    client: &'a AiGatewayClient,
    cfg: Config,
    calls: AtomicUsize,
    input_units: AtomicUsize,
    id: String,
}
impl Run<'_> {
    fn units(&self, system: &str, user: &str) -> usize {
        budget::estimate(system, user, self.cfg.overhead, self.cfg.bytes_per_token)
    }
    fn fits(&self, system: &str, user: &str, output: u32) -> bool {
        let n = self.units(system, user);
        n <= self.cfg.input_limit && n.saturating_add(output as usize) <= self.cfg.context_limit
    }
    async fn request(&self, system: &str, user: &str, output: u32) -> Result<Value> {
        if !self.fits(system, user, output) {
            return Err(GatewayFailure {
                kind: "context_exceeded",
                retry_after: None,
            }
            .into());
        }
        for attempt in 0..3u64 {
            self.input_units
                .fetch_update(Ordering::SeqCst, Ordering::SeqCst, |n| {
                    n.checked_add(self.units(system, user))
                        .filter(|n| *n <= self.cfg.max_input_units)
                })
                .map_err(|_| anyhow::anyhow!("analysis_budget_exceeded: cumulative input"))?;
            let call = self
                .calls
                .fetch_update(Ordering::SeqCst, Ordering::SeqCst, |n| {
                    (n < self.cfg.max_calls).then_some(n + 1)
                })
                .map_err(|_| anyhow::anyhow!("analysis_budget_exceeded: max calls"))?
                + 1;
            let start = Instant::now();
            tracing::info!(analysis_id = %self.id, call, attempt,
                model = self.client.model(), input_units = self.units(system, user),
                bytes_per_token = self.cfg.bytes_per_token, counting = "estimated_tokens",
                "AI call started");
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
            tracing::info!(analysis_id = %self.id, call,
                elapsed_ms = start.elapsed().as_millis() as u64, success = result.is_ok(), "AI call finished");
            match result {
                Ok(v) => return Ok(v),
                Err(e) => {
                    tracing::warn!(analysis_id = %self.id, call, error = %e, "AI call failed");
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
    fn repair_system(&self, system: &str) -> String {
        format!(
            "{}{}",
            system,
            self.cfg
                .repair_prompt
                .replace("{keywords}", &self.cfg.keywords.to_string())
        )
    }
    async fn checked<T>(
        &self,
        system: &str,
        user: &str,
        parse: impl Fn(Value) -> Result<T>,
    ) -> Result<T> {
        let mut system = system.to_owned();
        for attempt in 0..2 {
            let result = self
                .request(&system, user, self.client.output_limit())
                .await
                .and_then(&parse);
            match result {
                Ok(n) => return Ok(n),
                Err(e) if attempt == 0 && repairable(&e) => {
                    tracing::warn!(analysis_id = %self.id, error = %e, "Repairing AI response");
                    system = self.repair_system(&system);
                }
                Err(e) => return Err(e),
            }
        }
        unreachable!()
    }
    async fn notes(&self, system: &str, user: &str) -> Result<Notes> {
        self.checked(system, user, Notes::parse).await
    }
    async fn metadata(&self, system: &str, user: &str) -> Result<(String, Vec<String>)> {
        self.checked(system, user, |v| {
            let result = final_meta(v)?;
            ensure!(
                result.1.len() == self.cfg.keywords,
                "invalid_model_json: expected {} unique keywords, got {}",
                self.cfg.keywords,
                result.1.len()
            );
            Ok(result)
        })
        .await
    }
}
fn repairable(e: &anyhow::Error) -> bool {
    // Never retry transport/auth/budget failures as schema repairs.
    e.downcast_ref::<GatewayFailure>().is_none()
        && (e.to_string().starts_with("invalid_model_json") || e.to_string() == "output_truncated")
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
    if !client.settings().ai_document_enabled {
        return client.get_ai_meta(text).await;
    }
    let cfg = Config::from_cli(client.settings())?;
    ensure!(!text.trim().is_empty(), "empty_document");
    let id = uuid::Uuid::new_v4().to_string();
    tracing::info!(file_id, analysis_id = %id, bytes = text.len(), "Document analysis started");
    let duration = Duration::from_secs(cfg.total_secs);
    let mut run = Run {
        client,
        cfg,
        calls: AtomicUsize::new(0),
        input_units: AtomicUsize::new(0),
        id: id.clone(),
    };
    let result = tokio::time::timeout(duration, process(&mut run, text)).await;
    tracing::info!(file_id, analysis_id = %id, calls = run.calls.load(Ordering::SeqCst),
        input_units = run.input_units.load(Ordering::SeqCst),
        success = matches!(&result, Ok(Ok(_))), "Document analysis finished");
    result
        .map_err(|_| {
            anyhow::anyhow!(
                "analysis_budget_exceeded: deadline analysis {} file_id {}",
                id,
                file_id
            )
        })?
        .with_context(|| format!("AI analysis {} file_id {}", id, file_id))
}

async fn map(
    run: &Run<'_>,
    text: &str,
    range: Range<usize>,
    context: &chunking::ContextIndex<'_>,
) -> Result<Vec<Value>> {
    let map_prompt = &run.cfg.map_prompt;
    let mut pending = VecDeque::from([range]);
    let mut records = Vec::new();
    while let Some(range) = pending.pop_front() {
        let context = context.context_before(range.start, 1024);
        let user = format!(
            "{}\nSource bytes {}..{}:\n{}",
            context,
            range.start,
            range.end,
            &text[range.clone()]
        );
        match run.notes(map_prompt, &user).await {
            Ok(notes) => records.push(serde_json::json!({"source_start":range.start,"source_end":range.end,"notes":notes})),
            Err(e) if (size_error(&e) || e.to_string() == "output_truncated") && range.len() >= 8 => {
                let sub = chunking::ranges(&text[range.clone()], range.len() / 2);
                tracing::warn!(analysis_id = %run.id, source_start = range.start, source_end = range.end,
                    pieces = sub.len(), "Splitting rejected fragment");
                for r in sub.into_iter().rev() { pending.push_front(range.start+r.start..range.start+r.end); }
            }
            Err(e) => return Err(e).context(format!("map source {}..{}", range.start, range.end)),
        }
    }
    Ok(records)
}

async fn process(run: &mut Run<'_>, text: &str) -> Result<(String, Vec<String>)> {
    let system = run.client.system_prompt().to_owned();
    let map_prompt = run.cfg.map_prompt.clone();
    let reduce_prompt = run.cfg.reduce_prompt.clone();
    // Count without allocating a second full-document String.
    if run
        .units(&system, text)
        .saturating_add("Document (Markdown):\n\n".len())
        <= run.cfg.input_limit
    {
        let user = format!("Document (Markdown):\n\n{}", text);
        if run.fits(&system, &user, run.client.output_limit()) {
            match run.metadata(&system, &user).await {
                Ok(v) => return Ok(v),
                Err(e) if size_error(&e) || e.to_string() == "output_truncated" => {}
                Err(e) => return Err(e),
            }
        }
    }
    // Keep room for source coordinates, table context and a repair prompt.
    let reserve = run.units(&run.repair_system(&map_prompt), "") + 1200;
    ensure!(
        run.cfg.target > reserve + 256,
        "invalid_configuration: map budget"
    );
    let limit = (run.cfg.target - reserve).saturating_mul(run.cfg.bytes_per_token);
    let pending = chunking::ranges(text, limit);
    let context = chunking::ContextIndex::new(text);
    tracing::info!(analysis_id = %run.id, chunks = pending.len(),
        concurrency = run.cfg.concurrency, "Document map planned");
    let records = stream::iter(pending)
        .map(|range| map(run, text, range, &context))
        .buffered(run.cfg.concurrency)
        .try_collect::<Vec<_>>()
        .await?;
    // buffered preserves source order even when requests complete out of order.
    let mut records: Vec<Value> = records.into_iter().flatten().collect();
    let final_system = format!("{}{}", system, run.cfg.final_prompt);
    let mut group_limit = run.cfg.target;
    for level in 0..=run.cfg.max_depth {
        let all = serde_json::to_string(&records)?;
        if run.fits(&final_system, &all, run.client.output_limit()) {
            match run.metadata(&final_system, &all).await {
                Ok(v) => return Ok(v),
                Err(e) if size_error(&e) || e.to_string() == "output_truncated" => {
                    group_limit /= 2;
                }
                Err(e) => return Err(e).context("final synthesis"),
            }
        }
        // max_depth limits reductions, not the final synthesis after them.
        ensure!(
            level < run.cfg.max_depth,
            "analysis_budget_exceeded: reduction depth"
        );
        let mut next = Vec::new();
        let mut start = 0;
        while start < records.len() {
            let reserve = run.units(&run.repair_system(&reduce_prompt), "");
            ensure!(group_limit > reserve + 256, "reduction_budget_too_small");
            let max_bytes = (group_limit - reserve)
                .saturating_mul(run.cfg.bytes_per_token)
                .saturating_sub(2);
            let parts = reduction::split_record(records[start].clone(), max_bytes)?;
            records.splice(start..start + 1, parts);
            let mut end = start + 1;
            while end < records.len() {
                let candidate = serde_json::to_string(&records[start..end + 1])?;
                if run.units(&run.repair_system(&reduce_prompt), &candidate) > group_limit {
                    break;
                }
                end += 1;
            }
            let user = serde_json::to_string(&records[start..end])?;
            let result = async {
                let mut notes = run.notes(&reduce_prompt, &user).await?;
                let make_record = |notes: Notes| {
                    serde_json::json!({
                    "source_start":records[start]["source_start"],
                    "source_end":records[end-1]["source_end"], "notes":notes})
                };
                let mut record = make_record(notes);
                if serde_json::to_vec(&record)?.len() >= user.len().saturating_sub(2) {
                    notes = run.notes(&run.repair_system(&reduce_prompt), &user).await?;
                    record = make_record(notes);
                }
                Ok::<Value, anyhow::Error>(record)
            }
            .await;
            match result {
                Ok(notes) => next.push(notes),
                Err(e) if size_error(&e) || e.to_string() == "output_truncated" => {
                    group_limit /= 2;
                    continue;
                }
                Err(e) => return Err(e).context(format!("reduce level {}", level)),
            }
            start = end;
        }
        // A final singleton may grow slightly; progress is measured for the whole
        // level after individual compression retries, not for every group.
        ensure!(
            serde_json::to_vec(&next)?.len() < all.len(),
            "reduction_no_progress: compression retry exhausted"
        );
        tracing::info!(analysis_id = %run.id, level, records = next.len(), "Reduction level completed");
        records = next;
    }
    bail!("analysis_budget_exceeded: reduction depth")
}

==

reduction
use anyhow::{Result, ensure};
use serde_json::{Value, json};
use std::collections::VecDeque;

/// Split only oversized notes. Each field and every byte survive, in order;
/// source intervals may repeat because several pieces describe the same source.
pub(super) fn split_record(record: Value, max_bytes: usize) -> Result<Vec<Value>> {
    if serde_json::to_vec(&record)?.len() <= max_bytes {
        return Ok(vec![record]);
    }
    let mut pending = VecDeque::new();
    if let Some(s) = record["notes"]["summary"].as_str() {
        if !s.is_empty() {
            pending.push_back(("summary", s.to_owned()));
        }
    }
    for field in ["facts", "keywords"] {
        if let Some(values) = record["notes"][field].as_array() {
            for value in values {
                if let Some(s) = value.as_str() {
                    pending.push_back((field, s.to_owned()));
                }
            }
        }
    }
    ensure!(!pending.is_empty(), "invalid_intermediate_record");
    let mut result = Vec::new();
    while let Some((field, text)) = pending.pop_front() {
        let mut item = json!({"source_start":record["source_start"],
            "source_end":record["source_end"],
            "notes":{"summary":"", "facts":[], "keywords":[]}});
        item["notes"][field] = if field == "summary" {
            json!(text)
        } else {
            json!([text])
        };
        if serde_json::to_vec(&item)?.len() <= max_bytes {
            result.push(item);
        } else {
            let mut end = text.len() / 2;
            while end > 0 && !text.is_char_boundary(end) {
                end -= 1;
            }
            ensure!(end > 0, "reduction_budget_too_small: record envelope");
            pending.push_front((field, text[end..].to_owned()));
            pending.push_front((field, text[..end].to_owned()));
        }
    }
    Ok(result)
}
==
chun
use std::ops::Range;

/// Every source byte occurs exactly once; prefer paragraph/line boundaries.
pub fn ranges(text: &str, limit: usize) -> Vec<Range<usize>> {
    assert!(limit >= 4);
    let mut result = Vec::new();
    let mut start = 0;
    while start < text.len() {
        let mut end = start.saturating_add(limit).min(text.len());
        while !text.is_char_boundary(end) {
            end -= 1;
        }
        if end < text.len() {
            let slice = &text[start..end];
            let boundary = slice
                .rfind("\n\n")
                .map(|p| p + 2)
                .filter(|p| *p >= slice.len() / 2)
                .or_else(|| slice.rfind('\n').map(|p| p + 1));
            if let Some(p) = boundary.filter(|p| *p >= slice.len() / 2) {
                end = start + p;
            }
        }
        result.push(start..end);
        start = end;
    }
    result
}

/// Precompute state changes once. Cache columns of long rows for adaptive cuts.
pub struct ContextIndex<'a> {
    text: &'a str,
    rows: Vec<usize>,
    states: Vec<ContextState<'a>>,
    cells: std::collections::HashMap<usize, Vec<Range<usize>>>,
}
struct ContextState<'a> {
    offset: usize,
    headings: Vec<(usize, &'a str)>,
    table: &'a str,
    fenced: bool,
}
impl<'a> ContextIndex<'a> {
    pub fn new(text: &'a str) -> Self {
        let mut rows = vec![0];
        let mut states = vec![ContextState {
            offset: 0,
            headings: Vec::new(),
            table: "",
            fenced: false,
        }];
        let mut cells = std::collections::HashMap::new();
        let mut headings = Vec::new();
        let mut previous = "";
        let mut table = "";
        let mut fenced: Option<(char, usize)> = None;
        let mut offset = 0;
        for raw in text.split_inclusive('\n') {
            let line = raw.trim_end_matches('\n').trim_end_matches('\r');
            if line.len() > 1024 {
                cells.insert(offset, columns(line));
            }
            (|| {
                let trimmed = line.trim();
                if let Some((marker, width)) = fenced {
                    if trimmed.chars().take_while(|c| *c == marker).count() >= width
                        && trimmed.trim_matches(marker).trim().is_empty()
                    {
                        fenced = None;
                    }
                    previous = "";
                    return;
                }
                let marker = trimmed.chars().next().unwrap_or(' ');
                let width = trimmed.chars().take_while(|c| *c == marker).count();
                if matches!(marker, '`' | '~') && width >= 3 {
                    fenced = Some((marker, width));
                    table = "";
                    previous = "";
                    return;
                }
                if trimmed.starts_with('#') && trimmed.contains(' ') {
                    let level = trimmed.chars().take_while(|c| *c == '#').count();
                    if level <= 6 {
                        headings.retain(|(l, _): &(usize, &str)| *l < level);
                        headings.push((level, line));
                    }
                }
                let cells = columns(line);
                if cells.len() > 1
                    && cells.iter().all(|r| {
                        let cell = line[r.clone()].trim().trim_matches(':');
                        cell.len() >= 3 && cell.chars().all(|c| c == '-')
                    })
                    && columns(previous).len() == cells.len()
                {
                    table = previous;
                } else if trimmed.is_empty() || columns(line).len() <= 1 {
                    table = "";
                }
                previous = line;
            })();
            offset += raw.len();
            rows.push(offset);
            let last = states.last().unwrap();
            if last.headings != headings || last.table != table || last.fenced != fenced.is_some() {
                states.push(ContextState {
                    offset,
                    headings: headings.clone(),
                    table,
                    fenced: fenced.is_some(),
                });
            }
        }
        Self {
            text,
            rows,
            states,
            cells,
        }
    }

    pub fn context_before(&self, start: usize, max: usize) -> String {
        let text = self.text;
        let row_index = self.rows.partition_point(|p| *p <= start).saturating_sub(1);
        let row_start = self.rows[row_index];
        let row_end = self.rows.get(row_index + 1).copied().unwrap_or(text.len());
        let row = text[row_start..row_end]
            .trim_end_matches('\n')
            .trim_end_matches('\r');
        let state = &self.states[self
            .states
            .partition_point(|s| s.offset <= row_start)
            .saturating_sub(1)];
        let headings = &state.headings;
        let table = state.table;
        let fenced = state.fenced;
        let temporary;
        let cells = if let Some(cells) = self.cells.get(&row_start) {
            cells.as_slice()
        } else {
            temporary = columns(row);
            temporary.as_slice()
        };
        let column = cells
            .iter()
            .position(|r| r.end >= start - row_start)
            .unwrap_or(0);
        let mut value =
            format!("Repeated context only. Source byte {start}; row starts at byte {row_start}.");
        if !table.is_empty() && cells.len() > 1 && !fenced {
            value.push_str(&format!(
                " Table continuation; starting column {}.\n",
                column + 1
            ));
            let headers = columns(table);
            // Retain original column numbers. Prioritize the column containing the cut.
            for i in std::iter::once(column).chain((0..headers.len()).filter(|i| *i != column)) {
                if let Some(r) = headers.get(i) {
                    let item = format!("Column {}: {}\n", i + 1, table[r.clone()].trim());
                    if value.len() + item.len() <= max {
                        value.push_str(&item);
                    } else if i == column {
                        let item = format!(
                            "Column {} label too long; see original source header.\n",
                            i + 1
                        );
                        if value.len() + item.len() <= max {
                            value.push_str(&item);
                        }
                    }
                }
            }
        }
        for (_, heading) in headings.iter().rev() {
            if value.len() + heading.len() + 1 <= max {
                value.push('\n');
                value.push_str(heading);
            }
        }
        if value.len() > max {
            let mut end = max.min(value.len());
            while !value.is_char_boundary(end) {
                end -= 1;
            }
            value.truncate(end);
        }
        value
    }
}

/// Compatibility helper; repeated queries should reuse ContextIndex.
pub fn context_before(text: &str, start: usize, max: usize) -> String {
    ContextIndex::new(text).context_before(start, max)
}

/// Pipes in escaped sequences and inline code are not column separators.
fn columns(line: &str) -> Vec<Range<usize>> {
    let bytes = line.as_bytes();
    let mut boundaries = Vec::new();
    let mut code = 0;
    let mut i = 0;
    while i < bytes.len() {
        if bytes[i] == b'\\' {
            i += 2;
            continue;
        }
        if bytes[i] == b'`' {
            let start = i;
            while i < bytes.len() && bytes[i] == b'`' {
                i += 1;
            }
            let width = i - start;
            if code == 0 {
                code = width;
            } else if code == width {
                code = 0;
            }
            continue;
        }
        if bytes[i] == b'|' && code == 0 {
            boundaries.push(i);
        }
        i += 1;
    }
    let mut result = Vec::new();
    let mut start = 0;
    for end in boundaries {
        if end != 0 && !(start == 0 && line[..end].trim().is_empty()) {
            result.push(start..end);
        }
        start = end + 1;
    }
    if start < line.len() && !line[start..].trim().is_empty() {
        result.push(start..line.len());
    }
    result
}
