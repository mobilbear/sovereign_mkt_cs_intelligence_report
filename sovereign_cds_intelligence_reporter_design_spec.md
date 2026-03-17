# Sovereign CDS Intelligence Reporter
## Comprehensive Merged Project Specification

Version: 1.0  
Status: Implementation-ready merged spec  
Target build mode: Phase 1 MVP on a Bloomberg-enabled company laptop

---

## 1. Project Objective

Build a local, analyst-supervised sovereign CDS intelligence and reporting system that:

1. Pulls sovereign CDS and related market context data from Bloomberg using the Bloomberg Desktop API in Python via `blpapi`.
2. Stores raw and normalized data locally for historical analysis, reproducibility, and auditability.
3. Detects meaningful sovereign spread moves using transparent quantitative logic.
4. Retrieves relevant recent supporting evidence for either:
   - an analyst-selected theme, or
   - top movers detected by the signal engine.
5. Builds a frozen evidence pack.
6. Uses an LLM to generate a structured internal sovereign risk note from the evidence pack only.
7. Validates the generated output against the evidence pack.
8. Renders the report primarily to DOCX, with optional PDF / HTML outputs.
9. Archives raw inputs, curated data, signals, evidence packs, prompts, outputs, and analyst edits.

This system is **not** “let AI write a report from scratch.” It must follow the principle:

> **Data as the skeleton, AI as the narrative layer.**  
> Quantitative logic identifies what moved and why it may matter; the LLM explains only from structured, validated evidence.

---

## 2. Primary Use Case

An analyst wants to produce a daily or event-driven sovereign CDS note.

Example workflow:

- Analyst selects a theme such as:
  - “US and Israeli domestic politics may influence conflict duration”
  - “Oil shock spillover into sovereign risk”
  - “Election uncertainty repricing in EM sovereigns”
- System pulls latest sovereign CDS data and market context from Bloomberg.
- System computes daily / weekly / monthly changes, abnormality metrics, percentiles, and regional breadth.
- System retrieves recent supporting evidence from approved sources.
- System builds a frozen evidence pack.
- LLM drafts a concise sovereign risk note.
- Analyst reviews, edits if needed, and approves final output.

---

## 3. Non-Negotiable Principles

1. **The LLM must never compute primary market facts.** All numerical facts must come from deterministic code.
2. **Every run must be reproducible.** A report must be reconstructible from archived inputs.
3. **The Bloomberg layer must be real `blpapi` code.** No fake provider placeholder for live mode.
4. **No hardcoded Bloomberg security IDs or field names in source code.** All identifiers must come from config files.
5. **Use a modular pipeline.** Bloomberg ingestion, normalization, signal logic, evidence retrieval, LLM generation, rendering, and archiving must remain separated.
6. **No unsupported causal storytelling.** The report must distinguish observed facts from plausible interpretation.
7. **Human review remains mandatory in MVP.** Validation warnings should flag drafts for analyst review rather than silently passing.
8. **Support offline development.** A `DRY_RUN` mode must allow the full pipeline to run on fixtures without a live Bloomberg session.

---

## 4. Scope

### 4.1 In Scope for MVP

- Bloomberg sovereign CDS data pull via `blpapi`
- Bloomberg market context pull via `blpapi`
- Fixed sovereign monitoring universe
- Local historical data storage
- Change / z-score / percentile / breadth logic
- Stale quote filter
- Theme-led evidence retrieval
- Mover-led evidence retrieval support at the interface level
- Evidence pack generation
- LLM-generated report draft
- Output validation
- DOCX output
- Optional PDF / HTML renderers
- Full archival
- DRY_RUN fixture mode

### 4.2 Out of Scope for MVP

- Enterprise shared deployment
- Fully autonomous circulation without analyst approval
- Intraday streaming
- Direct enterprise system integration
- Portfolio P&L / VaR integration
- Automatic Bloomberg News workflow unless entitlement is explicitly confirmed

---

## 5. Target Users

- Sovereign risk analyst
- Credit risk analyst
- Portfolio risk analyst
- Market risk manager
- Senior risk management / CRO support teams

---

## 6. High-Level Architecture

```text
Analyst Theme Input
        +
Bloomberg Market Data (via blpapi)
        +
Approved News / Public Sources / Manual Evidence
        |
        v
[1] Bloomberg Ingestion Layer
        |
        v
[2] Normalization & Data Quality Layer
        |
        v
[3] Signal Logic Engine
        |
        v
[4] Intelligence / Evidence Engine
        |
        v
[5] Evidence Pack Builder
        |
        v
[6] LLM Narrative Engine
        |
        v
[7] Output Validator
        |
        v
[8] Report Renderer
        |
        v
[9] Review + Archive Layer
```

---

## 7. Technology Choices

### 7.1 Required Runtime

- Python 3.11+

### 7.2 Storage

- **DuckDB** for MVP local database
- Parquet for raw archival and selected curated / reporting layers where helpful

Reason: DuckDB is better suited than SQLite for analytical queries and native Parquet interaction.

### 7.3 Bloomberg API

- Bloomberg Desktop API via `blpapi`

### 7.4 LLM Provider

- Anthropic Claude via the `anthropic` Python SDK
- Model name configurable via config
- API key provided via `.env`

### 7.5 News Retrieval

- Pluggable provider design
- Preferred MVP providers:
  - SerpAPI, or
  - Exa API
- If no API key is present, fall back to manual evidence items provided by the analyst in theme input JSON

### 7.6 Fuzzy Matching

For country / entity matching between evidence and sovereign universe, use:

```python
from fuzzywuzzy import fuzz
```

---

## 8. Bloomberg API Implementation Specification (Mandatory)

### 8.1 Bloomberg Access Method

The project must use the **Bloomberg Desktop API in Python via `blpapi`**, running locally on a Bloomberg-enabled laptop.

The Bloomberg data layer must follow the native Bloomberg API request pattern:

1. Create `blpapi.Session()`
2. Start session with `session.start()`
3. Open service `//blp/refdata`
4. Get service with `session.getService('//blp/refdata')`
5. Create request using:
   - `HistoricalDataRequest`
   - `ReferenceDataRequest`
6. Append securities and fields
7. Add overrides if needed
8. Send request with `session.sendRequest(request)`
9. Read events using `session.nextEvent()`
10. Handle both:
   - `blpapi.Event.PARTIAL_RESPONSE`
   - `blpapi.Event.RESPONSE`
11. Parse:
   - `securityData`
   - `fieldData`
   - `fieldExceptions`
   - `responseError`
   - `securityError`
12. Convert results to structured records / DataFrames
13. Persist raw and normalized outputs locally

### 8.2 Existing User Code as Reference

Use the user’s attached local Bloomberg script as a **style and structure reference** for the Bloomberg calling pattern, especially:

- `blpapi.Session()`
- `session.openService('//blp/refdata')`
- `service.createRequest("HistoricalDataRequest")`
- `service.createRequest("ReferenceDataRequest")`
- appending securities dynamically
- appending fields dynamically
- building overrides
- setting request options such as:
  - `nonTradingDayFillOption`
  - `calendarCodeOverride`
  - `nonTradingDayFillMethod`
  - `periodicityAdjustment`
  - `periodicitySelection`
- event loop over `PARTIAL_RESPONSE` and `RESPONSE`
- extracting `securityData` and `fieldData`
- converting rows to `pandas.DataFrame`

Do **not** copy that file as-is. The production project must refactor those ideas into clean modules.

### 8.3 Bloomberg Request Types Required

#### HistoricalDataRequest
Use for:
- sovereign CDS history
- market context history
- initial backfill
- daily incremental update

Must support:
- `startDate`
- `endDate`
- `periodicitySelection`
- `periodicityAdjustment`
- `nonTradingDayFillOption`
- `nonTradingDayFillMethod`
- optional `calendarCodeOverride`
- optional overrides

#### ReferenceDataRequest
Use for:
- snapshot fields
- metadata fields
- reference-style same-day pulls where needed

### 8.4 Bloomberg Module Structure

Implement:

```text
src/providers/
  bloomberg_session_manager.py
  bloomberg_request_builder.py
  bloomberg_response_parser.py
  bloomberg_blpapi_provider.py
```

#### `bloomberg_session_manager.py`
Responsibilities:
- start session
- stop session
- open service
- manage connection errors cleanly

#### `bloomberg_request_builder.py`
Responsibilities:
- build `HistoricalDataRequest`
- build `ReferenceDataRequest`
- append securities
- append fields
- append overrides
- apply request defaults from config

#### `bloomberg_response_parser.py`
Responsibilities:
- parse `securityData`
- parse `fieldData`
- parse `fieldExceptions`
- parse `securityError`
- parse `responseError`
- return standardized records

#### `bloomberg_blpapi_provider.py`
Responsibilities:
- expose clean methods:
  - `validate_connection()`
  - `get_history(...)`
  - `get_snapshot(...)`
  - `get_context_series(...)`
- orchestrate session / request / send / event loop / parse / retry / logging

### 8.5 Bloomberg Coding Rules

#### Session lifecycle
- Centralize in `bloomberg_session_manager.py`
- No other module creates or destroys a session
- Stop session cleanly

#### Request lifecycle
- Build a fresh request object for each logical request
- Do not reuse mutated request objects across unrelated pulls

#### Event loop
Must explicitly handle:
- `PARTIAL_RESPONSE`
- `RESPONSE`

#### Error parsing
Must detect and log:
- top-level `responseError`
- per-security `securityError`
- `fieldExceptions`

#### Output normalization
Every parsed Bloomberg record must include:
- `run_id`
- `request_timestamp`
- `request_type`
- `service_name`
- `security`
- `field`
- `date` if applicable
- `value`
- `status`
- `error_code`
- `error_message`
- `provider_name = 'blpapi'`

### 8.6 Bloomberg Request Defaults

Use config for defaults, e.g.:

```yaml
bloomberg_request_defaults:
  historical:
    periodicitySelection: DAILY
    periodicityAdjustment: ACTUAL
    nonTradingDayFillOption: ALL_CALENDAR_DAYS
    nonTradingDayFillMethod: PREVIOUS_VALUE
    calendarCodeOverride: 5D
```

### 8.7 Bloomberg Overrides

Support Bloomberg overrides generically:

```python
overrides_el = request.getElement("overrides")
for ov in overrides:
    ov_el = overrides_el.appendElement()
    ov_el.setElement("fieldId", ov["fieldId"])
    ov_el.setElement("value", ov["value"])
```

### 8.8 Bloomberg IDs and Fields

Do **not** hardcode sovereign CDS Bloomberg security IDs or field names in source logic.

They must come from config and be confirmed on the target Bloomberg machine.

### 8.9 DRY_RUN Requirement

When `dry_run: true` is set in `app.yml`:
- the Bloomberg provider must not call `blpapi`
- instead, it must load fixture data from `tests/fixtures/`
- all downstream stages must still execute normally

This is mandatory for offline development and testing.

---

## 9. Required Modules

### 9.1 `config_manager`
Responsibilities:
- load YAML / JSON config
- validate required sections
- provide config to all modules

### 9.2 `market_data_provider`
Responsibilities:
- wrap Bloomberg `blpapi` calls
- fetch raw market data
- return normalized raw records

### 9.3 `market_normalizer`
Responsibilities:
- map Bloomberg identifiers to sovereign master
- standardize dates / names / metadata
- convert raw records into curated tables

### 9.4 `data_quality_engine`
Responsibilities:
- stale quote detection
- missing value handling
- duplicate handling
- invalid value checks
- data completeness scoring

### 9.5 `signal_engine`
Responsibilities:
- compute changes, z-scores, percentiles, ranks, breadth, cluster logic, theme relevance

### 9.6 `news_provider`
Responsibilities:
- retrieve recent approved evidence
- theme-led and mover-led retrieval
- deduplicate and summarize evidence

### 9.7 `theme_engine`
Responsibilities:
- handle analyst-selected theme input
- optional future theme suggestions

### 9.8 `evidence_builder`
Responsibilities:
- build frozen evidence pack
- include market facts, context, evidence, and style constraints

### 9.9 `llm_narrative_engine`
Responsibilities:
- build prompt from evidence pack
- call Anthropic Claude
- parse and validate output

### 9.10 `output_validator`
Responsibilities:
- check numeric claims against evidence pack
- check countries mentioned against evidence pack
- check structure and word count
- emit warnings

### 9.11 `report_renderer`
Responsibilities:
- DOCX primary render
- optional PDF / HTML
- tables and charts

### 9.12 `audit_archive`
Responsibilities:
- archive raw, curated, signals, evidence, prompt, output, edits

### 9.13 `scheduler`
Responsibilities:
- daily run
- manual run
- backfill / rerun

---

## 10. Directory Structure

```text
sovereign_cds_intelligence_reporter/
  README.md
  pyproject.toml
  requirements.txt
  .env.example

  config/
    app.yml
    universe.yml
    bloomberg_fields.yml
    bloomberg_request_defaults.yml
    context_series.yml
    theme_rules.yml
    report_template.yml
    llm_policy.yml
    storage.yml

  src/
    app.py
    cli.py

    core/
      models.py
      enums.py
      exceptions.py
      logging_utils.py
      date_utils.py
      config_manager.py
      validators.py

    providers/
      bloomberg_session_manager.py
      bloomberg_request_builder.py
      bloomberg_response_parser.py
      bloomberg_blpapi_provider.py
      base_news_provider.py
      public_news_provider.py

    engines/
      market_normalizer.py
      data_quality_engine.py
      signal_engine.py
      theme_engine.py
      evidence_builder.py
      llm_narrative_engine.py
      output_validator.py
      report_renderer.py

    pipeline/
      ingest_market.py
      normalize_market.py
      quality_checks.py
      compute_signals.py
      build_theme_pack.py
      retrieve_evidence.py
      build_evidence_pack.py
      generate_narrative.py
      validate_output.py
      render_report.py
      archive_run.py

    llm/
      prompt_builder.py
      output_parser.py
      factuality_checker.py

    render/
      docx_renderer.py
      pdf_renderer.py
      html_renderer.py
      chart_builder.py
      templates/

    storage/
      db.py
      repositories.py
      parquet_store.py

  data/
    raw/
    curated/
    reporting/
    archive/

  outputs/
    notes/
    charts/
    temp/

  tests/
    unit/
    integration/
    fixtures/
```

---

## 11. Configuration Files

### 11.1 `app.yml`

```yaml
app:
  env: local
  dry_run: true
  log_level: INFO
  timezone: America/Los_Angeles
```

### 11.2 `universe.yml`

```yaml
countries:
  - country_name: Israel
    iso3: ISR
    region: Middle East
    subregion: Middle East
    dm_em: DM
    priority_tier: 1
    bloomberg_security_id: "<CONFIRM_ON_TERMINAL>"
    tenor: "5Y"
    rating_bucket: "A"
    oil_exporter: false
    oil_importer: true
    gcc_flag: false
    frontier_flag: false
    bank_watchlist: true

  - country_name: Egypt
    iso3: EGY
    region: EMEA
    subregion: Middle East / Africa
    dm_em: EM
    priority_tier: 1
    bloomberg_security_id: "<CONFIRM_ON_TERMINAL>"
    tenor: "5Y"
    rating_bucket: "B/BB"
    oil_exporter: false
    oil_importer: true
    gcc_flag: false
    frontier_flag: false
    bank_watchlist: true
```

### 11.3 `bloomberg_fields.yml`

```yaml
sovereign_core:
  spread_snapshot_field:
    provider_field: "<CONFIRM_ON_TERMINAL>"
    retrieval_type: reference_or_snapshot

  spread_history_field:
    provider_field: "<CONFIRM_ON_TERMINAL>"
    retrieval_type: historical

derived_fields:
  chg_1d_bps:
    formula: "spread_bps[t] - spread_bps[t-1]"
  chg_5d_bps:
    formula: "spread_bps[t] - spread_bps[t-5]"
  chg_1m_bps:
    formula: "spread_bps[t] - spread_bps[t-21]"
  avg_30d:
    formula: "rolling_mean(spread_bps, 30)"
  std_30d:
    formula: "rolling_std(spread_bps, 30)"
  zscore_30d:
    formula: "(spread_bps[t] - avg_30d[t]) / std_30d[t]"
```

### 11.4 `bloomberg_request_defaults.yml`

```yaml
bloomberg_request_defaults:
  historical:
    periodicitySelection: DAILY
    periodicityAdjustment: ACTUAL
    nonTradingDayFillOption: ALL_CALENDAR_DAYS
    nonTradingDayFillMethod: PREVIOUS_VALUE
    calendarCodeOverride: 5D
```

### 11.5 `context_series.yml`

These are acceptable starting security IDs for config generation, but field usage still needs confirmation on the Bloomberg terminal:

```yaml
context_series:
  - series_name: Brent
    bloomberg_security_id: "CO1 Comdty"
    field: "<CONFIRM_ON_TERMINAL>"
  - series_name: UST10Y
    bloomberg_security_id: "USGG10YR Index"
    field: "<CONFIRM_ON_TERMINAL>"
  - series_name: DXY
    bloomberg_security_id: "DXY Curncy"
    field: "<CONFIRM_ON_TERMINAL>"
  - series_name: Gold
    bloomberg_security_id: "XAU Curncy"
    field: "<CONFIRM_ON_TERMINAL>"
  - series_name: VIX
    bloomberg_security_id: "VIX Index"
    field: "<CONFIRM_ON_TERMINAL>"
```

### 11.6 `theme_rules.yml`

```yaml
themes:
  geopolitical_conflict:
    expected_channels:
      - oil
      - risk_off
      - safe_haven_rates
    likely_regions:
      - Middle East
      - EMEA
    likely_flags:
      - oil_importer
      - gcc_flag

  domestic_election:
    expected_channels:
      - policy_uncertainty
      - fiscal_concern
      - risk_premium

  sanctions:
    expected_channels:
      - commodity
      - fx_pressure
      - external_funding
```

### 11.7 `report_template.yml`

```yaml
report:
  type: daily_theme_note
  sections:
    - Executive Summary
    - Theme and Market Reaction
    - Regional Sovereign CDS Developments
    - Transmission Channels
    - Implications for Bank Watchlist
    - Appendix Summary
  max_words: 900
  tone: formal_internal
```

### 11.8 `llm_policy.yml`

```yaml
llm_policy:
  provider: anthropic
  model_name: claude-sonnet-4-5
  forbid_unsupported_numbers: true
  forbid_unsupported_headlines: true
  require_observation_vs_interpretation_split: true
  allow_technical_explanation_when_no_news: true
  require_driver_relevance_labels: true
  relevance_labels:
    - High
    - Medium
    - Low
```

### 11.9 `storage.yml`

```yaml
storage:
  database: duckdb
  database_path: data/sovereign_cds_reporter.duckdb
  raw_path: data/raw/
  curated_path: data/curated/
  reporting_path: data/reporting/
  archive_path: data/archive/
```

---

## 12. Data Model

Use DuckDB for MVP. Support Parquet archival.

### 12.1 `dim_country_master`

```sql
CREATE TABLE dim_country_master (
    country_id INTEGER PRIMARY KEY,
    country_name TEXT NOT NULL,
    iso3 TEXT NOT NULL,
    region TEXT,
    subregion TEXT,
    dm_em TEXT,
    rating_bucket TEXT,
    oil_exporter_flag BOOLEAN,
    oil_importer_flag BOOLEAN,
    gcc_flag BOOLEAN,
    frontier_flag BOOLEAN,
    bank_watchlist_flag BOOLEAN,
    priority_tier INTEGER,
    active_flag BOOLEAN DEFAULT 1,
    effective_from TEXT,
    effective_to TEXT
);
```

### 12.2 `dim_security_map`

```sql
CREATE TABLE dim_security_map (
    security_map_id INTEGER PRIMARY KEY,
    country_id INTEGER NOT NULL,
    provider_name TEXT NOT NULL,
    security_type TEXT,
    provider_security_id TEXT NOT NULL,
    tenor TEXT,
    field_map_version TEXT,
    active_flag BOOLEAN DEFAULT 1,
    effective_from TEXT,
    effective_to TEXT,
    FOREIGN KEY(country_id) REFERENCES dim_country_master(country_id)
);
```

### 12.3 `raw_bloomberg_records`

```sql
CREATE TABLE raw_bloomberg_records (
    run_id TEXT,
    request_timestamp TEXT,
    request_type TEXT,
    service_name TEXT,
    security TEXT,
    field TEXT,
    date TEXT,
    value DOUBLE,
    status TEXT,
    error_code TEXT,
    error_message TEXT,
    provider_name TEXT
);
```

### 12.4 `fact_sovereign_cds_daily`

```sql
CREATE TABLE fact_sovereign_cds_daily (
    asof_date TEXT NOT NULL,
    country_id INTEGER NOT NULL,
    tenor TEXT NOT NULL,
    spread_bps DOUBLE,
    chg_1d_bps DOUBLE,
    chg_5d_bps DOUBLE,
    chg_1m_bps DOUBLE,
    chg_3m_bps DOUBLE,
    avg_30d DOUBLE,
    std_30d DOUBLE,
    zscore_30d DOUBLE,
    low_1y DOUBLE,
    high_1y DOUBLE,
    pctile_1y DOUBLE,
    source_timestamp TEXT,
    load_timestamp TEXT,
    PRIMARY KEY(asof_date, country_id, tenor)
);
```

### 12.5 `fact_market_context_daily`

```sql
CREATE TABLE fact_market_context_daily (
    asof_date TEXT NOT NULL,
    series_name TEXT NOT NULL,
    series_value DOUBLE,
    chg_1d DOUBLE,
    chg_5d DOUBLE,
    chg_1m DOUBLE,
    source_timestamp TEXT,
    load_timestamp TEXT,
    PRIMARY KEY(asof_date, series_name)
);
```

### 12.6 `fact_sovereign_signal_daily`

```sql
CREATE TABLE fact_sovereign_signal_daily (
    asof_date TEXT NOT NULL,
    country_id INTEGER NOT NULL,
    is_stale_quote BOOLEAN,
    is_top_widener BOOLEAN,
    is_top_tightener BOOLEAN,
    widener_rank INTEGER,
    tightener_rank INTEGER,
    is_extreme_1d_move BOOLEAN,
    is_above_75pct_1y BOOLEAN,
    is_above_90pct_1y BOOLEAN,
    regional_cluster_flag BOOLEAN,
    theme_relevance_score DOUBLE,
    transmission_flags_json TEXT,
    summary_facts_json TEXT,
    PRIMARY KEY(asof_date, country_id)
);
```

### 12.7 `fact_region_summary_daily`

```sql
CREATE TABLE fact_region_summary_daily (
    asof_date TEXT NOT NULL,
    region TEXT NOT NULL,
    num_names INTEGER,
    num_wider INTEGER,
    num_tighter INTEGER,
    avg_chg_1d_bps DOUBLE,
    median_chg_1d_bps DOUBLE,
    avg_chg_1m_bps DOUBLE,
    max_widener_country_id INTEGER,
    max_widener_bps DOUBLE,
    PRIMARY KEY(asof_date, region)
);
```

### 12.8 `fact_news_event`

```sql
CREATE TABLE fact_news_event (
    event_id TEXT PRIMARY KEY,
    asof_date TEXT,
    theme_title TEXT,
    theme_category TEXT,
    source_name TEXT,
    source_url_hash TEXT,
    headline TEXT,
    published_timestamp TEXT,
    summary_text TEXT,
    countries_impacted_json TEXT,
    regions_impacted_json TEXT,
    transmission_tags_json TEXT,
    confidence_score DOUBLE,
    approved_flag BOOLEAN,
    ingest_timestamp TEXT
);
```

### 12.9 `report_input_bundle`

```sql
CREATE TABLE report_input_bundle (
    report_id TEXT PRIMARY KEY,
    asof_date TEXT,
    report_type TEXT,
    theme_title TEXT,
    theme_description TEXT,
    market_summary_json TEXT,
    movers_table_json TEXT,
    regional_summary_json TEXT,
    context_series_json TEXT,
    news_pack_json TEXT,
    bank_relevance_json TEXT,
    bundle_version TEXT,
    created_timestamp TEXT
);
```

### 12.10 `report_output_archive`

```sql
CREATE TABLE report_output_archive (
    report_id TEXT PRIMARY KEY,
    prompt_version TEXT,
    model_name TEXT,
    generated_text TEXT,
    validation_flags_json TEXT,
    analyst_edits_text TEXT,
    final_text TEXT,
    approval_user TEXT,
    approval_timestamp TEXT,
    delivery_status TEXT,
    archive_timestamp TEXT
);
```

---

## 13. Data Ingestion Requirements

### 13.1 Initial Backfill

On first setup:
- backfill at least 6 months
- preferably 12 months
- enough history to compute rolling stats and percentiles

### 13.2 Daily Incremental Update

On normal runs:
- fetch latest snapshot or latest history slice
- patch recent short history if needed
- append only new observations
- avoid repeated full-history pulls

### 13.3 Raw Archival

Every Bloomberg pull must be archived before normalization.

### 13.4 Provider Metadata

Store:
- run id
- provider name
- request type
- requested securities
- requested fields
- request options
- request timestamp
- parse status

---

## 14. Data Quality Rules

### 14.1 Stale Quote Detection

A sovereign quote is potentially stale if:
- spread is unchanged for `N` consecutive business days, default `N = 3`
- and no freshness metadata or corroborating movement is available

This must be configurable.

### 14.2 Missing Data

If latest spread is missing:
- flag it
- exclude from ranking unless override enabled
- include in QA summary

### 14.3 Duplicate Rows

Remove duplicates deterministically using latest source timestamp.

### 14.4 Invalid Values

If value is null, negative, or implausibly large beyond threshold:
- flag invalid
- exclude from signal engine unless manually reviewed

### 14.5 Data Completeness Score

Each run should produce:
- expected country count
- pulled count
- stale count
- missing count
- usable count

---

## 15. Signal Logic Specification

### 15.1 Metrics

For each sovereign:
- current spread
- 1D change
- 5D change
- 1M change
- optional 3M change
- rolling 30D average
- rolling 30D std
- 30D z-score
- 1Y min
- 1Y max
- 1Y percentile

### 15.2 Rankings

Compute:
- top wideners
- top tighteners
- top 1M wideners
- top 1M tighteners

### 15.3 Regional Breadth

For each region:
- number wider
- number tighter
- average change
- median change
- share wider
- broad move flag

### 15.4 Extreme Move Eligibility

A sovereign is a candidate for deeper report discussion if any are true:
- `abs(chg_1d_bps) >= bp_threshold`
- `abs(zscore_30d) >= zscore_threshold`
- `pctile_1y >= pctile_threshold`
- `bank_watchlist_flag == true`
- `theme_relevance_score >= theme_threshold`

Suggested defaults:
- `bp_threshold = 10`
- `zscore_threshold = 2.0`
- `pctile_threshold = 0.90`

These must be configurable.

### 15.5 Theme Relevance Score

Increase score if:
- country belongs to theme geography
- country has matching structural flags
- move direction aligns with expected channels
- region shows confirming cluster behavior

### 15.6 Regional Cluster Confirmation

A region is cluster-confirmed if:
- more than 50% of relevant names move in the same direction
- and average move exceeds minimum materiality threshold

---

## 16. Intelligence / News Retrieval Specification

### 16.1 Theme-Led Retrieval

Input:
- theme title
- theme category
- optional must-cover countries
- lookback window, default 48h

Output:
- 3 to 5 high-relevance evidence snippets

### 16.2 Mover-Led Retrieval

Input:
- list of movers
- context series state
- lookback window

Output:
- candidate evidence by mover or cluster

### 16.3 Source Design

News retrieval must be pluggable.

MVP:
- approved public / trusted web sources via SerpAPI or Exa if available
- manual analyst evidence entry supported

If no API keys are available, the pipeline must not fail; it must fall back to manual evidence mode.

### 16.4 Retrieval Rules

- deduplicate evidence
- short factual snippets only
- no long article dumps
- score by relevance, geography, recency, and transmission channel

### 16.5 Fuzzy Country Matching

Use `fuzzywuzzy` to map country mentions from evidence to sovereign universe names.

---

## 17. Theme Engine Specification

### 17.1 Analyst-Led Mode

The analyst may input:

```json
{
  "theme_title": "US and Israeli domestic politics may influence conflict duration",
  "theme_category": "geopolitical_conflict",
  "theme_description": "Potential interaction between domestic political incentives, conflict duration, oil risk, and sovereign spread repricing",
  "must_cover_countries": ["Israel", "Egypt", "Jordan"],
  "must_cover_regions": ["Middle East", "EMEA"],
  "lookback_hours": 48,
  "manual_evidence_items": []
}
```

### 17.2 Future Optional Mode

System-suggested themes may be added later but are not required for MVP.

---

## 18. Evidence Pack Specification

The evidence pack is the only object the LLM should see.

```json
{
  "report_meta": {
    "report_id": "2026-03-17_geopolitical_conflict_001",
    "asof_date": "2026-03-17",
    "report_type": "daily_theme_note",
    "audience": "internal_risk_management"
  },
  "theme": {
    "theme_title": "US and Israeli domestic politics may influence conflict duration",
    "theme_category": "geopolitical_conflict",
    "theme_description": "Potential interaction between domestic political incentives, conflict duration, oil risk, and sovereign spread repricing"
  },
  "market_facts": {
    "top_wideners": [],
    "top_tighteners": [],
    "watchlist_movers": [],
    "regional_summary": [],
    "anomalies": []
  },
  "context_series": {
    "Brent": {},
    "UST10Y": {},
    "DXY": {},
    "Gold": {},
    "VIX": {}
  },
  "news_evidence": [
    {
      "headline": "",
      "source": "",
      "summary": "",
      "published_timestamp": "",
      "relevance_score": 0.0
    }
  ],
  "bank_relevance": {
    "watchlist_countries": [],
    "notes": []
  },
  "style_constraints": {
    "max_words": 900,
    "tone": "formal_internal",
    "must_separate_observation_from_interpretation": true,
    "driver_relevance_labels_required": true
  }
}
```

---

## 19. LLM Narrative Specification

### 19.1 Provider and Model

- Provider: Anthropic
- SDK: `anthropic`
- Model name configurable in `llm_policy.yml`
- API key sourced from `.env`

### 19.2 Allowed Tasks

- write executive summary
- describe cross-regional sovereign CDS moves
- connect moves to plausible drivers
- label driver relevance as High / Medium / Low
- write short watchlist implications

### 19.3 Forbidden Tasks

- invent numbers
- invent headlines
- invent countries
- perform primary calculations
- state unsupported causality as fact

### 19.4 Required Report Structure

1. Executive Summary
2. Theme and Market Reaction
3. Regional Sovereign CDS Developments
4. Transmission Channels
5. Implications for Bank Watchlist
6. Appendix Summary

### 19.5 Prompt Rules

- use only the evidence pack
- separate observation from interpretation
- cautious causality language
- if evidence is weak, say so
- if no relevant evidence exists, mention technical / liquidity / sentiment explanations instead
- include driver relevance labels

---

## 20. Output Validation Specification

### 20.1 Validation Goals

Before rendering any DOCX, the factuality checker must:
- extract all numeric values from generated text
- verify each number exists in the evidence pack within tolerance
- extract all country names and verify each appears in the evidence pack
- verify required sections exist
- check word count against configured max

### 20.2 Tolerances

Suggested default tolerance:
- 1 bps for spread / change values
- 0.1% for percentage values

### 20.3 Failure Behavior

If any check fails:
- create specific warnings
- **do not hard-block rendering**
- flag for analyst review
- archive validation warnings alongside output

This is preferable to silently passing unsupported claims or completely stopping analyst workflow.

---

## 21. Report Rendering Specification

### 21.1 Output Priority

- Primary: DOCX
- Secondary: PDF / HTML

### 21.2 DOCX Structure

- title
- date
- theme subtitle
- executive summary
- regional discussion
- transmission channel section
- watchlist implications
- appendix
- charts / tables

### 21.3 Required Charts

Use Matplotlib:
- top wideners bar chart
- 1M change bar chart
- current spread vs 1Y range chart
- regional summary chart or heatmap

### 21.4 Required Tables

- top movers table
- watchlist movers table
- regional summary table

---

## 22. CLI / Execution Flow

### 22.1 Initial Backfill

```bash
python -m src.cli backfill --start 2025-01-01 --end 2026-03-17
```

### 22.2 Daily Run With Theme

```bash
python -m src.cli run-daily --date 2026-03-17 --theme-file ./theme_input.json
```

### 22.3 Daily Run Without Theme

```bash
python -m src.cli run-daily --date 2026-03-17
```

### 22.4 Rerender Archived Report

```bash
python -m src.cli rerender-report --report-id 2026-03-17_geopolitical_conflict_001
```

### 22.5 Validate Output Only

```bash
python -m src.cli validate-report --report-id 2026-03-17_geopolitical_conflict_001
```

---

## 23. Core Python Interfaces

```python
from abc import ABC, abstractmethod
from typing import List, Dict, Any, Optional

class MarketDataProvider(ABC):
    @abstractmethod
    def validate_connection(self) -> Dict[str, Any]:
        pass

    @abstractmethod
    def get_history(
        self,
        securities: List[str],
        fields: List[str],
        start_date: str,
        end_date: str,
        overrides: Optional[List[Dict[str, str]]] = None,
        request_options: Optional[Dict[str, Any]] = None,
    ) -> List[Dict[str, Any]]:
        pass

    @abstractmethod
    def get_snapshot(
        self,
        securities: List[str],
        fields: List[str],
        overrides: Optional[List[Dict[str, str]]] = None,
    ) -> List[Dict[str, Any]]:
        pass

class NewsProvider(ABC):
    @abstractmethod
    def get_theme_news(self, theme: Dict[str, Any], asof: str) -> List[Dict[str, Any]]:
        pass

    @abstractmethod
    def get_mover_news(self, movers: List[Dict[str, Any]], asof: str) -> List[Dict[str, Any]]:
        pass

class SignalEngine(ABC):
    @abstractmethod
    def compute_country_signals(self, cds_df, context_df, metadata_df):
        pass

    @abstractmethod
    def compute_region_summary(self, cds_df):
        pass

    @abstractmethod
    def score_theme_relevance(self, signals_df, theme: Dict[str, Any], news_df):
        pass

class EvidenceBuilder(ABC):
    @abstractmethod
    def build(self, run_context: Dict[str, Any]) -> Dict[str, Any]:
        pass

class NarrativeEngine(ABC):
    @abstractmethod
    def generate(self, evidence_pack: Dict[str, Any]) -> Dict[str, Any]:
        pass

class ReportRenderer(ABC):
    @abstractmethod
    def render_docx(self, report_payload: Dict[str, Any], output_path: str) -> None:
        pass
```

---

## 24. Logging and Observability

Log:
- run id
- start / end time
- Bloomberg provider status
- securities count
- fields count
- missing / stale / invalid count
- evidence item count
- model generation status
- validation warnings
- output paths

Use structured logging where practical.

---

## 25. Error Handling

### 25.1 Bloomberg Connection Failure
- fail early
- log reason
- do not continue

### 25.2 Partial Data Failure
- continue only if completeness threshold is met
- otherwise abort report generation

### 25.3 No Relevant Evidence Found
- continue
- mark report as evidence-limited
- use cautious technical narrative

### 25.4 LLM Failure
- retry once if configured
- otherwise archive failure and stop rendering

### 25.5 Rendering Failure
- keep evidence and generated text archived
- allow rerender later

---

## 26. Testing Requirements

### 26.1 Unit Tests

Cover:
- config loading
- request builder
- response parser
- stale quote logic
- z-score logic
- percentile logic
- theme relevance score
- evidence pack construction
- output validator

### 26.2 Integration Tests

Cover:
- mock Bloomberg raw response -> normalized tables
- normalized tables -> signals
- signals + theme + news -> evidence pack
- evidence pack -> narrative generation validation
- narrative -> DOCX render

### 26.3 Fixtures

Provide in `tests/fixtures/`:
- `sample_bloomberg_historical_response.json`
- `sample_bloomberg_reference_response.json`
- `sample_universe.yml`
- `sample_theme.json`
- `sample_evidence_items.json`
- `sample_llm_output.json`

The historical response fixture should represent a realistic blpapi-style response for at least 5 sovereigns and around 90 days of spread history.

---

## 27. Governance Requirements

- No final circulation without analyst approval in MVP
- Archive prompt versions
- Archive config versions
- Archive provider and model versions
- Avoid storing unnecessary copyrighted long-form article bodies
- Keep evidence traceable and inspectable

---

## 28. Recommended Implementation Order

Follow this order strictly:

1. Project scaffold and directory structure
2. Config loading and validation
3. DuckDB schema creation
4. Sovereign universe and security map loading
5. Bloomberg session manager
6. Bloomberg request builder
7. Bloomberg response parser
8. Bloomberg `blpapi` provider
9. Raw ingestion pipeline
10. Normalization pipeline
11. Data quality engine
12. Signal engine
13. Theme input flow
14. News provider abstraction and public news provider
15. Evidence pack builder
16. LLM prompt builder and narrative engine
17. Output validator
18. DOCX renderer with charts and tables
19. Archival and rerender flow
20. CLI commands
21. Unit tests and integration tests
22. README

During implementation, it is reasonable to pause every 3–4 major modules and verify coherence before continuing.

---

## 29. Acceptance Criteria

MVP is accepted only if:

1. It can connect to Bloomberg from the target laptop in live mode.
2. It can run the full pipeline in DRY_RUN mode without Bloomberg.
3. It can pull configured sovereign securities via `blpapi`.
4. It can pull configured context series via `blpapi`.
5. It stores raw and curated data locally.
6. It computes 1D / 5D / 1M changes and 30D z-score correctly.
7. It flags stale quotes.
8. It builds a valid evidence pack from a theme.
9. It generates a structured report draft.
10. It validates report claims against the evidence pack.
11. It exports a polished DOCX.
12. It archives the run reproducibly.
13. It can rerender a historical report from archived data.

---

## 30. Deliverables

Required deliverables:
1. Full Python project scaffold
2. Requirements / dependency file
3. Config templates
4. DuckDB schema creation scripts
5. Bloomberg `blpapi` provider implementation
6. Signal engine
7. Evidence pack builder
8. LLM narrative engine
9. Output validator
10. DOCX renderer
11. CLI entry points
12. Tests and fixtures
13. README

---

## 31. README Requirements

README must include:
- project purpose
- setup steps
- dependency installation
- environment variables
- how to populate Bloomberg security IDs and fields
- how to run backfill
- how to run daily theme-led report
- how to rerender a historical report
- known limitations
- how to tune thresholds and templates
- how to use DRY_RUN mode

---

## 32. Final Instruction to the Implementing Agent

Build this as a clean, modular, configurable Python project for local analyst use on a Bloomberg-enabled company laptop.

Prioritize:
- correctness
- auditability
- configurability
- deterministic transformations
- strong Bloomberg API implementation via `blpapi`
- strong post-generation validation
- offline testability via DRY_RUN mode

Do not build a fake placeholder Bloomberg layer.
Implement real Bloomberg request / response workflows using:
- `blpapi.Session`
- `//blp/refdata`
- `HistoricalDataRequest`
- `ReferenceDataRequest`
- `session.sendRequest`
- event loop parsing for `PARTIAL_RESPONSE` and `RESPONSE`

Use the user’s attached Bloomberg script as a structural reference for the API calling style, but refactor it into a proper modular architecture with:
- session manager
- request builder
- response parser
- provider
- raw archival
- normalization
- signal logic
- evidence retrieval
- evidence pack generation
- LLM narrative generation
- output validation
- rendering
- archive
