#[arg(long, env = "AI_DOCUMENT_ENABLED", default_value_t = true, action = clap::ArgAction::Set)]
    pub ai_document_enabled: bool,
    #[arg(long, env = "AI_DOCUMENT_INPUT_LIMIT", default_value_t = 24_000)]
    pub ai_document_input_limit: usize,
    #[arg(long, env = "AI_DOCUMENT_TARGET", default_value_t = 20_000)]
    pub ai_document_target: usize,
    #[arg(long, env = "AI_DOCUMENT_CONTEXT_LIMIT", default_value_t = 32_768)]
    pub ai_document_context_limit: usize,
    #[arg(long, env = "AI_DOCUMENT_MAX_CALLS", default_value_t = 512)]
    pub ai_document_max_calls: usize,
    #[arg(long, env = "AI_DOCUMENT_MAX_INPUT_UNITS", default_value_t = 12_000_000)]
    pub ai_document_max_input_units: usize,
    #[arg(long, env = "AI_DOCUMENT_MAX_DEPTH", default_value_t = 12)]
    pub ai_document_max_depth: usize,
    #[arg(long, env = "AI_DOCUMENT_REQUEST_SECS", default_value_t = 180)]
    pub ai_document_request_secs: u64,
    #[arg(long, env = "AI_DOCUMENT_TOTAL_SECS", default_value_t = 1800)]
    pub ai_document_total_secs: u64,
    #[arg(long, env = "AI_DOCUMENT_MAX_BODY_BYTES", default_value_t = 256_000)]
    pub ai_document_max_body_bytes: usize,
    #[arg(long, env = "AI_DOCUMENT_OVERHEAD", default_value_t = 512)]
    pub ai_document_overhead: usize,
    #[arg(long, env = "AI_DOCUMENT_MAP_PROMPT", default_value = crate::document_analysis::prompts::MAP)]
    pub ai_document_map_prompt: String,
    #[arg(long, env = "AI_DOCUMENT_REDUCE_PROMPT", default_value = crate::document_analysis::prompts::REDUCE)]
    pub ai_document_reduce_prompt: String,
    #[arg(long, env = "AI_DOCUMENT_FINAL_PROMPT", default_value = crate::document_analysis::prompts::FINAL)]
    pub ai_document_final_prompt: String,
