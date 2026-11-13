# 07 — ARGUS reuse review (what to port into the Polymarket bot)

> Read-only review of the sibling project `argus-project/` (full path
> `[local source path omitted]`), done 2026-10-07. Nothing there was modified
> and no tests were run (running them would write caches into ARGUS).
>
> **Sources.** Every claim cites a local file as `path:line` or `doc §section`, read on 2026-10-07.
> Labels: **[code]** = checked in source; **[doc]** = stated in ARGUS docs and not checked in code;
> **[derived]** = my arithmetic or inference. ARGUS's last commit is 2026-10-06 23:12 +05 (110 commits).

## TL;DR

- **ARGUS is a well-built LLM pipeline that has never made a trade.** It runs on OKX demo. Zero
  orders have filled, in demo or live, so there is no PnL, no calibration and no evidence that its
  forecasts are any good (`docs/project status` "Still open"). What has been proven is the LLM plumbing,
  the deployment and the safety interlocks.
- **Port the router pattern, but not as-is.** Its routing works: it ranks providers by remaining
  quota, puts a provider on cooldown after a 429 (rate-limit) reply, allows one repair attempt and
  caps attempts. Six things are missing: a USD meter, any token accounting, per-model quota
  counters, a global daily call ceiling, a strict family split when one is required, and
  provider-specific reset clocks.
- **The $2/month paid cap is dead config.** By design, and under test, the paid tier cannot be
  reached at all (`tests/agents/test_paid_fallback.py:58`). `tokens_in` and `tokens_out` are never
  written, so the journal has no data from which to compute USD cost **[code]**.
- **About 40% of the design pack's "port from ARGUS" list is net-new work.** ARGUS has no
  calibration code, no shadow ledger and no `replay --prompt-version`, and nothing in it forecasts a
  probability. The forecaster schema, the calibrator, and evidence ids with `fetched_at` must be
  designed fresh.
- **Biggest hidden risk: shared free-tier quotas.** If multiple projects share LLM credentials, scheduled evaluation jobs may bypass a router's local counters while consuming the same per-project and per-model limits.
- **The design pack has one real bug.** Its client order id hashes `attempt_no`, which breaks
  idempotency on a retry after a timeout. ARGUS derives the id from the intention, never from a
  counter (`argus/execution/idempotency.py:6,31`). (Idempotent means safe to repeat: a retried order
  carries the same id, so it is not placed twice.)

---

## 1. What ARGUS is, and its real status

**What it is.** ARGUS is an LLM-assisted swing-trading agent for OKX USDT perpetuals: 1–7 day
holds, 2–10x leverage derived by code, scheduled batch runs and free data and LLM tiers. Its
budget is about €6–10 a month (`docs/README.md`; `docs/ARCHITECTURE.md §0`). Its core stance is
*"the LLM proposes; code sizes, gates, executes"*. It runs as a Python 3.12 package of about 17.6k
lines in `argus/` (`wc -l` **[derived]**), with 792 test functions (`grep "def test_"` **[derived]**;
STATUS still says 582).

| Capability | Status | Evidence |
|---|---|---|
| LLM router against real free providers | **Proven live**: 15/15 calls in 43 s; 101 s with Groq degrading to Gemini | STATUS 2026-09-20, 2026-09-24; F-118 |
| Every configured model reachable | **Proven 2026-10-06**, except one OpenRouter model that was withdrawn | STATUS 2026-10-06 (late) |
| 10 agents with golden suites (`eval-agents`) and measured leader models | **Built**. Only 7 have a production caller: the batched trader, consensus, red team and risk debate (manager); the reviewer and post-mortem (review); the reflector (weekly). **The regime-LLM, analyst and catalyst agents run only in evals** | `grep` of `argus/runs`, `argus/cli.py` **[code]** |
| Adversarial chain: Trader → Consensus → Red Team → Risk Debate → gate | **Wired 2026-10-06** | `argus/runs/pipeline.py`; `argus/cli.py:268,319` |
| Scheduled manager and review runs on demo | **Enabled 2026-10-06.** First real run: regime TREND_UP, **0 plans** | STATUS "Decided 2026-10-06"; F-141, F-145 |
| Order placement with the stop attached (`attachAlgoOrds`) | **Demo placement tests only**: 6 orders, all priced so they cannot fill. Found the long→SELL bug (F-137) | STATUS "Still open"; F-137/F-138 |
| Watchdog stop repair | **Proven on OKX demo** (Phase 1 gate, 2026-09-17) | STATUS 2026-09-17 (evening) |
| Hash-chained ledger + `argus audit` | **Proven** by a tamper test | `argus/journal/ledger.py`; STATUS 2026-09-16 |
| GitHub CI gate | **Never ran a test until 2026-10-05**: 40/40 runs failed and were reported green (F-142) | F-142 |
| Logfire LLM tracing | Broken until F-121 and F-128 were fixed (2026-09-24 / 25) | F-121, F-128 |
| **Filled trades / PnL / profit factor / drawdown** | **None.** The dashboard table is all "—" | `docs/project status` dashboard |
| **Calibration table, shadow ledger** | **Not built.** `argus/journal/` has no `calibration.py` or `shadow.py` | `ls argus/journal` **[code]** |
| `replay --prompt-version` (CLAUDE.md requires it for every prompt change) | **Not built**: exits 2 with "landing in Phase 4" | `argus/cli.py:695` **[code]** |
| `backtest-rules` | **Stub** (`_not_yet`) | `argus/cli.py:780` **[code]** |
| Writers for `fills`, `circuit_breaker_events`, `data_quality_events` | **Missing.** Only `positions_snapshots` and `postmortems` got writers | F-132 (partly fixed) |

**Doc drift inside ARGUS.** The STATUS header still says "Mode: none (no code yet)". "Ideas not yet
scheduled" lists post-mortem wiring, yet `write_postmortems` is called at `argus/cli.py:545`. This
is the lesson behind keeping one source of project truth.

---

## 2. The ARGUS approach, on one page

**Thesis** (`docs/ARCHITECTURE.md §0`):

1. LLMs given full autonomy lose money. In Alpha Arena S1, 4 of 6 models lost 30–63% (F-015). So
   the LLM is an analyst: it never sizes, never chooses leverage and never widens a stop.
2. The budget is tiny, so calls are batched, cached and kept compact.
3. Free data is unreliable in repeatable ways (18 known traps). So every number carries provenance,
   "no data" never means 0, and data-quality gates can veto a trade.

**Run types** (`ARCHITECTURE.md §3`; `deploy/systemd/*.timer` **[code]**):

| Run | Schedule (UTC) | LLM calls in production | Job |
|---|---|---|---|
| Manager | 00:30, 12:30 | 1 batched Trader + ≤2 plans × 3 vet calls = **≤7** [derived] | regime (code) → universe → dossiers → Trader → vet → gate → place |
| Review | 00:15, 06:15, 12:15, 18:15 | 1 per open position + post-mortems | Sync, check deterministic flags, Reviewer decides HOLD/CLOSE/TRIM/MOVE_STOP, gate |
| Watchdog | every 20 min | 0 | HALT → reconcile → verify or repair stops → margin check → drawdown kill-switch |
| Reflect | Sun 09:30 | 1 | Lessons drawn from closed trades |
| eval-models | daily 08:30 | 3 cases × 23 models ≈ **69** [derived] | Health and quality screen of every model (bypasses the router counters, see §3) |

**Pipeline in production** (`argus/runs/pipeline.py` **[code]**):

```
dossier (typed, provenance) ─► Trader (1 batched call, ≤2 plans or NO_TRADE)
   ─► Consensus  (different family, never sees the first answer)   VETO, not vote
   ─► Red Team   (read by severity: <3 ×1.0, 3 ×0.5, 4–5 drop)    F-136
   ─► Risk Debate (proceed / reduce / decline = 1.0 / 0.5 / 0.0)
   ─► size_factor = product of factors (can only shrink) ─► Risk Gate (code, 14 rules) ─► execution
```

**Agent criticality** (`LLM_AGENTS.md §5.2`; `config/models.yaml agents:`). Each agent declares
what its failure means:

- `required` (regime, analyst, trader): no answer means no signal.
- `safety` (redteam, riskdebate, consensus, reviewer): a failure **drops the trade**. Silence from
  an agent whose job is to object is not approval.
- `optional` (catalyst, postmortem, reflector): the run continues and only the log notices.

**Decision logic** (`DECISION_LOGIC.md §4–6`):

- **Scoring.** Five factor scores, each from −2 to +2. A non-zero score must cite dossier field
  paths, or it is set to 0. Trade only when the total is ≥ +4 with no −2 (≥ +6 against the regime).
- **Confidence.** LLM confidence is tracked, never used for sizing (F-024).
- **Sizing.** Risk-based: `risk_usd = equity × 1% × regime_mult × protection_mult`, then
  `notional = risk_usd / stop_dist`, then caps.
- **Small accounts.** A small-account ladder raises risk in steps up to 5% below $500.
- **Gate.** Ordered rules; the first REJECT wins. An ADJUST that cuts below 50% of the implied size
  becomes a REJECT, and the comparison uses cumulative multipliers, not rounded notionals (F-054).
- **Protections** (freqtrade-style): daily 3%, weekly 6%, a cooldown after 2 stop-outs, and turnover
  caps of ≤2 entries per run and ≤3 per day.

**Guardrails, in three layers** (`OBSERVABILITY_AND_GUARDRAILS.md §2`):

1. **Output: Pydantic `Strict` types** (`extra="forbid"`).
   - The schemas have no size or leverage fields.
   - Validators enforce side and price consistency and the citation rule.
   - One output shape, never a union (F-087).
2. **Input: `scrub()`.**
   - Strips HTML and control characters and truncates text (200 chars for headlines, 600 for
     excerpts).
   - A regex flags injection attempts without dropping the text.
   - Text is wrapped in `<evidence>` blocks, with `<` and `>` neutralised.
3. **Behaviour: the Risk Gate** (code).

Lessons the Reflector proposes must be grounded: every cited trade must actually have been shown,
there must be ≥3 supporters, and risk-legislating text is refused (`argus/journal/lessons.py:95–118`).

**Memory loop.** Post-mortem (per trade) → Reflector (≤30 trades) → `lessons.py` decides →
`{lessons}` in the Trader prompt (at most 10 active, 60-day expiry). Promotion has two bars:
≥3 supporters is enforced by the type; supporters ≥3× contradictions is enforced in code and
hidden from the model (F-115).

---

## 3. The LLM router (`argus/llm/router.py`, 523 lines; `config/models.yaml`, 193 lines)

### 3.1 Providers and measured limits (`config/models.yaml`, read 2026-10-07)

| Provider | Family | Configured rpm / rpd | Measured note | In a tier? |
|---|---|---|---|---|
| gemini | google | 15 / 500. Per model: 3.5- and 3.1-flash-lite 15/500; **3.5-flash and 3.8-flash 10/20** | 20 RPD for big flash, per Google's 429 body (F-088); quota resets at **midnight Pacific** (F-108) | yes (4 models) |
| groq | groq-open | **4** / 1000 | Metered in **tokens**: 8000 TPM, so about 4 structured calls/min (F-113); output capped at 1000 tokens/min, so `max_tokens` is 800 (F-089) | yes (3) |
| openrouter | openrouter | varies by account and model | Free-model requests may receive 429 responses from upstream congestion (F-084); verify current limits for the account used | yes (2) |
| mistral | mistral | 6 / 200 | Free tier serves small models only; mistral-small and medium always 429 (F-096) | yes (4) |
| cloudflare | cloudflare | 30 / 200 | Billed in **neurons**: 10k/day ≈ 200–400 calls; known fault where the reset does not happen (F-094, F-108) | yes (8) |
| zhipu | zhipu | 3 / 100 | Only glm-4.x-flash is free; glm-5.3-flash is paid | yes (2) |
| nvidia | nvidia | 40 / 2000 | Not measured | **no**: configured but unused |
| deepseek, openai | deepseek, openai | 60 / 100000, `paid: true` | — | only `paid_fallback`, which **no agent can reach** |

There are 23 unique free models across the `fast`, `strong_json` and `reasoning` tiers
**[derived]**. Gemini flash-lite leads every tier in the reviewed configuration; verify the applicable limits before use.

**Measured leader per role** (`LLM_AGENTS.md §5.1`). Scores come from each role's own golden
suite. Trader: gemini-3.5-flash-lite, 8/10, 0 invented, 3.1 s. Reviewer: groq qwen3.8-27b, 18/20.
Consensus: open-mistral-nemo, 6/7. Red Team: qwen3.8-27b, 6/6.

### 3.2 Selection (pure code, tested offline)

- **Headroom bands.**
  - `RESERVE=0.1` (`router.py:38`): with <10% of quota left, a provider is not offered.
  - `COMFORTABLE=0.4` (`:39`): with <40% left, it is demoted behind roomier providers.
  - Headroom is the tighter of the RPM window (`llm_attempts`, 60 s) and the daily count (`llm_quota`).
- **429 cooldown.** 90 s normally; 6 h when the error body says the daily quota is spent
  (`:45–55`, F-107). A 429 does **not** trip the circuit breaker ("not now" is not "broken").
- **Circuit breaker per provider.** Genuine errors trip it; schema failures count as successes for
  the breaker.
- **Family policy.**
  - Each agent declares a tier and an ordered family preference.
  - An adversarial agent (redteam, consensus and reviewer judge the trader) has the counterpart's
    family removed.
  - **If nothing else is available, it degrades to the same family with only a log warning**
    (`:334`).

### 3.3 Execution

- **Caps and timeouts.** `max_attempts: 4` (`:393`), a 600 s deadline per agent (`:209`), a 45 s
  provider timeout, and `max_tokens` always sent (2000; Groq 800).
- **Temperature per agent.** Reviewer 0.05, Trader 0.1, Red Team 0.5.
- **Attempt flow.** A schema failure gets **one repair** on the same model: the validation error is
  appended to the prompt. Then the router moves to the next model. When every candidate fails it
  raises `AllModelsFailedError`, and the caller applies the agent's criticality.
- **Transport.** Pydantic AI `Agent(model, output_type=…)`: native for Gemini and Mistral,
  OpenAI-compatible for everyone else. No LiteLLM (supply-chain incident, F-033).

### 3.4 Journaling (`migrations/005_llm.sql`, `006_llm_attempts.sql`)

Every attempt is a row in `llm_calls`: agent, tier, provider, model, family, attempt number,
status, latency, prompt version, full prompt, output and error. `llm_attempts` is written *before*
the call so in-flight requests count toward the RPM window, and `llm_quota` counts calls per
provider per UTC day.

### 3.5 Gaps to fix when porting (all checked in code)

| # | Gap | Evidence | Port fix |
|---|---|---|---|
| R1 | **No USD meter or cap.** `paid_llm_monthly_cap_usd=2.0` exists but nothing reads it. The paid tier is unreachable by design | `argus/config.py:64`; `tests/agents/test_paid_fallback.py:58` | Add `usd_spent` per call from token usage × a price table, a monthly ledger, and refuse at the cap. Keep the "paid unreachable unless the cap is >0" test |
| R2 | **Tokens never recorded.** `record_call` is called without `tokens_in` / `tokens_out` | `router.py:429–470` | Use the Pydantic AI result's `usage()`; required for R1 |
| R3 | `raw_output` holds `str(answer.output)`, the repr of the parsed object, not raw text or reasoning, although the schema comment says "including reasoning" | `router.py:437`; `005_llm.sql` | Store `model_dump_json()` plus the raw text where available; add `input_hash`, `repair_used`, `evidence_ids` columns (`LLM_AGENTS.md §7` promises them; they do not exist) |
| R4 | Quota is counted **per provider** but limits are **per model** (Google and Groq ration per model) | `router.py:222,255` vs `limits_for()` | Key counters on (provider, model) wherever the provider meters per model |
| R5 | The daily counter resets at **00:00 UTC**; Gemini resets at midnight Pacific | `router.py:217`; F-108 | Store a reset timezone per provider |
| R6 | 429 cooldowns are process-local (`_cooldowns` dict) | `router.py:213` | Persist them in SQLite. `pm-research` runs as separate timer processes |
| R7 | `eval-models` calls models directly (`build_model` + `Agent`) and **bypasses the quota counters** | `golden.py:654–663` | Route evals through the same counters, or give them separate keys |
| R8 | **No global daily call ceiling.** The "hard ceiling 100/day" exists only in the docs | no code hit for a ceiling | The design pack's 75/day hard stop is new code |
| R9 | The family split degrades silently. `LLM_AGENTS.md §5.4` says the router "never" lets a pair share a family; the code does | `router.py:329–336` | Add a `require_distinct_families` flag. For K-sample forecasts, fewer than 2 families means `INSUFFICIENT`, never a degraded answer |
| R10 | The `timeout` status is documented but never emitted. A timeout lands as `error` and trips the breaker | `router.py:443–455` | Classify `TimeoutException` separately (cooldown, no breaker trip) |
| R11 | Model ids live in YAML (good), but the catalogue rotates monthly (F-063, F-044) | — | Keep `doctor --models` (256-token ping, F-064) plus a property-based test of the catalogue |

---

## 4. Top FAILURE_LOG lessons for an LLM Polymarket bot

Source: `argus-project/docs/FAILURE_LOG.md` (169 rows, read 2026-10-07). One line each, plus how
it applies here.

| F-id | Lesson | Polymarket application |
|---|---|---|
| F-001 | A "must produce ≥1 signal" prompt caused overtrading | Abstaining (`AMBIGUOUS`, `INSUFFICIENT`) is a first-class answer, with turnover caps |
| F-015 | LLMs with full autonomy lost 30–63% (Alpha Arena) | The LLM outputs only P(YES) and flags; code does everything else |
| F-024 | Kelly sizing on raw LLM confidence inflates size | Kelly only on **calibrated** q after N resolved forecasts; until then q = market price |
| F-016 / F-063 | Hard-coded model ids rot; tests pinned ids, so a dead catalogue looked "tested" | Ids in config; tests assert properties; daily `doctor --models` |
| F-017 / F-086 | Bursty 429s; unset `max_tokens` made reasoning models return nothing; an unbounded fallback chain ate the run budget | Always send `max_tokens`; cap attempts; put a deadline on each agent |
| F-087 | A union output type made weak models invent a tool name | One discriminated `verdict` shape for the forecaster |
| F-088 / F-094 / F-113 | Published free limits ≠ real limits (20 vs 250 RPD; neurons; tokens per minute) | Measure every provider's limit before relying on it |
| F-107 / F-108 | Counters do not see other consumers; quota reset clocks differ by provider | Separate keys from ARGUS, or a shared quota ledger; per-provider reset timezone |
| F-019 | A headline containing instructions changed the scores | Market descriptions, comments and news go in `<evidence>` blocks; injection test cases are mandatory |
| F-119 | The model restated the most salient label instead of reading the data (anchoring) | Hide the market price in pass 1; ask for reasons tied to specific evidence ids |
| F-102 / F-111 | LLMs are poor at arithmetic buried in JSON | Code computes days to resolution, fee per share and implied odds, and passes them as facts |
| F-092 / F-099 | An unstated sign convention and an ambiguous rule ("at 1.5R") produced systematic refusals | State conventions explicitly: P(YES) of *this* token, the rules text is authoritative, and so on |
| F-115 | Telling the model the code's threshold made it pre-filter its own answers | Keep `min_edge` and spread thresholds out of the prompt |
| F-117 | A per-item call shape inherited from a stub meant 15 calls where 1 was designed | Design the call shape around the budget (shortlist ≤15, K samples) from day one |
| F-145 | A scheduled job silently ran the deterministic stub, and 17 plans looked like real decisions | The snapshot reports `forecaster=llm` with model ids; an armed run refuses the stub; no stub in production paths |
| F-136 | Safety agents can block everything and still score perfectly on veto-only suites | Golden suites need "agree / proceed" cases |
| F-090 / F-097 / F-105 / F-116 | Fixture and scorer bugs looked like model failures | Test the golden fixtures and the scorer itself |
| F-095 | Screening models in parallel made them rate-limit each other | Run evals serially, with gaps between calls |
| F-093 | `str.format` treated a literal `{` in a JSON example as a placeholder | Test that every template renders with its JSON example intact |
| F-029 | A cooldown counted successes instead of attempts | Rate limits and caps count attempts (rule 12 of the pack's gate) |
| F-062 | Overlapping runs could double-place orders | `flock` single-instance lock per wallet |
| F-071 / F-073 | A partial list returned as complete; a missing balance row read as equity 0, which reads as a 100% drawdown | A partial or missing venue answer **raises**; never turn "missing" into 0 |
| F-069 / F-075 | The mode interlock checked one direction only; a malicious host string passed validation | Validate all 4 paper/live × key quadrants; allowlist the CLOB host |
| F-070 / F-133 | Bot token and API keys leaked through span URLs and exception text | `scrub_url()` and `scrub_token()`; never interpolate an httpx exception unscrubbed |
| F-127 / F-137 / F-138 / F-139 | Every long went out as a SELL; a resting order was journaled REJECTED. All were seam bugs between individually tested units | End-to-end integration test: forecast → gate → order payload actually sent (YES/NO token, BUY/SELL) |
| F-143 / F-144 | No component owned resting orders; cancelled orders stayed `open` in the journal | GTD expiry on every order, plus a sweep that reconciles order state by venue id |
| F-125 / F-129 / F-130 / F-132 | Wrong `shadow` flag; venue ids never stored; thesis dropped at a boundary; tables with no writer | Journal mode from the actual arguments; store the venue order id; carry the rationale as JOURNAL ONLY; test that every table has a producer |
| F-141 / F-142 | The timer was never enabled for 6.7 days; CI never ran a test (40/40 failures reported green) | Test the installed units; check that CI actually reaches pytest |
| F-043 | The test suite read the real `.env` | Clear secret environment variables in `conftest.py` |
| F-020 | A long-running scheduler died silently for days | `pm-exec` is long-running, so a dead-man switch is mandatory |

---

## 5. Port table

Line counts from `wc -l` on 2026-10-07. Verdicts: **as-is** = copy and rename; **adapt** = keep the
design, change the code; **pattern** = reimplement the idea; **skip**.

| ARGUS module | Lines | What it does | Verdict | Notes for the Polymarket bot |
|---|---|---|---|---|
| `argus/llm/router.py` | 523 | Tiers, headroom routing, 429 cooldown, breakers, family exclusion, one repair, attempts journaled | **adapt** | Fix R1–R10 (§3.5). Add a `sample_k(agent, k, distinct_families=True)` helper for the ensemble |
| `config/models.yaml` | 193 | Providers, tiers, measured limits, agent policies, temperatures | **adapt** | Re-measure every limit; keep the measurement comments; add `usd_per_mtok_in/out` and `reset_tz` per provider; remove the 10-agent map |
| `argus/llm/schemas.py` | 432 | `Strict` base (`extra="forbid"`), single discriminated `TraderAnswer`, citation validator | **adapt** | Keep `Strict` and the discriminator pattern. New `Forecast` schema: `verdict`, `p_yes ∈ [0.01, 0.99]`, `evidence_ids`, `resolution_reading`, `ambiguity_notes`. No side, price or size fields |
| `argus/llm/prompts.py` + `argus/prompts/*.md` | 65 + 10 files | `version:` header, pure render, a missing placeholder raises | **as-is** (module) / skip prompts | The F-093 brace-escape test comes along. Prompts are new (forecaster, rules-reader) |
| `argus/data/scrub.py` | 57 | HTML/control strip, truncation, injection regex flag, `<evidence>` wrapping | **adapt** | Add `id=` and `fetched_at=` to `as_evidence`. Raise the limit for rules text (resolution criteria often exceed 600 chars: hash the full text, never truncate it silently). Strip or neutralise URLs (C2 never follows them). Tests live in `tests/data/test_scrub.py` |
| `argus/llm/agents/catalyst.py` `filter_sourced()` | 77 | Drops candidates whose `source` was not in the prompt | **pattern** | This, not `lessons.ground()`, is the direct model for evidence-id grounding |
| `argus/journal/lessons.py` | 228 | `ground()`, `unsafe()` regex, fingerprint merge, 60-day expiry, ≤10 active | **adapt (Phase 3+)** | A lesson loop over resolved forecasts later. Port `unsafe()`'s idea now: refuse any text that legislates size or limits |
| `argus/llm/golden.py` | 2871 | Per-role suites, `Expectation` enum, invented-evidence scorer, `eval-models` / `eval-agents` | **pattern** | Forecaster suite tests *behaviour*: abstains on ambiguous rules, ignores injection, cites only real ids, includes agree cases. Accuracy comes only from the forward test (resolved-market evals leak through training data) |
| `argus/journal/ledger.py` | 100 | Insert-only, SHA-256 `prev_hash` chain over 6 tables, `verify_all` | **as-is** | Ledger tables: forecasts, gate_results, orders, fills, resolutions, kill_switches |
| `argus/journal/db.py` + `migrations/` | 168 + 9 SQL | WAL + `synchronous=FULL`, numbered checksummed migrations, `start_run` context | **as-is** | Keep "never edit an applied migration" (F-085) |
| `argus/journal/states.py` | 97 | Plan, order and position state enums with terminal states | **adapt** | Order states must match the CLOB's live / matched / cancelled states (verify against V2); position ends at RESOLVED → REDEEMED |
| `argus/journal/replay.py` | 188 | Reconstructs a run from the journal, offline | **adapt** | Build the prompt-version replay ARGUS never finished (cheap, high value) |
| `argus/execution/idempotency.py` | 35 | Deterministic client order id from (run, instrument, side) | **pattern** | **Never include `attempt_no`** (see C1). Polymarket orders are EIP-712 signed; if V2 has no client-id field, use a deterministic salt or a pre-send journal row plus an open-order lookup (**verify**) |
| `argus/runs/lock.py` | 55 | `flock` lock released by the kernel on death | **as-is** | Hold it for the process lifetime in `pm-exec` (one process per wallet) |
| `argus/runs/pipeline.py` `vet()` | 122 | Consensus veto, red-team severity factor, risk debate; safety checks fail closed; factor only shrinks | **pattern** | Start with forecaster + 1 adversarial "how could this resolve the other way" check; fail closed; the factor only shrinks |
| `argus/observability/setup.py` (+ `metrics.py`, `alerts.py`, `spans.py`) | 177 + 72 + 151 + 24 | Logfire no-op without a token, httpx hook with `scrub_url`, metric constants, pure alert rules | **adapt** | Keep `scrub_url` and the pure alert functions; extend the secret pattern to 0x-prefixed 64-hex private keys and the `POLY_*` headers. Alerts go to ntfy (design pack), not Telegram |
| `argus/data/fetchers/polymarket.py` | 82 | Gamma `/events` by tag, top 10 by volume, parses `outcomePrices` defensively | **adapt (minor)** | About 10% of the scanner (C1). Missing: pagination, `clobTokenIds`, `negRisk`, rules / description, fee rate, book depth. Keep the defensive JSON-in-string parse |
| `argus/data/fetchers/kalshi.py` | 62 | Kalshi public markets | **as-is (later)** | For a future cross-venue divergence strategy (ARGUS's own Polymarket idea, S7) |
| `argus/data/http.py` + `cache.py` + `ratelimit.py` + `envelope.py` + `parallel.py` | 143 + 79 + 99 + 38 + 46 | One fetch path: per-host token bucket, diskcache, jittered retries counting attempts, `Observation` with provenance, never raises | **adapt (research only)** | Fine for the evidence ingester. **Never use it for CLOB book or prices on the execution path**: it returns cached data and turns failure into `missing` (pack rules R22, R70, R74). It also logs `str(exc)`, which can embed URLs (F-133) |
| `argus/exchange/breakers.py` | 97 | Breakers per dependency, attempt-counted, `allow(force=True)` | **as-is** | — |
| `argus/config.py` | 225 | pydantic-settings, `SecretStr`, live interlock (F-069), host allowlist (F-075), comment parsing (F-035) | **adapt** | Two switches for real money: `MODE=live` + `LIVE_TRADING=I_UNDERSTAND`. The CLOB host is allowlisted |
| `argus/doctor.py` | 521 | Connectivity, keys, model pings (256 tokens) | **adapt** | Add a geoblock assertion, V2 / pUSD / signature-type checks and a fee-rate fetch |
| `tests/conftest.py` | — | Clears secret env vars; demo and network tests opt-in | **as-is** | — |
| `deploy/update.sh`, `install-units.sh`, `tests/deploy/test_installed_units.py` | 98 + 93 | Pull, run the gate on the box, roll back, self-exec (F-083), refuse while the lock is held; every timer has an explicit enable decision | **adapt** | **Live is never restarted automatically** (pack) |
| `argus/gate/killswitch.py` | 141 | HALT first, then cancel, then close; retry marker (F-079) | **pattern** | KILL latch, cancel only; flattening is an owner decision (D8) |
| `argus/gate/protections.py` | 114 | Daily and weekly loss, cooldowns, entry caps (attempt-counted) | **pattern** | Reuse the shape; the thresholds come from pack §04 |
| `argus/gate/rules.py`, `sizing.py`, `fees.py` | 502 + 180 + 64 | Perp-specific gates (ATR stops, leverage, funding) | **skip** | Pack C5 to C7 are new |
| `argus/exchange/okx.py`, `fake.py`, `reconcile.py` | 522 + 363 + 110 | OKX adapter, in-memory fake venue | **skip** / **pattern** (fake) | Build a fake CLOB first, and test the seam with it (F-139) |
| `argus/execution/orders.py`, `place.py`, `lifecycle.py`, `resting.py` | 78 + 104 + 193 + 236 | Bracket orders; journal before send; resting sweep | **pattern** | Journal-before-send and the resting sweep (F-143/F-144) transfer; brackets do not |
| `argus/runs/watchdog.py` | 493 | Stop verification and repair | **skip** (pattern: HALT → reconcile → verify → alert) | Polymarket has no stop orders |
| `argus/llm/stub.py` | 88 | Deterministic proposer | **skip** | Caused F-117 and F-145 |
| `argus/features/*`, `data/quality.py`, `data/fetchers/*` (crypto) | ~2.9k | Indicators, regime, dossier, 18 traps | **skip** | The "every number has provenance, missing ≠ 0" ideas come with `Observation` |
| `argus/notify/telegram.py`, `argus/ui/*` | 224 + 568 | Telegram commands; FastAPI + HTMX dashboard | **skip (MVP)** | The pack uses a snapshot JSON + ntfy |
| `argus/llm/agents/{consensus,redteam,riskdebate,reviewer,reflector,postmortem}.py` | 60–92 each | Role wrappers around `router.call` | **pattern** | `consensus.compare()` and `redteam.size_factor()` are the reusable ideas |

---

## 6. Tech stack and tooling conventions worth mirroring

| Item | ARGUS choice (evidence) | Mirror? |
|---|---|---|
| Python | 3.12 (`.python-version`); `requires-python = ">=3.12,<3.14"` because numba / pandas-ta have no 3.14 wheels | Yes, 3.12. The upper bound can relax without pandas-ta; check what `polymarket-client` requires |
| Environment / locking | `uv`, hash-locked `uv.lock`, hatchling build, `[project.scripts]` CLI entry point | Yes |
| Locked versions (`uv.lock`) | pydantic-ai-slim 2.43.0, pydantic 2.13.5, logfire 5.1.0, httpx 0.28.1, SQLAlchemy 2.0.53, structlog 26.1.0, typer 0.27.2, ruff 0.16.7, mypy 2.3.1, pytest 9.1.1 | Yes, same majors (drop ccxt and pandas-ta) |
| Lint | `ruff` line-length 100, rules `E,F,I,B,UP,N,W`, `extend-exclude=["docs"]`, plus `ruff format --check` | Yes |
| Types | `mypy --strict` + pydantic plugin, **only on money paths** (`gate/`, `exchange/`, `execution/`, `journal/states.py`) | Yes: strict on edge, sizer, gate, execution and signer |
| Tests | `tests/` mirrors the package (`agents/`, `data/`, `execution/`, `integration/`, `deploy/` …); `addopts = "-q -m 'not demo and not network'"`; `respx` for HTTP; `freezegun` for time | Yes; markers `live` and `network` are opt-in |
| Makefile | `sync` (`uv sync --all-extras`), `test`, `lint`, `type`, `doctor`, `gate1 = lint type test` | Yes |
| CI | `.github/workflows/gate.yml`: `uv sync --locked --all-extras` (the F-142 fix), ruff, format, mypy, pytest, `bash -n` over deploy scripts | Yes, and check that it actually reaches pytest |
| Verification | Two-gate "done": Gate 1 automated; Gate 2 = six questions in `AGENTS.md` | Yes |
| Docs | Tiered context (CLAUDE.md, then one area doc, then source), `FAILURE_LOG.md` entry *before* a fix, `project status` changelog, `design_docs_changelog.md`, `[tune]` marks on numbers | Yes, but keep one STATUS header honest |
| Logging | `structlog` JSON to file, human console, `run_id` on every line; UTC everywhere | Yes |
| Money maths | `Decimal` in execution; floats allowed in analysis | Yes (prices in 0–1 with a tick size) |
| Runtime | Short-lived processes under systemd timers; `flock` lock; healthchecks.io ping in `ExecStartPost` | Research: yes. Exec: long-running (pack) plus a dead-man switch |
| Prompts | `.md` with `version:` / `agent:` front matter and a `---` separator; bump the version and replay the diff | Yes, and actually build the replay |

---

## 7. Where ARGUS contradicts (or does not back) the Polymarket design pack

| # | Pack says | ARGUS reality | Impact | Recommendation |
|---|---|---|---|---|
| C1 | Client order id = hash of (market, side, forecast_id, **attempt_no**), "ARGUS `idempotency.py`" (`02 §7`) | ARGUS deliberately derives the id from the intention and "**never from a clock or a counter**". A retry after a timeout must reuse the id (`argus/execution/idempotency.py:3–8,31`) | Including `attempt_no` lets a lost-reply retry place a **second** order | Drop `attempt_no`. Verify whether CLOB V2 has a client-id field; if not, use a deterministic salt and look up open orders before any retry |
| C2 | ARGUS has "a paid fallback" (`02 §8`); its "$2/mo cap was design-only" | Correct about the cap. But the paid tier is also **unreachable** (test-enforced), and tokens are never journaled | Porting the router does not give you a USD meter; there is nothing to extend | Build token capture, a price table and a monthly USD ledger first (R1, R2). `PM_LLM_USD_CAP_MONTH=0` means paid models stay unreachable |
| C3 | "Side, size, price, order type do not exist in [the LLM] schema (ARGUS pattern)" (`02 §3`) | ARGUS `TradePlan` *does* carry `side`, `entry_kind` (limit/market), `entry_low/high`, `stop`, `targets`; only size and leverage are absent (`schemas.py:68–86`) | The pack is stricter than ARGUS (good), but the forecaster schema is **new**, not a port | Design `Forecast` from scratch on the `Strict` base |
| C4 | K samples across ≥2 families; disagreement means `INSUFFICIENT` (`02 §4`) | The router only excludes a single counterpart's family, and **degrades to the same family** when nothing else is available (`router.py:329–336`). ARGUS docs claim it never does (`LLM_AGENTS.md §5.4`) | A degraded "ensemble" of one family looks like agreement | Add a hard `distinct_families` mode that fails closed to `INSUFFICIENT` |
| C5 | A 75-call/day budget on free tiers, "reuses that router" (`02 §8`) | Scheduled runs and evaluations can share provider limits while bypassing a router’s counters (R7). Gemini quotas can be per project and model (F-088). F-107 already saw "a provider exhausted by another project … looks fresh" | Both bots can starve each other, and each router's headroom figure is wrong | Separate API keys / Google projects for the Polymarket bot, or one shared quota ledger. Decide before Phase 2 |
| C6 | Scrub tests live in `tests/agents/test_injection.py` (`05` l.131) | The file does not exist. Coverage is in `tests/data/test_scrub.py`, plus injection cases in `tests/agents/test_prompts.py` and `tests/features/test_dossier.py` | Wrong pointer | Fix the reference |
| C7 | Evidence as `<evidence id=… source=… fetched_at=…>`, with full rules text (`02 §4`) | `as_evidence()` emits `source` and `injection_suspect` only, and `scrub()` truncates to 200 / 600 chars (`scrub.py:15–16,39–43`) | Rules text would be silently cut, and grounding has no ids | Extend `scrub` (§5 row) |
| C8 | The journal stores prompt hash + version and evidence hashes (C9) | `llm_calls` has no `input_hash` or evidence ids, and `raw_output` is a repr; it is **not** hash-chained (`005_llm.sql` comment) | Forecasts would not be auditable as designed | Make `forecasts` a ledger table with prompt and evidence hashes |
| C9 | `pm-exec` is a long-running service (`02 §9`) | ARGUS's thesis is short-lived timer processes; its run lock is per-run. F-020: "long-running scheduler died silently" | A new failure class ARGUS never faced | Hold the lock for the process lifetime; add a dead-man switch (healthchecks-style ping or an ntfy heartbeat); the long-running process must refuse loudly (R73) |
| C10 | "ARGUS … LLM plumbing is proven" (`05` l.143–145) | Mostly true (§1). But CI ran no tests until 2026-10-05 (F-142), `replay --prompt-version` is unbuilt (`cli.py:695`), and 3 of 10 agents have no production caller | Over-trust in "ported = tested" | Re-run the ported tests in the new repo; do not inherit ARGUS's green claims |
| C11 | The calibrator (C4) is the core gate | ARGUS has **no** calibration or shadow-ledger code (designed in `ARCHITECTURE.md §11, §13`, never built) | Nothing to port | C4 (isotonic or Platt fit, Brier/ECE, stored N) is net-new and on the critical path |
| C12 | Daily ceiling "hard stop in code" (`02 §8`) | ARGUS's "hard ceiling 100/day" exists only in the docs (R8) | Same | New code plus a test |
| C13 | P1 (LLM forecast) is the first strategy (`00`, decision 6) | ARGUS's own Polymarket plan was **cross-venue divergence**, Polymarket vs Kalshi vs futures-implied (`SIDE_PROJECTS.md` SP4; `STRATEGY_STACKS.md` S7) | Not a conflict, but a ready-made second-strategy candidate (`kalshi.py` exists) | Record it in the strategies roadmap next to P3 |
| C14 | The scanner starts from ARGUS `fetchers/polymarket.py` (`05` l.136) | It goes through `fetch_json`, which serves diskcache hits and returns `missing` instead of raising | On the exec path that breaks "never act on cached or absent data" (R22, R70, R74) | Research path only; the exec path reads CLOB data live, and failures raise |
| C15 | "SHRINK below 50% → REJECT, as in ARGUS (`DECISION_LOGIC.md` l.130)" | The rule is in the §6 footer (l.128). F-054 shows it must compare **cumulative multipliers**, not rounded sizes | Rounding to the venue step would cause false REJECTs | Port the F-054 nuance, with its test |
| C16 | Core safety metric | ARGUS's invariant is "every position has an exchange-side stop", with a watchdog repair loop. Polymarket has no stops | The whole watchdog design does not transfer | Replace it with "every stake sized as a total loss" plus reconcile and redeem jobs |

## Unverified / to check before relying on

- CLOB V2 order fields: is there a client order id? How do salt, nonce and expiry work, and do
  GTD/FOK semantics match? ARGUS has no V2 knowledge; this needs an official docs read.
- Whether `polymarket-client` pins a Python range that conflicts with 3.12.
- Whether ARGUS's 2026-09 provider limits still hold. Free tiers rotate monthly (F-044); re-measure
  them on the Polymarket bot's own keys.
