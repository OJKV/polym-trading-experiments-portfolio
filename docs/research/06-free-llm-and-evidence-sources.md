# 06 — Free LLM tiers and free evidence sources (as of 2026-10-07)

**Track:** what a forecaster making ~50–150 long-context calls a day (rules + evidence ≈ 8–20k input tokens,
~1–2k output) can run on for $0, and what $5–10/month buys.
**Method:** official pricing/limits pages first, then public APIs probed read-only (`curl`), then ARGUS's own
measurements (`argus-project/docs/LLM_AGENTS.md` §5.3b, `LLM_PROVIDERS_AND_MODELS.md`, measured 2026-09-24).
**Labels:** `[Sn]` = source in §6, all accessed **2026-10-07** unless stated. **derived** = computed by us.
**claimed** = third-party or user report, not confirmed by the provider. **measure** = must be confirmed on our
own key in week 1 (ARGUS F-088/F-094/F-113: free limits read from docs were wrong three times).

---

## 0. Bottom line

1. **Free long-context capacity is narrower than it looks.** Groq's free tier has an **8K tokens/minute** cap per
   model, so a single 15k-token prompt is **rejected outright** (HTTP 413) [S5][S6]. Groq is usable only for short
   (<~7k token) tasks. Cerebras is no longer free: $5 credit, card required, expires in 30 days [S8].
   GitHub Models was **retired on 2026-07-30** [S13].
2. **Gemini 3.5 Flash-Lite is the workhorse.** It is free, has a 1M context, about 15 RPM and 500 RPD [S3][S30], and
   supports structured output and the free URL-context tool [S2][S4]. Gemini 3.8 Flash is stronger but has only
   **about 20 RPD** on the free tier (claimed, from a forum post) [S3].
3. **Gemini Search grounding is *not* free on the API for Gemini 3.x.** Free-tier pricing says "Not available", and
   the footnote reads "Can be tested in Google AI Studio". On the **paid** tier, the first **5,000 searches/month** are
   free, then $14 per 1,000 [S2]. Free grounding (500 RPD) exists only on Gemini 2.5 Flash and Flash-Lite, and 2.5 is
   limited to projects that used it before [S2][S4]. **measure:** whether the owner's ARGUS project still has 2.5 access.
4. **For the second model family**, the candidates are Mistral free (about 6 RPM and 200 RPD measured by ARGUS [S30]),
   OpenRouter `:free` (20 RPM; **50 RPD**, or 1,000 RPD after a lifetime purchase of $10 or more [S7]), and Cloudflare
   Workers AI (10k neurons/day, about 17 gpt-oss-120b calls/day, derived [S11]). OpenRouter's free roster has rotated:
   on 2026-10-07 it holds **no DeepSeek, Qwen or Llama `:free` models** [S9].
5. **The best open forecasters are only free on NVIDIA's trial API.** Kimi K3, GLM-5.3 and DeepSeek V4.1 Flash are
   all listed on NVIDIA's API catalog [S14], but its terms say "internal testing and evaluation… not in production" [S15].
   Use them for **paper-phase research only**, never for live-money decisions.
6. **Benchmarks: no free model beats the market on market questions.** On ForecastBench, simply copying the market
   price scores **71.4** on market questions, against **62.8** for the best baseline LLM [S17][S18]. On Prophet Arena,
   the top models beat the Kalshi price by ≤0.003 (1−Brier), which is inside their CIs [S19]. Gemini 3.8 Flash ranks
   well but is **poorly calibrated** (ECE 0.042, versus 0.012–0.015 for Kimi K3 and GLM-5.3) [S19]. This supports the
   pack's design: market-anchored blending plus per-family recalibration.
7. **Evidence can be had for $0** from these sources:
   - Market rules and the named resolution source;
   - Google News RSS and GDELT DOC for discovery;
   - Exa ($10 free credit/month, about 2,500 searches), Tavily (1,000 credits/month) and Brave ($5 credit/month, card required) for search;
   - Jina Reader or Gemini URL-context for full text;
   - FRED, BLS, NOAA and Wikipedia for structured facts.

   **Terms caveats:** the Google News RSS feed is licensed for "personal, non-commercial use" [S23]. AskNews's free
   plan is 25 credits/month, non-commercial [S26]. Reddit requires approval for new OAuth apps [S27].
8. **What $5–10/month buys** (derived):
   - A one-time **$10 OpenRouter top-up**: it unlocks 1,000 free RPD, and the balance pays for fallback calls.
   - Or **DeepSeek `deepseek-flash`** as the paid fallback: about $0.006 per 15k/1.5k call at peak [S16], which comes
     to about $1.5–3/month at 10–20% fallback. Running all 75 calls/day on it would cost about $14/month.
   - A $5 prepaid **Gemini paid project**, kept separate from the free one, unlocks 5,000 grounded searches/month.
     Grounded calls cost about $0.008 each on Flash-Lite [S2][S10].
9. **LLM spend is a large drag at this capital level** (derived). At $50 capital, $5/month of LLM spend means the
   strategy must return more than 10%/month just to break even on inference. **Recommendation:** $0 during paper
   trading and `PM_LLM_USD_CAP_MONTH ≤ $5` at go-live. Raise the cap only from measured forecast skill.

---

## 1. LLM providers (track A)

### 1.1 Provider matrix

| Provider | Best models usable at $0 | Context | Free limits | Card? | Tools / search | Data use & terms that matter | Stability | Role for us |
|---|---|---|---|---|---|---|---|---|
| **Google Gemini API** (AI Studio key) | `gemini-3.5-flash-lite`, `gemini-3.8-flash`; 3.7/3.6/3.5 Flash also free [S2]; Gemma 4 free [S2]; 2.5 models restricted to prior users [S4] | 1,048,576 in / 65,536 out (3.5 Flash-Lite, 3.8 Flash) [S4] | Not published since Google moved them to the AI Studio dashboard [S1][S29]. 3.5 Flash-Lite: 15 RPM, 500 RPD [S30], and the forum also says 500 RPD [S3]. 3.8 Flash: **20 RPD** (claimed [S3]). Per project, not per key; resets at midnight Pacific [S1]. TPM: **measure** | No | Structured output, function calling, thinking, **URL context (free)** [S2][S4]. **Search grounding: 3.x free = "Not available"**; 2.5 Flash/Flash-Lite free = 500 RPD shared [S2] | Free tier: content "used to improve our products", and human reviewers may read it [S2][S12]. EEA/CH/UK clause: "only Paid Services when making API Clients available to users in the EEA" [S12]; we serve no users, so low risk. Grounded results: display, storage (≤2y) and "analyze" restrictions [S12]. **Linking billing converts the project to paid; keep a separate free project** [S10] | Limits change silently; 2.5 access was withdrawn from new projects [S4] | **Primary** (forecast and parse) |
| **Groq** (free plan) | `openai/gpt-oss-120b`, `openai/gpt-oss-20b`, `qwen/qwen3.8-27b` (only LLMs in the free table) [S5] | 131k for Llama 3.3 70B [S5b]; gpt-oss: **measure** | 30 RPM; 1K RPD; **8K TPM**; 200K TPD per model [S5]. A single request above the TPM cap → **413 "Request too large"** [S6] | No | Prompt caching (cached tokens excluded from limits) [S5] | No training on customer data; ZDR toggle [S28] | ARGUS: a 15-call run got 4 answers before 429 [S30] | **Short tasks only**: rule parsing, JSON repair (≤6k tokens) |
| **Cerebras** | gpt-oss-120b, qwen-3.8-27b [S8b] | n/a in docs | **No perpetual free tier**: one-time $5 credit, valid card needed, expires 30 days after activation [S8]. Trial: 5 RPM, 1M tokens/day [S8b] | **Yes** | — | — | Changed from "1M tokens/day free" (ARGUS note) | Optional: one 30-day burst during paper phase |
| **OpenRouter `:free`** | 2026-10-07 roster (20 zero-price IDs): `nvidia/nemotron-3-ultra-550b-a55b:free` (1M ctx), `nvidia/nemotron-3-super-120b-a12b:free` (262k), `google/gemma-4-31b-it:free` (262k), `thinkingmachines/inkling:free` (1M), `dots-3-note-preview:free`, `cohere/north-mini-code:free` [S9] | per model, 65k–1M [S9] | 20 RPM; **50 RPD if <$10 ever purchased, 1,000 RPD if ≥$10** [S7]; negative balance → 402 even on free [S7]. ARGUS measured 20 RPM / 1,000 RPD [S30] | No (Yes for 1,000 RPD) | Per model | Separate "may train on data" setting for free models; opting out removes those providers [S7b] | Roster rotates; ARGUS 2026-09-24: **6 of 16** tool-capable free models answered at all [S30] | Second family (Nvidia/other); research ensemble member |
| **Mistral** (free mode) | Mistral Small 4, Medium 3.5, Large 4 (open-weight), Ministral 3; GLM 5.3 listed on platform [S31]; free-mode availability per model: **measure** | 256k (Small 4, Medium 3.5, Ministral 3) [S31b] | Not published ("Free mode has the lowest limits"; see Admin panel) [S32]. ARGUS measured **6 RPM, 200 RPD** (F-096) [S30] | No (phone verification historically) | Function calling, JSON mode | Free mode: "we may use your data (input and output) to train" [S33] | Stable for ARGUS | **Second family** (forecast sample) |
| **GitHub Models** | — | — | **Retired 2026-07-30**: inference API, playground and catalog removed [S13] | — | — | — | Dead | Do not use |
| **Cloudflare Workers AI** | `@cf/openai/gpt-oss-120b`, `@cf/meta/llama-3.3-70b-instruct-fp8-fast`, `@cf/zai-org/glm-4.7-flash`, `@cf/google/gemma-4-26b-a4b-it`, `@cf/nvidia/nemotron-3-120b-a12b` [S11]. Kimi K2.6/K2.7, GLM-5.2/5.3, DeepSeek V4 need paid billing [S11] | model-dependent | **10,000 neurons/day** free, reset at 00:00 UTC; $0.011 per 1k neurons after [S11]. At 15k/1.5k tokens (derived): gpt-oss-120b ≈ 580 neurons → **~17 calls/day**; llama-3.3-70b ≈ 14; glm-4.7-flash ≈ 72; gemma-4-26b ≈ 56 | No | — | Cloudflare terms | Neuron meter misread once in ARGUS (F-094) | Third family / overflow |
| **NVIDIA NIM** (build.nvidia.com) | Catalog (public `GET /v1/models`, 80 models): `moonshotai/kimi-k3`, `z-ai/glm-5.3`, `z-ai/glm-5.3-flash`, `deepseek-ai/deepseek-v4.1-flash`, `nvidia/nemotron-3-ultra-550b-a55b`, `google/gemma-4-31b-it`, `openai/gpt-oss-20b` [S14] | model-dependent | **40 RPM** (no increase on the free tier, per moderator) [S15b]. Credits: "1,000-credit free tier" exhausted in a day by a heavy user (claimed) [S15c] | No | — | **"Trial… only for internal testing and evaluation purposes, not in production"** [S15]. Live trading ≈ production → **terms risk** | Rate-limit complaints common [S15b] | **Paper-phase research only**: check whether Kimi K3 or GLM-5.3 beat our free roster |
| **SambaNova** | DeepSeek-V3.1/V3.2, Llama-3.3-70B, gpt-oss-120b, gemma-4-31B [S34] | — | Docs: free tier (no card) **20 RPM, 20 RPD, 200K TPD** [S34]; plans page: "add a payment method and purchase credits to run your first requests" [S34b]. **Conflicting** | Effectively yes | — | — | — | Skip |
| **Together AI** | — | — | **No free trial; $5 minimum purchase** [S35] | Yes | — | — | — | Skip |
| **Fireworks** | — | — | $1 signup credit (claimed [S36]); pricing page says "add a payment method to buy credits" [S36b] | Yes | — | — | — | Skip |
| **Cohere** | Command family | — | Trial key: 20 req/min chat, **1,000 calls/month** [S37] | No | — | Trial = evaluation, not production [S37] | — | Skip (licence) |
| **Z.ai** (direct) | `GLM-4.7-Flash`, `GLM-4.5-Flash` free [S38] | — | Free models listed at $0; limits not published [S38] | No | — | China-based provider | — | Optional extra family (measure) |
| **DeepSeek** (paid, cheap) | `deepseek-flash` (1M ctx, thinking/non-thinking), `deepseek-v4-pro` (1M ctx) [S16] | 1M / 384K out [S16] | Paid only. Flash: input miss $0.30 peak / $0.15 off-peak, cache hit $0.006/$0.003, output $1.20/$0.60 per 1M. V4-Pro: $1.32/$0.66 in, $3.96/$1.98 out [S16] | Prepay | JSON output, tool calls [S16] | Inputs stored in China; training on by default with opt-out (secondary source [S39]) | Stable API | **Paid fallback** and a third family |

**Model-ID note:** the IDs above come from live listings on 2026-10-07 (Gemini model pages, the OpenRouter
`/api/v1/models` list, the NVIDIA `/v1/models` list, and the Cloudflare and Groq tables). Free rosters rotate
monthly, so keep model IDs in config, never in code (ARGUS `config/models.yaml` pattern).

### 1.2 Capacity at our call shape (derived: 15k input + 1.5k output per call)

| Provider / model | Nominal free calls/day | Plan to use/day | Binding limit |
|---|---|---|---|
| Gemini 3.5 Flash-Lite | ≤ 500 | 40–60 | RPD 500; TPM unknown (**measure**) |
| Gemini 3.8 Flash | ~20 (claimed) | 15 | RPD |
| Gemini 3.7 / 3.6 / 3.5 Flash | unknown, separate buckets? | 0 until measured | **measure** |
| Mistral (free) | ~200 (ARGUS measured) | 20–30 | 6 RPM, RPD |
| OpenRouter `:free` | 50 (or 1,000 with $10 ever paid) | 10–30 | RPD; model availability |
| Cloudflare gpt-oss-120b | ~17 | 10 | 10k neurons/day |
| Groq gpt-oss-120b | **0 at 15k tokens** (413); ~30/day at ≤6.5k tokens | 20 short calls | 8K TPM, 200K TPD |
| Cerebras trial | ~60 (1M tok/day), 30 days only | 0 (optional) | Credit expiry |
| NVIDIA NIM | 40 RPM; credits unclear | eval only | ToS |
| **Long-context total, free** | **~800 nominal** | **~95–135** | Covers the pack's 75/day hard ceiling (`02-architecture.md` §8) with 1.3–1.8× headroom |

---

## 2. Which models forecast best (track B)

### 2.1 ForecastBench, baseline leaderboard (no tools; "Brier Index", higher is better) [S17][S18]

Parsed from `leaderboard_baseline_full.js`, 130 rows, page cached 2026-10-06.

| Rank | Model | Dataset | Market | Overall | N | Reachable at ~$0? |
|---|---|---|---|---|---|---|
| 1 | Superforecaster median | 63.8 | 72.0 | 67.6 | 578 | — |
| 2 | Public median | 59.4 | 66.0 | 62.5 | 578 | — |
| 3 | **kimi-k3-max-128k** | 61.2 | 62.6 | 61.9 | 1,198 | NVIDIA trial (`kimi-k3`) |
| 21 | **glm-5** | 60.0 | 60.6 | 60.3 | 1,114 | NVIDIA trial (glm-5.3) |
| 43 | gemini-3-flash-preview | 60.1 | 58.0 | 59.1 | 8,928 | Gemini free (successors) |
| 45 | **gemini-3.5-flash** | 60.7 | 57.4 | 59.0 | 5,127 | Gemini free |
| 45 | deepseek-r1-scratchpad | 58.9 | 59.1 | 59.0 | 6,166 | paid |
| 49 | qwen3-235b-a22b scratchpad | 58.7 | 59.1 | 58.9 | 7,541 | — |
| 67 | gemini-2.5-flash | 58.5 | 57.3 | 57.9 | 7,906 | legacy only |
| 68 | llama-3.3-70b-instruct | 55.2 | 60.3 | 57.7 | 10,736 | Cloudflare |
| 96 | **gemini-3.1-flash-lite** | 58.9 | 53.1 | 55.9 | 5,127 | Gemini free |
| 100 | deepseek-v4-pro | 57.0 | 54.1 | 55.5 | 1,700 | paid |
| 92–105 | mistral-large (2407/2411/latest) | 54.6–56.0 | 55.9–56.4 | 55.2–56.2 | — | Mistral free (old versions) |
| 41 | **Imputed Forecaster** (0.5 on dataset questions, **market price** on market questions [S18]) | 50.0 | **71.4** | 59.3 | 40,234 | free |

- **Read: copying the market beats every baseline LLM on market questions** (71.4 vs a best of 62.8, derived; the
  question sets differ, and FB adjusts for this with two-way fixed effects [S17]).
- gpt-oss, Qwen3.8 and Gemini 3.5 Flash-Lite are **not on the board**, so their skill is **unknown** until our own
  forward record exists.
- The tournament board, which allows tools, is topped by anonymised Google DeepMind entries at 69.1, against the
  superforecaster median at 68.7. gpt-5.5 with web search scores 67.3 [S17].

### 2.2 Prophet Arena (live Kalshi events; 1−Brier projected on shared events; scored 2026-10-05/07) [S19]

| Model | Projected 1−Brier | Edge vs matched market | Avg return (sim.) | **ECE** | Resolved events |
|---|---|---|---|---|---|
| Claude Fable 5.1 (top) | 0.9488 ±0.0028 | +0.062 | 97.9% | 0.023 | 1,326 |
| **GLM-5.3** | 0.9483 ±0.0038 | +0.065 | 98.8% | **0.015** | 1,543 |
| DeepSeek V4 Pro 0813 | 0.9475 ±0.0058 | +0.064 | 102.6% | 0.015 | 1,481 |
| **Gemini 3.8 Flash** | 0.9472 ±0.0061 | +0.083 | 104.7% (±17.7) | **0.042** | 232 |
| **Kimi K3** | 0.9469 ±0.0032 | +0.046 | 92.0% | **0.012** | 2,343 |
| **DeepSeek V4.1 Flash** | 0.9462 ±0.0086 | +0.073 | 122.8% (±60.1) | 0.037 | 232 |
| **Kalshi Markets** | **0.9458** | — | — | — | all |

- The margins over the market are 0.000–0.003, inside the CIs. Simulated returns sit around 100% (break-even) with
  CIs of ±6–60 points. **No reliable money edge is visible even for frontier models.**
- Calibration (ECE) differs about 3× across models that rank similarly. **Recalibrate per model family**, and do not
  average raw probabilities across families.

### 2.3 Metaculus FutureEval / AI Benchmark, Spring 2026 (published 2026-09-09) [S20][S21]

| Finding | Number |
|---|---|
| Bot team vs Pros (head-to-head, 99 questions) | −1.25 (95% CI −4.87 to 2.37), not significant; Q2 2025 was −20.03 [S20] |
| Best Metaculus single-model bot | `gpt-5.1-high + asknews` 11.32; Claude 4.5 Sonnet-high 8.94; GPT-5.2-high 8.62; o3 8.46 [S20] |
| Cheap or open models | DeepSeek-3.2-reasoning 7.22 (5th among Metaculus bots); no budget model in the top 10 [S20] |
| High-reasoning vs standard variant | Wins 8 of 8 (p = 0.004) [S20] |
| Search provider | "No search provider is clearly better": Perplexity 3.47 vs AskNews 3.23 on DeepSeek bots, CIs overlap [S20]. Winners used 1.75 sources on average vs 1.00 [S21] |
| Ensembles | 86% of Fall-2025 winners ensembled; ~28 LLM calls/question for winners vs ~7 [S21]. Bot-team aggregation best at 2–10 members [S20] |
| Recalibration | Logistic (Platt) recalibration improved Brier by 0.016 on binary questions (p < 0.001); capping extremes correlated with winning (r = +0.48) [S21] |

### 2.4 Implications for the design

- **Free-reachable models sit ~9–18 Brier-index points below the market on market questions**: kimi-k3 62.6, gemini-3.5-flash 57.4 and gemini-3.1-flash-lite 53.1, against 71.4 for copying the market (§2.1, derived). The pack's rule therefore
  holds: q equals the market price until our own calibration says otherwise (`02-architecture.md` C4).
- **Prefer the strongest model on fewer markets** over many weak samples. Thinking or "high" modes beat standard ones
  [S20], so use Gemini 3.8 Flash with high thinking for one sample per market, inside its ~20 RPD.
- **Recalibrate per family** (Platt or isotonic on our resolved forward record) and cap extremes at [0.03, 0.97]
  [S21]. Gemini 3.8 Flash's ECE of 0.042 makes this mandatory [S19].
- **Use 2–3 evidence sources, not one premium source** [S20][S21]. A free search stack is therefore not a
  handicap by itself.

---

## 3. Free evidence and news sources (track C)

| Source | What it gives | Key? | Free limits | Terms that matter | Role |
|---|---|---|---|---|---|
| **Polymarket Gamma** (market `description`, `resolutionSource`) | Rules and the named resolution source | No | Gamma general 4,000 req/10s; `/events` 500/10s; `/markets` 300/10s [S25] | Public | **Tier 0**, always first |
| **Polymarket comments** `gamma-api…/comments?parent_entity_type=Event&parent_entity_id=…` | User comments (probe returned a list for event 30829 on 2026-10-07) | No | 200 req/10s [S25] | Untrusted and manipulable; **prompt-injection vector** | Optional signal; never evidence |
| **Gemini Search grounding** | Google results synthesised with citations | Gemini key | Free tier: **not available on 3.x** (AI Studio UI only); 2.5 Flash/Flash-Lite 500 RPD free, 2.5 restricted to prior users. Paid project: 5,000 searches/month free on 3.x, then $14/1k. One prompt can trigger several billable queries [S2][S4] | Show search suggestions; store ≤2 years; no cache, resell or "analyze" of results [S12] | Paid-project option (§4.4) |
| **Gemini URL context** | Model fetches given URLs | Gemini key | **Free** on the free tier (as tokens) [S2] | Gemini terms | Full-text read inside Flash-Lite |
| **Google News RSS** `news.google.com/rss/search?q=…+when:7d` | Headlines, publisher, date | No | Unpublished | Feed states: "solely for… a personal feed reader for **personal, non-commercial use**. Any other use… expressly prohibited" (live probe [S23]) | Discovery, low rate, cached. **Terms risk**, so keep it replaceable |
| **GDELT DOC 2.0** `api.gdeltproject.org/api/v2/doc/doc` | Article lists across 65 languages, tone, timelines; rolling 3-month window; ≤250 records/query [S22] | No | **1 request per 5 s** (live response text [S22b]) | Open | Discovery and non-English coverage |
| **Exa** | Neural web search and contents | Key | **$10 credit/month (≈2,500 Instant searches)**, resets monthly, no card [S24a] | Standard | **Primary search API** |
| **Tavily** | Search (basic 1 credit, advanced 2) and extract (1 credit/5 URLs) | Key | **1,000 credits/month**, no card; PAYG $0.008/credit [S24b][S24c] | Standard | Secondary search |
| **Brave Search API** | Independent web index | Key | **$5 credit/month** (≈1,000 requests at $5/1k); **card required** [S24d] | Storing results or using them "to train or tune an LLM" needs a plan with storage rights [S24d] | Tertiary search |
| **Serper** (Google SERP) | Web/News/Scholar SERP | Key | **2,500 free queries** one-time, no card [S24e]; paid price unverified | — | Reserve |
| **Jina Reader / Search** `r.jina.ai`, `s.jina.ai` | URL → clean text; search | Reader no; search yes | Reader 20 RPM with no key, 500 RPM with a free key; Search 100 RPM with a key; **10M free tokens per new key**; a search costs ≥10k tokens [S24f] | — | Article text extraction |
| **AskNews** | Licensed news API built for forecasting | Key | PAYG **$0/month, 25 credits**; Pro $7.99 for 500 credits; `/news` (48h) = 1 credit; 1 req/2s [S26] | **Non-commercial licence** on all self-serve plans [S26]. Metaculus AIB bots get sponsored access, renewed each season [S21b] | Only via AIB participation; not a base dependency |
| **Reddit Data API** | Subreddit discussion | OAuth | 100 QPM (approved clients) [S27] | **New OAuth tokens require approval** (Responsible Builder Policy); free = non-commercial [S27] | Skip |
| **Wikipedia / Wikidata** | Background, entities, office-holders | No | 10 req/min unidentified; **200 req/min with a contact User-Agent**; 2,000 authenticated [S40] | CC BY-SA | Background facts |
| **FRED** (+ ALFRED vintages) | Macro series | Free key | 120 req/min (per ARGUS `DATA_SOURCES.md`; FRED docs [S41]) | Attribution | Econ/rates markets |
| **BLS API v2** | CPI, jobs | Free registration | **500 queries/day**, 50 series/query, 20 years; v1 25/day; 50 req/10s [S42] | Public domain | CPI/NFP markets |
| **NOAA CDO v2** / NWS `api.weather.gov` | Station observations / forecasts | CDO token / NWS none (User-Agent) | CDO **5 req/s, 10,000/day** [S43] | Public domain | Weather markets (match the station in the rules) |
| **Election commissions, official registries** | Results and certification | Varies | Varies | Varies | Map per market from `resolutionSource` |

---

## 4. Recommendation (track D)

### 4.1 Router configuration (free-first, ≥2 families, every limit enforced in code)

```yaml
# config/models.yaml (proposal). IDs verified 2026-10-07; limits marked * must be measured on our keys.
families: [google, mistral, nvidia-open, openai-open, deepseek]   # consensus needs ≥2 distinct
providers:
  gemini_free:   {key_env: GEMINI_API_KEY_FREE, project: free-only, rpd: {gemini-3.8-flash: 20*, gemini-3.5-flash-lite: 500*}, rpm: 15*}
  gemini_paid:   {key_env: GEMINI_API_KEY_PAID, prepay_usd: 5, grounding_free_month: 5000, enabled: false}   # separate project [S10]
  mistral_free:  {rpm: 6*, rpd: 200*}
  openrouter:    {rpm: 20, rpd: 50, rpd_if_topped_up: 1000}
  cloudflare:    {neurons_day: 10000}
  groq_free:     {tpm: 8000, tpd: 200000, rpd: 1000, max_prompt_tokens: 6500}   # 413 above TPM [S6]
  nvidia_trial:  {rpm: 40, use: paper_eval_only}                                    # ToS [S15]
  deepseek_paid: {usd_cap_month: 5, prefer_offpeak: true}
roles:
  parse_rules:        # rules → structured question + ambiguity flags, ≤6k tokens
    chain: [groq_free:openai/gpt-oss-120b, gemini_free:gemini-3.5-flash-lite, mistral_free:mistral-small-latest]
  forecast_sample_A:  # family google, strongest free sample, thinking=high
    chain: [gemini_free:gemini-3.8-flash, gemini_free:gemini-3.7-flash*, gemini_free:gemini-3.5-flash-lite]
  forecast_sample_B:  # family google-lite (cheap second view)
    chain: [gemini_free:gemini-3.5-flash-lite]
  forecast_sample_C:  # must be a NON-google family
    chain: [mistral_free:mistral-large-latest*, openrouter:nvidia/nemotron-3-ultra-550b-a55b:free,
            cloudflare:@cf/openai/gpt-oss-120b, deepseek_paid:deepseek-flash]
  research_ensemble:  # paper phase only; logged, never traded on
    chain: [nvidia_trial:moonshotai/kimi-k3, nvidia_trial:z-ai/glm-5.3, nvidia_trial:deepseek-ai/deepseek-v4.1-flash]
  json_repair:
    chain: [groq_free:openai/gpt-oss-20b, gemini_free:gemini-3.5-flash-lite]
rules:
  - skip a provider at 90% of RPD/TPD; circuit-break after 3 consecutive 429/5xx
  - sample C never shares a family with A or B; if no non-google family is reachable → NO_FORECAST (not a google-only forecast)
  - per-family calibration map; blend in code; raw p never averaged across families
  - paid spend counter enforced before each call; on cap → stop forecasting, keep managing positions, alert once
```

### 4.2 Evidence-retrieval stack (per shortlisted market)

1. **Tier 0 (always):** Gamma rules, `resolutionSource`, end date, and CLOB mid/spread. The LLM never sees the price in pass 1.
2. **Discovery (free, no key):** Google News RSS (`when:7d`, ≤1 request/market) and GDELT DOC artlist (≤1 per 5 s, 25 records).
   Then deduplicate by URL and title, and sort by date.
3. **Search (quota-rotated):** 2–3 queries per market across Exa → Tavily → Brave. At 15 markets × 3 queries = 45/day ≈
   1,350/month (derived), which is inside the combined ≈ 4,500/month free (2,500 + 1,000 + ~1,000).
4. **Read:** the top 3–5 URLs through Jina Reader (keyed). The fallback is the Gemini URL-context tool inside the
   Flash-Lite call. Trim to ≤12k tokens of evidence, inside `<evidence id=…>` blocks (untrusted).
5. **Structured facts by category:** FRED, BLS, NOAA/NWS, Wikipedia/Wikidata and official result pages. Each gets
   a resolver that maps from `resolutionSource`.
6. **Cache everything** by (query, day). Re-retrieve only when the re-forecast trigger fires (price move, TTL, new
   evidence), per pack §8.

### 4.3 Expected daily capacity (derived)

| Item | Budget/day | Free capacity used |
|---|---|---|
| Markets forecast | ≤ 15 | — |
| Forecast calls (3 samples + ≤1 repair) | ≤ 60 | A: 15 on 3.8 Flash (~20 RPD); B: 15 on Flash-Lite; C: 15 on Mistral/OpenRouter/Cloudflare; repair: Groq |
| Parse calls | ≤ 15 | Groq (≤6.5k tokens) |
| Search queries | ≤ 45 | Exa/Tavily/Brave (≈150/day pooled) |
| Page reads | ≤ 75 | Jina (10M tokens ≈ 2–3 months at ~5k tokens/page, derived) |

### 4.4 Monthly cost scenarios (derived from [S2][S7][S16])

| Scenario | What changes | $/month | Notes |
|---|---|---|---|
| **S0: free only** (paper phase) | — | **0** | Capacity ~95–135 long calls/day; fragile to roster rotation |
| **S1: + $10 OpenRouter once** | 1,000 free RPD; balance pays fallback | ~0 for the free RPD, + ~$1.5–3 fallback | Fallback at 10–20% of 75 calls on a DeepSeek-class model ≈ $0.0063/call peak |
| **S2: + DeepSeek direct** as fallback | `deepseek-flash`, off-peak preferred | **$1.5–3** (10–20% fallback); $14 if all 75/day; K=3 sharing a 15k prefix ≈ $0.010/market → $4.5 for 15 markets/day | Cache hits $0.006/M make repeated-prefix samples cheap [S16] |
| **S3: + Gemini paid project** for grounding | 5,000 searches/month free; Flash-Lite tokens $0.30/$2.50 | **≈ $3.7** for 15 grounded calls/day ($0.0083/call); 3.8 Flash $0.017/call, **doubling on 2027-01-01** | $5 minimum prepay; keep the free project separate [S10] |
| **Recommended at go-live** | S1 or S2, cap $5 | **≤ $5** | Matches the pack's suggestion of ≤10% of capital; revisit when capital reaches $100–150 |

### 4.5 Week-1 measurements (need owner keys; cost $0)

1. Per-model free RPD/RPM/TPM on the owner's Gemini project, including whether `gemini-2.5-flash` is still
   reachable (free grounding, 500 RPD) and whether 3.5/3.6/3.7 Flash have separate buckets.
2. Mistral free mode: which of Large 4, Medium 3.5 and GLM 5.3 answer, plus RPM/RPD.
3. OpenRouter: which `:free` models return schema-valid JSON on the forecaster prompt (ARGUS found 6 of 16).
4. Groq: confirm the 413 at >8k tokens and size the parse prompt under 6.5k.
5. NVIDIA trial: how fast credits are consumed per call, to decide whether the research ensemble fits the paper phase.

---

## 5. Questions for the owner

1. **Paid fallback:** allow ≤$5/month at go-live (S1/S2), or stay strictly $0 and accept `NO_FORECAST` days? Should
   inference cost count against the $50 bankroll in PnL, as the pack says?
2. **NVIDIA trial models in paper phase:** OK to use them for research only, knowing they cannot be used for live
   decisions under their terms?
3. **Google News RSS terms:** keep it as a discovery source (the common practice), or drop it for GDELT plus the
   search APIs only?
4. **Brave card-on-file:** acceptable, or skip Brave?
5. **Metaculus AI Benchmark:** enter a bot? It is free sponsored LLM and AskNews access, plus an external Brier
   scorecard for our forecaster, at the cost of extra work each season.

---

## 6. Sources (all accessed 2026-10-07 unless noted)

| ID | Source |
|---|---|
| S1 | Gemini API rate limits (page updated 2026-09-02): https://ai.google.dev/gemini-api/docs/rate-limits |
| S2 | Gemini API pricing (updated 2026-10-07; raw `pricing.md.txt`): https://ai.google.dev/gemini-api/docs/pricing |
| S3 | Google AI forum, "Gemini 3.8 Flash Free Tier 20 RPD…" (2026-09-03, user report): https://discuss.ai.google.dev/t/gemini-3-8-flash-free-tier-20-rpd-is-too-limited-for-practical-evaluation/180609 |
| S4 | Gemini models (2.5 restricted to prior users; 3.5 Flash-Lite and 3.8 Flash specs): https://ai.google.dev/gemini-api/docs/models , …/models/gemini-3.5-flash-lite , …/models/gemini-3.8-flash |
| S5 | Groq rate limits: https://console.groq.com/docs/rate-limits ; S5b Groq models: https://console.groq.com/docs/models |
| S6 | Groq community, 413 on free tier: https://community.groq.com/t/gpt-oss-20b-free-tier-getting-413-errors/1411 |
| S7 | OpenRouter limits: https://openrouter.ai/docs/api-reference/limits ; S7b provider logging: https://openrouter.ai/docs/guides/privacy/provider-logging |
| S8 | Cerebras pricing (free trial terms): https://www.cerebras.ai/pricing ; S8b Cerebras rate limits: https://inference-docs.cerebras.ai/support/rate-limits |
| S9 | OpenRouter public model list (`GET /api/v1/models`, probed 2026-10-07): https://openrouter.ai/api/v1/models |
| S10 | Gemini billing ($5 min prepay; billing converts project): https://ai.google.dev/gemini-api/docs/billing |
| S11 | Cloudflare Workers AI pricing: https://developers.cloudflare.com/workers-ai/platform/pricing/ |
| S12 | Gemini API Additional Terms (effective 2026-03-23): https://ai.google.dev/gemini-api/terms ; regions: https://ai.google.dev/gemini-api/docs/available-regions |
| S13 | GitHub changelog, "GitHub Models is now retired" (2026-07-30): https://github.blog/changelog/2026-07-30-github-models-is-now-retired/ |
| S14 | NVIDIA API catalog (`GET https://integrate.api.nvidia.com/v1/models`, probed 2026-10-07) |
| S15 | NVIDIA API Trial Terms of Service: https://assets.ngc.nvidia.com/products/api-catalog/legal/NVIDIA%20API%20Trial%20Terms%20of%20Service.pdf ; S15b forum "Is NIM free" (2026-05-08): https://forums.developer.nvidia.com/t/is-nim-free/369368 ; S15c forum (2026-09-10, claimed): https://forums.developer.nvidia.com/t/rate-limit-credits-increase-request-40-200-rpm-1-000-5-000-credits-sustained-agentic-workload-on-kimi-k3/382895 |
| S16 | DeepSeek pricing: https://api-docs.deepseek.com/quick_start/pricing |
| S17 | ForecastBench leaderboards (cached 2026-10-06): https://www.forecastbench.org/leaderboards/ (data: `/assets/js/leaderboard_baseline_full.js`) |
| S18 | ForecastBench changelog (Imputed Forecaster definition): https://github.com/forecastingresearch/forecastbench/wiki/Changelog |
| S19 | Prophet Arena leaderboard (last scored 2026-10-05; rows dated 2026-10-06→07): https://www.prophetarena.co/leaderboard |
| S20 | Metaculus FutureEval Spring 2026 results (2026-09-09): https://www.lesswrong.com/posts/wZBbDqzfBjYG58CxK/futureeval-spring-results-pros-beat-bots-but-the-gap-is |
| S21 | "AI forecasting in 2026: what 11 analyses say" (secondary synthesis): https://www.lesswrong.com/posts/a82q6yd8zKpYk56cF/ai-forecasting-in-2026-what-11-analyses-say ; S21b Spring 2026 AIB announcement (2025-12-18): https://www.lesswrong.com/posts/dvGYXZfiqCgcck9im/announcing-spring-2026-ai-forecasting-benchmark |
| S22 | GDELT DOC 2.0 API: https://blog.gdeltproject.org/gdelt-doc-2-0-api-debuts/ ; S22b live API response "Please limit requests to one every 5 seconds" (probed 2026-10-07) |
| S23 | Google News RSS feed `<copyright>` element (probed 2026-10-07): https://news.google.com/rss/search?q=ceasefire+when:1d&hl=en-US&gl=US&ceid=US:en |
| S24 | a Exa pricing: https://exa.ai/pricing ; b Tavily pricing: https://www.tavily.com/pricing ; c Tavily credits: https://docs.tavily.com/documentation/api-credits ; d Brave Search API: https://brave.com/search/api/ ; e Serper: https://serper.dev/ ; f Jina Reader: https://jina.ai/reader/ |
| S25 | Polymarket API rate limits: https://docs.polymarket.com/quickstart/introduction/rate-limits ; comments probe `gamma-api.polymarket.com/comments` (2026-10-07) |
| S26 | AskNews pricing and rate limits: https://docs.asknews.app/en/rate-limiting |
| S27 | Reddit Responsible Builder Policy (r/redditdev): https://www.reddit.com/r/redditdev/comments/1oug31u/introducing_the_responsible_builder_policy_new/ ; rate-limit summary (secondary): https://www.redditapis.com/blogs/reddit-data-api-2026 |
| S28 | Groq "Your Data in GroqCloud": https://console.groq.com/docs/your-data |
| S29 | yingtu.ai, Gemini free-tier limits (2026-09-30, secondary): https://yingtu.ai/en/blog/google-gemini-api-free-tier-limits-2026 |
| S30 | ARGUS internal measurements, 2026-09-24: `[local source path omitted]` §5.3b; `LLM_PROVIDERS_AND_MODELS.md` §1 |
| S31 | Mistral models: https://docs.mistral.ai/getting-started/models/ ; S31b known limitations (context): https://docs.mistral.ai/resources/known-limitations |
| S32 | Mistral rate limits help: https://help.mistral.ai/en/articles/698531-why-am-i-hitting-api-rate-limits-and-how-do-i-increase-them |
| S33 | Mistral training on free-mode data: https://help.mistral.ai/en/articles/347617-do-you-use-my-user-data-to-train-your-artificial-intelligence-models |
| S34 | SambaNova rate limits: https://docs.sambanova.ai/docs/en/models/rate-limits ; S34b plans: https://cloud.sambanova.ai/plans |
| S35 | Together AI free-tier change: https://support.together.ai/articles/1862638756-changes-to-free-tier-and-billing-july-2025 |
| S36 | Fireworks $1 credit (secondary, claimed): https://www.getaiperks.com/en/blogs/27-ai-api-free-tier-credits-2026 ; S36b https://fireworks.ai/pricing |
| S37 | Cohere rate limits: https://docs.cohere.com/docs/rate-limits |
| S38 | Z.ai pricing: https://docs.z.ai/guides/overview/pricing |
| S39 | DeepSeek privacy summary (secondary): https://meetily.ai/llm-privacy/deepseek |
| S40 | Wikimedia API rate limits (page edited 2026-06-03): https://www.mediawiki.org/wiki/Wikimedia_APIs/Rate_limits |
| S41 | FRED API docs: https://fred.stlouisfed.org/docs/api/fred/ (rate figure via ARGUS `docs/DATA_SOURCES.md`) |
| S42 | BLS API FAQ: https://www.bls.gov/developers/api_faqs.htm |
| S43 | NOAA CDO web services v2: https://www.ncdc.noaa.gov/cdo-web/webservices/v2 |

## Verification (2026-10-07)

Adversarial re-check of 10 load-bearing claims, re-run from scratch on 2026-10-07; this replaces the earlier pass. Every primary source was re-downloaded live, and at least one independent source was tried for each claim. Verdicts: confirmed / partly / unverifiable / refuted. Nothing was refuted, so no inline markers were added. All URLs were accessed 2026-10-07.

| # | Claim | Verdict | Note | Better / independent source |
|---|---|---|---|---|
| 1 | Groq free plan: gpt-oss-120b, gpt-oss-20b and qwen3.8-27b each have 30 RPM, 1K RPD, 8K TPM and 200K TPD. A single request above the TPM cap returns 413 "Request too large", so 15k-token prompts cannot run. | confirmed | The page's `freeRows` data matches exactly for all three models. `devRows` (Developer plan) gives 1K RPM, 500K RPD and 250K TPM. The 413-on-TPM behaviour is **not documented** by Groq: the rate-limits page says breaches return 429, and the errors page defines 413 only as "request body is too large". Users do observe it. One report reads "413 Request too large … (TPM): Limit 8000, Requested 33128" for gpt-oss-20b; another reads "Request too large … (TPM): Limit 6000, Requested 6294". Cached tokens are excluded from limits, but the first, uncached 15k call still exceeds 8K (derived). **[S6] is dead**: it returns a 301 redirect to groq.com. | https://github.com/elizaOS/eliza/issues/4040 ; https://console.groq.com/docs/errors ; https://console.groq.com/docs/rate-limits.md |
| 2 | On the free tier, Gemini 3.x Search grounding is "Not available" ("Can be tested in Google AI Studio"). Paid tier: 5,000 free searches/month shared across 3.x, then $14 per 1,000. Free 500 RPD grounding exists only on 2.5 Flash/Flash-Lite. | partly | `pricing.md.txt` confirms free = "Not available" for 3.8, 3.7, 3.6 and 3.5 Flash, 3.5 and 3.1 Flash-Lite, 3 Flash Preview and 3.1 Pro Preview. Paid is 5,000/mo shared, then $14/1k. 2.5 Flash and Flash-Lite are free up to 500 RPD with a shared limit; 2.5 Pro free = Not available. **Off on wording:** the "Gemini 3.8 Live / 3.1 Flash Live Preview" row shows free-tier grounding as "**Supported**", so "only 2.5" is not literally true. Those are audio Live API models, irrelevant to us. The "Can be tested in AI Studio" footnote is absent from the 3.8 and 3.7 Flash **Standard** rows. Combined with claim 3, a new project has no free Search grounding on any text model (derived). | https://ai.google.dev/gemini-api/docs/pricing.md.txt ; https://www.80aj.com/2025/12/18/gemini-3-flash-free-version-disables-google-search-function-sparking-user-concern-en/ (secondary) |
| 3 | Gemini 2.5 models are restricted to users who actively used them in the past. New projects are steered to 3.5 Flash-Lite or 3.8 Flash, which take 1,048,576 input tokens. | confirmed | The models page carries the note verbatim on the 2.5 Flash, Flash-Lite and Pro sections: "limiting access to the 2.5 models to users who have actively used them in the past… For any new projects, use our latest models: 3.5 Flash-Lite or 3.8 Flash." The model pages list input 1,048,576 and output 65,536 tokens for both. Independently, from 2026-07-09 new projects got 404 "models/gemini-2.5-flash is no longer available", and forum threads report it persisting into September. | https://discuss.ai.google.dev/t/gemini-2-5-flash-and-gemini-2-5-flash-lite-returning-404-no-longer-available-today-july-9-contradicts-oct-16-2026-shutdown-date/174267 ; https://ai.google.dev/gemini-api/docs/models/gemini-3.8-flash.md.txt |
| 4 | OpenRouter `:free` models: 20 RPM; 50 requests/day if under $10 of credits ever purchased, 1,000/day at $10 or more. | confirmed | The page constants are `FREE_MODEL_RATE_LIMIT_RPM=20`, `FREE_MODEL_NO_CREDITS_RPD=50`, `FREE_MODEL_HAS_CREDITS_RPD=1e3` and `FREE_MODEL_CREDITS_THRESHOLD=10`, and the FAQ uses the same constants. Nuance: "the higher daily ceiling is granted starting one credit below the table's threshold (currently 9 credits)". The counter resets per UTC day and is visible as `free_model_daily_requests` in `GET /api/v1/key`. A third-party guide (Sep 2026) gives the same 20 / 50 / 1,000 figures. | https://openrouter.ai/docs/api-reference/limits.md ; https://openrouter.ai/docs/faq ; https://pinggy.io/blog/free_ai_model_apis_unlimited_tokens_openrouter/ (secondary) |
| 5 | GitHub Models was fully retired on 2026-07-30: inference API, playground and catalog. | confirmed | The changelog says: "As of July 30, 2026… The playground, model catalog, inference API, and bring your own key (BYOK) are no longer available to any customer, including existing customers with active usage." A 2026-07-01 notice pre-announced it, with brownouts on Jul 16 and Jul 23. | https://github.blog/changelog/2026-07-01-github-models-is-being-fully-retired-on-july-30-2026 ; https://www.developersdigest.tech/blog/github-models-retired-2026 (secondary) |
| 6 | Cerebras no longer has a perpetual free tier. It gives a one-time $5 promotional credit, requires a valid payment method, and the credit expires 30 days after activation. | confirmed | The pricing page says: "Add a valid payment method to receive a one-time $5 promotional credit. The credit expires 30 days after activation". It also says the trial is "not intended for production use". The rate-limits FAQ says: "Is there a permanently free tier? No… doesn't currently offer a no-cost tier that renews automatically." Free Trial limits for gpt-oss-120b: 5 RPM, 30K uncached TPM, 90K total TPM, 1M TPH and 1M TPD. A third-party tracker dates the "Free" → "Free Trial" switch to late May–Jul 2026. | https://inference-docs.cerebras.ai/support/rate-limits ; https://usagepricing.com/blueprint/activity/cerebras-2026-07-21-packaging (secondary) |
| 7 | NVIDIA API Trial terms: the API Service may be used only for internal testing and evaluation, not in production, unless a subscription is purchased. | confirmed | The PDF is dated "v. September 19, 2025". §1.4 says: "Unless you purchase a Subscription… you may only use the API Service for internal testing and evaluation purposes, not in production." §1.2 adds "limited trial purposes only and without use of the API Service or Generated Content in production". NVIDIA's NIM FAQ independently says free endpoints are for prototyping, development and testing. | https://docs.api.nvidia.com/nim/docs/faq ; https://forums.developer.nvidia.com/t/nvidia-nim-faq/300317 |
| 8 | ForecastBench baseline: on market questions the Imputed Forecaster (market price) scores 71.4, against at most 62.8 for any baseline LLM. Overall: kimi-k3-max 61.9, gemini-3.5-flash 59.0, superforecaster median 67.6. | confirmed | Re-parsed live `leaderboard_baseline_full.js` (130 rows). Imputed: market 71.4 (N_market = 3,662), overall 59.3. The best LLM on market questions is o3-2025-04-16-scratchpad at 62.8, but on **only 93 market questions**, so the comparison is not like-for-like. kimi-k3-max-128k scores 62.6 (N = 214); its overall is 61.9. gemini-3.5-flash is 59.0 overall (market 57.4). The superforecaster median is 67.6 overall and 72.0 on market (N = 57). The Naive Forecaster scores 71.3 on market questions and also beats every LLM. The changelog defines Imputed as 0.5 on dataset questions and the market value on market questions. | https://github.com/forecastingresearch/forecastbench/wiki/Changelog |
| 9 | Prophet Arena (scored 2026-10-05 to 07): Kalshi Markets 0.9458 projected 1−Brier; GLM-5.3 0.9483 (ECE 0.015); Gemini 3.8 Flash 0.9472 (ECE 0.042, 232 events); Kimi K3 0.9469 (ECE 0.012); DeepSeek V4.1 Flash 0.9462 (ECE 0.037). | confirmed | Live HTML shows "Last scored Oct 5" with rows "Oct 6 → Oct 7". Kalshi Markets: 0.9458 (rank 18). GLM-5.3: 0.9483 ±0.0038 (rank 4), ECE 0.0149, 1,543/2,626. Gemini 3.8 Flash: 0.9472, ECE 0.0424, 232/625 resolved/predicted. Kimi K3: 0.9469, ECE 0.0116. DeepSeek V4.1 Flash: 0.9462, ECE 0.0372, return 122.80% ±60.11. The GLM-5.3 model page (JS-rendered) independently shows edge +0.0651, return 98.81% and 1,543/2,626. No third-party source carries these numbers, so the check is same-site only. | https://www.prophetarena.co/model/glm-5.3 (same publisher, separate view) |
| 10 | DeepSeek `deepseek-flash`: 1M context. Per 1M tokens, peak / off-peak: input cache miss $0.30 / $0.15, cache hit $0.006 / $0.003, output $1.20 / $0.60. | confirmed | The live page matches exactly; max output is 384K and the concurrency limit is 2,500. The page itself lists MODEL VERSION = **DeepSeek-V4.1-Flash**, so the alias is now verified. Legacy `deepseek-v4-flash` names are retired and served by V4.1 Flash. Peak hours are 01:00–04:00 and 06:00–10:00 UTC, Mon–Fri (excluding Chinese public holidays); every other hour is off-peak at half price. TechNode independently gives off-peak prices in RMB: 0.02 per 1M for cache hit, 1 for cache miss and 4 for output, with peak at 2×. At about 7.1 RMB/USD that is ≈ $0.003, $0.14 and $0.56, consistent with the USD page (derived). Side conflict, **unverified**: TechNode says V4-Pro requests have been routed to V4.1 Flash since Sep 14, but the live pricing page still lists `deepseek-v4-pro` (V4-Pro-0813) at $1.32 in / $3.96 out peak. This affects the report's V4-Pro figures, not this claim. | https://technode.com/2026/09/10/deepseek-formally-launches-v4-1-flash-routes-v4-pro-requests-to-flash/ ; https://www.yottalabs.ai/post/deepseek-v4-1-flash-pricing-specs-v4-pro-routing-2026 (secondary) |
