# Decisions log

One row per decision. **Status:** `final` (owner decided), `provisional` (default applied because the
owner has not answered yet — confirm or override), `proposed` (not in the design until the owner approves), `lead` (technical choice by the coding agent;
the owner may override). Superseded rows stay, struck through, with a pointer to the new row.

Longer reasoning lives in the linked section; this file is the index.

## Owner decisions carried from the design pack (`docs/design/00-README.md`)

| ID | Decision | Status | Date | Where |
|---|---|---|---|---|
| D-001 | Separate repo and project from ntx and ARGUS | final | 2026-10-06 | — |
| D-002 | Owner accepts a signing key that can withdraw; still prefer a no-withdraw **session key**; float-only wallet; key never in argv, logs or git | final | 2026-10-06 | RISK_AND_CUSTODY §1 |
| D-003 | Budget: $50 start, up to $100–150 (owner, 2026-10-07); free data; isolated hosting | final | 2026-10-07 | — |
| D-004 | LLM forecasts and selects only; deterministic code sizes, gates, signs and places | final | 2026-10-06 | ARCHITECTURE §3 |
| D-005 | Official `polymarket-client` SDK + thin custom engine + ARGUS-style LLM router; **not** NautilusTrader | final | 2026-10-07 | design/01 |
| D-006 | First strategy P1 (market-anchored LLM forecast on long-tail markets); other strategies documented and built later | final | 2026-10-07 | STRATEGIES §P1 |
| D-007 | Validation is forward-only; resolved-market LLM backtests are not evidence | final | 2026-10-06 | BUILD_PLAN §Gates |
| D-008 | Never bypass a geoblock (no VPN/proxy/relocation); a block means wind down | final | 2026-10-06 | RISK_AND_CUSTODY §5 |
| D-010 | Git: owner makes the first 2–3 commits; afterwards the agent may commit and push (owner grant, 2026-10-07, takes effect after those commits) | final | 2026-10-07 | AGENTS.md §Git |
| D-011 | Paper trading first, real money only after it | final | 2026-10-07 | BUILD_PLAN |

## Grilling round 1 (asked 2026-10-07; owner said "proceed" on 2026-10-08 without answering → defaults applied)

| ID | Question | Applied default | Status |
|---|---|---|---|
| D-012 (Q1) | Goal at 3 months | Prove or kill an edge cheaply; $5–10k/month is a later scale question that needs evidence and real capital (see `research/04`) | provisional |
| D-013 (Q2) | Paper → live | Paper simulator + **plumbing canary** (≤ $10 total, venue-minimum orders) once execution exists; strategy capital waits for gate G2 | provisional |
| D-014 (Q3) | G2 design | Scoring universe ≈ 15 new markets/day (seeded random sample when more are eligible), separate from the trading shortlist. G2 unit = first forecast per market, event-clustered; early kill at 100; final at `N_final` from the Phase 1 power simulation (default 350); one extension to 600; outcomes PASS / FAIL / INCONCLUSIVE (BUILD_PLAN §G2) | provisional |
| D-015 (Q4) | LLM spend | Free models first (ARGUS-style router), then Vercel AI Gateway's free $5/month credit (140 of 267 models usable, measured 2026-10-08; enough for a full sample-C family); paid fallback chain DeepSeek → OpenAI → OpenRouter (owner direction 2026-10-08); hard cap **$5/month cash, binding** (sub-caps DeepSeek $2, OpenAI $1.5, OpenRouter $1, TypeSafe $0.5) + ~$0.33/day, enforced before each call; chain order per role in DATA_SOURCES §3; a paid model fills at most one sample slot ([research/10](research/10-paid-llm-fallback.md)) | provisional (cap amount) |
| D-016 (Q5) | Custody | Owner emails builder@polymarket.com for a Builder key → session key; fallback hot float wallet ≤ phase cap | provisional (owner action) |
| D-017 (Q6) | Human in the loop | Telegram approve/reject for the first 20 **live** trades (outbound-only bot, proposals expire in 15 min, re-gated after approval); every canary order is approved too and does not count toward the 20 | provisional |
| D-018 (Q7) | Multi-strategy | Strategy interface + per-strategy budget and journal tags in v1; only P1 goes live first; P6 weather runs in paper (D-022); deterministic observers run alongside | provisional |
| D-019 (Q8) | Niche | Long-tail universe chosen by volume and text-heaviness (geopolitics is fee-free but liquid, so it is a slice, not the universe), resolving in 3–45 days; owner's niche added as a tagged slice when named | provisional |
| D-020 (Q9) | Reporting | ntfy alerts + daily markdown/HTML report; no web UI in v1 | provisional |
| D-021 (Q10) | Runtime | 24/7 on an isolated host from Phase 1 under a separate Linux user; develop on the local computer | provisional |
| D-022 (Q11) | Weather | Weather forecasting (no LLM) as paper strategy #2 from Phase 2 on the same engine | provisional |
| D-023 (Q12) | Execution style | **Maker-first**: post-only GTD limits at fair value minus margin; take only when edge > fee + margin (or fee-free market); paper maker fills follow the pre-registered rule in ARCHITECTURE §11.2 | provisional |
| D-024 (Q13) | LLM keys | New Google AI Studio project, Mistral and OpenRouter keys for this bot only; never share ARGUS keys; $0 search grounding | provisional |
| D-025 (pack D7) | Risk defaults | Use RISK_AND_CUSTODY §2 defaults (¼ Kelly, 10% per market, 30% DD → KILL, …) | provisional |
| D-026 (pack D8) | KILL behaviour | KILL cancels resting orders and stops new entries; it never auto-flattens into thin books | provisional |
| D-028 (synthesis) | Min-lot policy | When quarter-Kelly is below the 5-share lot, round up to the lot only if its cost ≤ the **full**-Kelly stake and ≤ every cap; else REJECT. Amends R6 for this project (needs owner OK under R14). Paper runs with it now; live uses it only once this row is final, otherwise strict R6. Without it almost nothing trades at $50 (ARCHITECTURE §9) | provisional |
| D-035 | Jev (TypeSafe) fast classifier | **Proposed, awaiting owner approval.** Use for scanner triage, evidence relevance and an extra injection pre-filter only; never for P(YES), sizing or text; served via **TypeSafe direct** (primary; reported $5 signup credit) with **OpenRouter** as fallback, both through the official `typesafe-sdk` (Vercel's free tier excludes Jev, tested 2026-10-08); Gemini fallback via the MIT adapter; 2-week shadow test before promotion; TypeSafe sub-cap $0.5 inside D-015 ([research/11](research/11-jev-system-one.md)) | proposed |
| D-036 | LLM roster by role | Sample A/B: Gemini free direct; **sample C: Vercel free-credit `grok-4.1-fast`** (backups: Mistral free, OpenRouter :free, Vercel `deepseek-v3.1`, then paid); short roles: Vercel `gpt-5-nano`; paper ablation decides sample C among Grok / DeepSeek / gpt-5-mini ([research/12](research/12-vercel-free-tier-models.md)) | provisional |
| D-037 | Area coverage ("bet on everything") | All 29 areas are tagged and scored from Phase 1; trading happens per area only through a strategy whose gate passed (P1 evidence areas; P6 weather; P10 sports, P11 esports, P12 crypto ladders as backlog strategies); insider-contaminated and single-decider markets are excluded ([STRATEGIES §Area coverage](STRATEGIES.md)) | provisional |
| D-038 | Non-commercial-licence sources (Jolpica, MLB StatsAPI, Finnhub, Twelve Data, Open-Meteo, Google News RSS, PredictIt) | Allowed in paper research only; any one feeding a live decision needs owner sign-off on its terms | provisional |
| D-029 | Weather data licence (P6) | Use only sources whose licence allows personal trading use (NOAA GEFS/NWS, public domain; ECMWF open data if its licence is confirmed in DATA_SOURCES); Open-Meteo's free API (non-commercial) only if the owner accepts the risk | provisional |
| D-030 | Protocol V2 markets | Forecast, score and paper-trade them now; trade them live only after the canary has exercised their signing and redeem path | provisional |
| D-031 | OpenRouter $10 one-time credit purchase | **Yes** (~$10.80 incl. fee): raises `:free` models from 50 to 1,000 requests/day; the balance doubles as the $1/month paid pool | provisional |
| D-032 | Crash behaviour | Heartbeat on for resting live orders: a crashed exec has all resting orders cancelled within 10–15 s | provisional |
| D-033 | NVIDIA free models | Evaluation-only terms: paper research ensemble at most, never an input to a live decision | provisional |
| D-034 | Metaculus AI Benchmark entry | Not now (deferred) | provisional |

## Technical decisions by the lead (owner may override)

| ID | Decision | Why | Alternatives rejected |
|---|---|---|---|
| T-001 | One codebase, one `Venue` interface with two adapters: `LiveVenue` (py-sdk) and `PaperVenue` (simulator over live books). Strategies, sizer, gate and journal never know which one runs | Paper results are only meaningful if the same code goes live; avoids a rewrite at Phase 3 | Separate paper script (diverges); NautilusTrader sandbox (D-005) |
| T-002 | Two processes per environment: `research` (LLM keys only) and `exec` (signer only); they share the SQLite journal, nothing else | A compromised LLM pipeline cannot reach the signer (design/02 §2) | Single process |
| T-003 | Order identity (`intent_id`) derived from the **intention** (strategy, market, token, side, forecast id, limit price), never from an attempt counter | The pack's `attempt_no` breaks idempotency on retry after a timeout (`research/07`) | Counter-based ids |
| T-004 | Python 3.12, `uv`, `ruff`, `mypy --strict` on core packages, `pytest`, `pydantic` v2, `httpx`, stdlib `sqlite3` (WAL), systemd timers; no web framework, no ORM, no scheduler library | Mirrors ARGUS tooling; fewest dependencies that do the job | SQLAlchemy, APScheduler, FastAPI (not needed in v1) |
| T-005 | Journal is insert-only and hash-chained (ARGUS `ledger.py` pattern); the venue stays the source of truth for positions and balances | Audit trail for forward evaluation; ntx R27 | Mutable state tables as truth |
| T-006 | ~~LLM forecaster output schema has no order fields; calibrator and blender are code; LLM weight in the blend starts at 0 and is fitted on our own record (prior 0.25–0.33 only once N is reached)~~ → superseded by T-010 | Literature: LLM alone loses to liquid markets; blend in code beats market slightly (`research/02`) | Showing the price to the LLM in pass 1 |
| T-008 | Ensemble `K = 3`, one sample per model, ≥ 2 families, configurable; a paper ablation K = 3 vs 6 decides | Free capacity ~95–135 long calls/day; G2 power needs more markets more than more samples | K = 6 everywhere (halves markets/day) |
| T-009 | G2 counts **distinct events**; bootstrap resamples by event (cluster bootstrap) | Markets in one event are correlated; counting them separately overstates evidence | Per-forecast counting |
| T-010 | One calibrator on the ensemble aggregate (identity until 500 resolved, clip [0.03, 0.97]); blend weight `w_prior` fixed (P1 0.30, P6 0.50) through G2 and early live, refit only by a separately pre-registered refit; cohorts freeze roster, prompt, calibrator, `w` | Per-family calibration before data exists is noise; a moving `w` makes G2 unfalsifiable (review 2026-10-08) | Weight starting at 0 (G2 could never pass); per-family Platt |
| T-011 | Journal split: `pythia.db` (decisions; per-table hash chains) and monthly `market/market-YYYY-MM.db` + `weather/weather-YYYY-MM.db` (high-rate, rotating, unchained) | One chain across many writers serialises everything; recorder bursts would block exec | Single DB, single chain |
| T-012 | Sign once, journal the signed order and its hash before sending, resubmit only identical bytes, re-sign only after a proven terminal state | CLOB V2 has no verified client-id field | Venue client ids (unverified) |
| T-013 | NAV equity with a `flows` ledger; exposure counted if filled (positions + resting buys); cluster sizing for mutually exclusive outcomes | Sweeps and deposits would otherwise fake PnL and HWM; per-bucket Kelly overbets neg-risk sets | Cash + marks only |
| T-014 | Two Linux users (`research-process`, `execution-process`) in group `pythia`, env files 0600 per user, hardened systemd units | One shared user would let the LLM process read the signer key, defeating PM3 (review 2026-10-08) | Single user `pythia` |
| T-015 | Gated cohorts are pinned: no fallback for forecast roles during a gate; missing sample → NO_FORECAST + alert; cohort carries a `config_hash` | Silent fallbacks would change the system under test mid-gate | Fallback with the realised roster recorded |
| T-016 | Hash chains are computed over a stored `canon_json` per row (+ `hash_version`) | Lets schema migrations add columns without breaking verification | Hash over current columns |
| T-017 | Global scan records every market; strategies apply their own `universe:` blocks; generic `model_runs` + `signal_snapshots` + `labels` tables serve P6 and P10–P12 | A global 3–45-day filter and LLM-shaped tables would block weather, sports, esports and crypto strategies | Strategy-specific tables later |
| T-007 | Project name **PYTHIA** (oracle of Delphi), package `pythia` (owner confirmed 2026-10-08) | Short, fits a forecasting oracle; internal package, never published (an unrelated `pythia` physics framework exists on PyPI, so never `pip install pythia`) | — |
