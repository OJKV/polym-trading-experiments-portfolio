# 02 — LLM forecasting agents: what exists, what they achieved

All sources accessed **2026-10-07**. Labels: **derived** = computed by us from cited numbers;
**claimed** = vendor/self-reported, not independently verifiable; **sim** = simulated or backtest P&L;
**real** = real-money P&L. Source IDs `[Rn]` resolve in §7. Nothing here was run, signed into or cloned;
GitHub code was read through raw files and the public API only.

## TL;DR

1. **No independently verified evidence that an LLM forecasting agent makes money after fees on a real-money market.** Every real-money record we found is flat or negative over its lifetime. Prediction Arena: 6 frontier models lost 16–31% on Kalshi in 57 days, and averaged −1.1% on Polymarket [R25]. Olas Polystrat (about 138 agents) shows a lifetime trading ROI of −7% [R5, claimed]. ryanfrigo's Kalshi bot, n=55 settled trades, reports "NO MEASURED EDGE" [R22].
2. **An LLM on its own loses to a liquid market price.** AIA scores Brier 0.1258 against the market's 0.1106 [R21]. Halawi et al. score 0.179 against the crowd's 0.149 [R35]. On ForecastBench market questions, LLMs that do not see the price score Brier Index 60–63, while copying the price scores 71.3 [R10, derived].
3. **An LLM combined with the market beats the market, but only slightly.** AIA's regression blend scores 0.106 against 0.1106, with the LLM getting about ⅓ of the weight on liquid markets [R21]. Halawi's blend with the crowd weighted 4× scores 0.146 against 0.149 [R35]. This is the only consistent positive signal, and it is our P1 thesis.
4. **Do not show the LLM the price and let it adjust; blend in code instead.** On ForecastBench, 442 of 527 entries that saw the market price scored *below* simply copying it [R11, secondary analysis]. Schoenegger et al. found that feeding the human median into the LLM was worse than averaging the two afterwards [R36].
5. **What the forecasting tournaments say works:** frontier reasoning models, agentic multi-pass search over at least 2 distinct sources, at least 5 samples across model families aggregated by the median, Platt recalibration, and clipping extremes [R12, R14–R18, R21]. These gains are measured in Brier points, not dollars.
6. **The belief-to-trade layer matters as much as the forecaster.** The same forecasts made −55.5% ROI with edge-proportional sizing and +15.9% with selection plus quarter-Kelly plus risk limits (backtest, 42 trades) [R27]. Olas sizes with execution-aware Kelly, walking the book and including fees, with a `min_edge`/`max_edge` band [R6].
7. **Our current design is under-powered on two points.** K = 3 samples is below the evidence, which shows a sharp gain from 1 to 5 samples [R21]. A minimum of 60 resolved forecasts cannot detect a Brier edge of 0.02, which needs about 350 [R33]. See §5a.

---

## 1. Scorecard

| # | System | Kind | Venue | Money | Headline outcome | Verdict | Src |
|---|---|---|---|---|---|---|---|
| 1 | Polymarket/agents | OSS framework | Polymarket | none published | No P&L published. Archived, last push 2024-11-05. The execution line is commented out | Reference only; anti-pattern source | R1 |
| 2 | Gnosis prediction-market-agent (+ tooling) | OSS agents, 60+ variants | Omen/Presagio (+ Manifold, Polymarket, Metaculus) | real (small, $1–5 bets) | No public P&L found. Code comments flag 4 agents: "Even after optimizing, this doesn't seem to get profitable" | Good tooling, no proven edge | R2, R3 |
| 3 | Olas Omenstrat / **Polystrat** (valory-xyz/trader + mech tools) | Production agent fleet | Omen (1,208 agents); **Polymarket (138 agents)** | real | Polystrat trading ROI: **−7% lifetime** (since 2026-01-16); +10% over 30d; +3% over 90d. Omenstrat trading ROI: **−5% lifetime**, but **+69% total including OLAS staking subsidies** | Best public real-money dataset; lifetime negative before subsidies | R5 (claimed) |
| 4 | Prophet Arena | Benchmark (Kalshi events, models see the price) | Kalshi (76% sports) | sim | Paper: no model broke even (GPT-5 avg return 0.943). Live board 2026-10-05: returns 89.9%–131.8%, where 100% is break-even. Fees not stated | Sim only; sports-heavy | R8, R9 |
| 5 | ForecastBench | Benchmark | Mixed questions (market + dataset) | none | Top entries tie superforecasters (Brier Index 69.0 vs 68.6). On market questions the best entry scores 76.0, against **71.3 for copying the price** | LLMs need the price to compete on market questions | R10, R11 |
| 6 | Metaculus AIB / FutureEval | Tournament | Metaculus (no money) | none | Spring 2026: pros beat the top-10 bot team by only −1.25 head-to-head (n.s., p=0.247). In Q2 2025 the gap was −20.03 | Best source of forecaster engineering practice | R12, R13 |
| 7 | AIA Forecaster (Bridgewater AIA Labs) | Research system | ForecastBench + MarketLiquid | none | FB Brier 0.1076 vs superforecasters 0.1110. Liquid markets: 0.1258 vs market 0.1106. Blend: 0.106 | Strongest evidence for the "blend with market" design | R21 |
| 8 | ryanfrigo/kalshi-ai-trading-bot | OSS bot | Kalshi | **real** | 2026-10-02, n=55 forward trades: win rate 58% vs 60% implied, edge −1.9 pts, Brier 0.2356 | Honest negative result; good harness design | R22 |
| 9 | OctagonAI/kalshi-trading-bot-cli | OSS CLI + paid API | Kalshi | none published | No P&L. Flags "mechanical" edges that come from a market-anchored model | Useful edge-provenance idea | R23 |
| 10 | cryptuon/polybot | OSS framework (28★) | Polymarket, Kalshi, Opinion, Binance | none | No P&L or backtests. Defaults to shadow mode with human approval | Architecture reference only | R24 |
| 11 | Prediction Arena (paper) | Benchmark, **real capital** $10k/model | Kalshi + Polymarket | **real** | Kalshi: −16.0% to −30.8% over 57 days. Polymarket: −1.1% average | Strongest negative real-money evidence | R25 |
| 12 | PolyBench (paper) | Benchmark | Polymarket CLOB replay | sim | 2 of 7 models positive (MiMo-V2-Flash +17.6% CWR, Gemini-3-Flash +6.2%); 5 negative. One-week window | Weak positive signal, high variance | R26 |
| 13 | Raven-Agent (paper) | Agent | Polymarket | sim (+ tiny live) | +15.9% ROI on 42 trades (bootstrap CI +3.8 to +30.0%). Live: +$53.92 over 20 predictions | Shows the trade layer matters; tiny n | R27 |
| 14 | WC2026-Agents (paper) | Benchmark | Bookmaker odds | sim | No model beat the bookmaker's Brier. ROI −18% to +10%. Flat bets on favourites beat all 4 agents | Negative | R29 |
| 15 | Foresight-32B (Lightning Rod) | Fine-tuned model | Polymarket (251 questions) | sim | Brier 0.199 vs market 0.170. "Positive simulated profit" with no figures given | Marketing-grade evidence | R34 |

---

## 2. System details

### 2.1 Polymarket/agents [R1]
| Aspect | Finding |
|---|---|
| Status | `archived=true`, last push 2024-11-05, about 3.8k stars (GitHub API) |
| Architecture | Linear pipeline `one_best_trade()`: Gamma events → RAG filter (Chroma) → map to markets → LLM filter → `source_best_trade` → `format_trade_prompt_for_execution`. The `execute_market_order` call is **commented out** in `trade.py` |
| Retrieval | Chroma vector DB over event/market text, plus news and web search connectors |
| Prompting | One "superforecaster" prompt: break the question down, compare with similar past events, give a probability. Default model `gpt-3.5-turbo-16k`, `temperature=0` |
| Ensemble / calibration | None. One sample |
| Prob → bet | **The LLM outputs `size` as a fraction of total funds.** The code parses it with a regex and multiplies it by the USDC balance (`executor.py` l.184–189) |
| Outcome | None published |

### 2.2 Gnosis prediction-market-agent + prediction-market-agent-tooling [R2, R3]
| Aspect | Finding |
|---|---|
| Status | Active (pushes in Apr 2026). On-chain activity is tracked on a Dune dashboard, which renders in JS and was not read |
| Architecture | `DeployableTraderAgent` subclasses, one per model. Variants include "think thoroughly", "Prophet" (PredictionProphet library), microchain (self-modifying prompt), coinflip and a known-outcome baseline |
| Retrieval | Prophet: generate subqueries → Tavily search → scrape → summarise → predict. The research agent runs at T=0.7 and the predictor at T=0. The temperatures come from a third-party docs mirror (mintlify.wiki) and are unverified |
| Ensemble | One model per agent. The agents compete as a population rather than as an ensemble |
| Prob → bet | `FullBinaryKellyBettingStrategy(max_position_amount ∈ [$1,$5], max_price_impact=0.7)`. The max price impact is tuned per agent on historical bets (`scripts/optimize_betting_strategy.py` simulates ROI). The "simplified" Kelly computes `edge·confidence / odds`, which equals (p−m)·m/(1−m) and is smaller than true Kelly (p−m)/(1−m) by a factor m. **derived**: it under-bets. Strategies include `take_profit` exits and a `GuaranteedLossError` guard |
| Outcome | No public PnL found. `deploy.py` l.417/493/767/810 comment out the GPT-4o-mini, GPT-4-Turbo, Claude-3-Opus and Claude-3.5-Haiku agents with: "Even after optimizing, this doesn't seem to get profitable" |

### 2.3 Olas Predict: Omenstrat and Polystrat (Valory) [R5, R6, R7]
| Aspect | Finding |
|---|---|
| Architecture | Trader agent loop: discover markets → sample one → pay a **mech**, a separate "prediction broker" agent, for `p_yes`/`p_no`/confidence → profitability check → trade from a Safe → redeem. Polymarket runs on CLOB v2 (pUSD); Omen runs on an FPMM AMM [R6] |
| Tool choice | The **ε-greedy** policy (`EGreedyPolicy` in `decision_maker_abci/policy.py`) picks among mech "prediction tools" by each tool's historical accuracy |
| Retrieval / prompting | Mech tools such as `superforcaster`, `superforcaster_full_search`, `factual_research`, and a fine-tuned "OLAS Predict R1 14B" |
| Prob → bet | `kelly_criterion.py` grid-searches 500 points for the spend that maximises **E[log wealth]**. It walks the CLOB ask book to get a VWAP fill, or uses the FPMM invariant, and subtracts `fee_per_trade`. Defaults: `min_edge=0.03`, `max_edge` band (Polymarket only), `min_oracle_prob=0.5`, `max_bet 5 USDC`, `n_bets=1`, `floor_balance`, plus a spread gate `POLYMARKET_SPREAD_MIN/MAX` |
| Accuracy | Polystrat tools by accuracy: `superforcaster_full_search` 74.4% (n=121); `superforcaster-polymarket-v1` **56.1% (n=1,835)**; `superforcaster` 53.5% (n=5,858). Omenstrat 30-day accuracy is 69% with Brier 0.18 [R5, claimed]. "Accuracy" ignores the price paid, so on its own it says nothing about edge |
| **Real P&L** | Polystrat trading ROI, prediction only, excluding staking: 7d +27%, 30d +10%, 90d +3%, **since 2026-01-16 −7%**. Total ROI including OLAS staking is −2% lifetime. Omenstrat trading ROI is **−5% lifetime** (since 2025-11-22), while total ROI including staking is **+69%**, so the gain is a token subsidy and not forecasting. "60% of 40 Polystrat agents with ≥10 trades are net-positive vs 15.9% of 2.5M Polymarket wallets" compares a selected subset with everyone [R5, claimed]. The press claim "37% of agents positive, a single trade up 376%" is cherry-picked [R7, claimed] |

### 2.4 Prophet Arena [R8, R9]
| Aspect | Finding |
|---|---|
| Setup | Every model gets the **same context**: an LLM search agent's news snippets plus Kalshi Yes/No prices and volume. The paper covers 1,367 events and 72,136 markets up to 2025-10-11: 76% sports, 8% entertainment, 7% politics |
| Prob → bet (paper) | Risk-neutral: put $1 on YES if p/q_Y ≥ (1−p)/q_N, otherwise on NO. Return = payout per $1 |
| Paper results | GPT-5: Brier 0.184, ECE 0.042, return 0.943. Grok-4: 0.189 / 0.043 / 0.864. Claude Sonnet 4: 0.194 / 0.041 / 0.909. **Market: 0.187 / 0.069 / 0.899.** No model broke even. LLMs did better at long horizons and worse near resolution, "markets incorporate breaking information … more rapidly" |
| Live board (scored 2026-10-05) | Market row 1−Brier 0.9458. Top model 0.9488 ± 0.0028. Simulated returns run from 89.9% (Claude Opus 5.5) to 131.8% (GPT-6.1 Sol, n=438). Agentic Claude Fable 5 returns 124.5% (n=2,708) and GPT-5.6 Sol (fixed context) 105.3% (n=3,088). Method: "buys YES or NO sized by the disagreement" against the ask, beyond the spread. Fees not stated. The board ranks by Brier, not return |
| Caveat | The returns are simulated, sports-heavy and probably pre-fee. Models see the price, so the forecasts are market-anchored |

### 2.5 ForecastBench (FRI) [R10, R11]
Scored with the **Brier Index** (0–100, higher is better). Overall = the mean of dataset and market scores. Data comes from `leaderboard_*_full.js`, last updated 2026-10-07.

| Entry (tournament board; FB-team models get the crowd/market forecast as context) | Dataset | **Market** | Overall |
|---|---|---|---|
| Google DeepMind "fire hedgehog" (#1) | 65.0 | 73.6 | 69.0 |
| Superforecaster median (#3) | 63.6 | 74.6 | 68.6 |
| FutureSearch bull_v4 (#5) | 63.3 | 74.8 | 68.5 |
| Cassi-2026-05-10 (#6; best market score) | 62.2 | **76.0** | 68.4 |
| gpt-5.5 high + web search (#23) | 63.3 | 71.9 | 67.3 |
| claude-opus-5 max + search + code (#41) | 61.7 | 72.4 | 66.6 |
| Public median (#235) | 59.2 | 68.1 | 63.4 |
| **Naive forecaster: copy the market freeze value** (#364) | 48.1 | **71.3** | 58.1 |
| *Baseline board (no tools, no crowd):* best LLM kimi-k3-max | 61.2 | **62.6** | 61.9 |
| *Baseline board:* claude-sonnet-4-6 / gpt-5.5 | 60.1 / 60.8 | 62.1 / 60.8 | 61.1 / 60.8 |

- **derived**: an LLM that does not see the price trails the price by about 9–11 Brier Index points on market questions. The best entries that do see it beat it by about 4.7 points, and their CIs overlap the superforecasters'.
- Secondary analysis [R11] (2026-08-21): "Of 527 entries, 442 score significantly below the market price". Only Cassi and the superforecasters clear it. Seeing the freeze value is worth about 16 BI points.
- FRI projects LLM–superforecaster parity for about Nov 2026 (95% CI Dec 2025–Jan 2028) [R17, R42].

### 2.6 Metaculus AI Benchmark / FutureEval [R12–R20]
| Season | Field | Pros vs top-10 bot team (head-to-head) | Notable |
|---|---|---|---|
| Q3 2024 → Q2 2025 | — | −11.3, −8.9, −17.7, **−20.03** (p=0.00001) | Q2 2025: o3 was the best model. Aggregating multiple runs was worth +1,799 pts on average [R13] |
| Fall 2025 | — | — | 86% of winners ensembled. Winners averaged **~28 LLM calls/question vs 7** for non-winners, and used **1.72–1.75 research sources vs 1.00** (r=0.42, p=0.006) [R16–R18] |
| Spring 2026 (2026-01-07 → 04-15; 297 questions, 173 bots, $50k) | — | **−1.25** [−4.87, 2.37], p=0.247 | Winner **GreeneiBot2**: AskNews + Perplexity sonar-reasoning-pro + GPT-5 search + BrowserUse, multi-model ensemble. Reasoning beat non-reasoning in **8/8** model pairs (p=0.004). Bot teams of **4–10** members scored best; teams of 30+ declined. The best single in-house model was GPT-5.1-high + AskNews at 11.32 [R12] |

Engineering findings, with the strength of evidence:

| Practice | Evidence | Src |
|---|---|---|
| Logistic (Platt) recalibration on your own record | Binary Brier −0.016 (p=0.0005); multiple choice −0.005 | R15 |
| Clip final probabilities to [min, max] | r=+0.48 (p=0.005) among winners | R17 |
| Explicit base-rate step | r=+0.38 among winners | R17 |
| ≥5 forecasts, then the median | "After … cut number of forecasts to 1 which really hurt" | R14 |
| No single search provider is best | AskNews, Perplexity, Exa and GPT-search all have overlapping CIs; source *diversity* is what matters | R16 |
| Agentic search over one-shot | "generate queries → search → identify gaps → search again" | R14, R41 |
| Avoid persona swarms, LLM-merging of distributions, and heavy agent frameworks | Bot-maker reports: "Multiple personas … was a mistake", "Using LLMs to combine distributions was silly" | R14 |
| Guard against treating unresolved questions as already resolved | Prevents catastrophic 99% forecasts | R14 |
| Template prompt structure | (a) time left, (b) status quo, (c) scenario for No, (d) scenario for Yes, "extra weight on the status quo"; ends with `Probability: ZZ%`; clamp to [0.01, 0.99]; 5 predictions per research report; **median** aggregation | R19 |
| nostream (7/93 in Spring 2026, best open-source bot) | Multi-model frontier ensemble through OpenRouter; median; AskNews, Grok-fast grounding and Gemini Flash for research; FRED and Yahoo for finance; monthly review of the worst misses; "write lots of tests" | R20 |

### 2.7 AIA Forecaster [R21]
| Aspect | Finding |
|---|---|
| Architecture | **10 independent agentic forecasters** per question. Each queries two search providers plus an internal news corpus, deciding iteratively what to search. A supervisor agent reviews where they disagree and can run more searches. Then statistical calibration |
| Aggregation | Mean, median and trimmed mean are about equal: 0.1140 / 0.1138 / 0.1142. Supervisor refinement adds a small further gain, reported as about 0.1125 |
| Samples | Brier falls "sharply" from 1 to 5 forecasts, with a modest further gain up to 15. 1 forecast scores 0.1182 and 10 score 0.1140 |
| Calibration | Platt/extremizing with a fixed **α = √3 ≈ 1.73** on the log-odds. FB-7-21 Brier: 0.1140 → **0.1076**. The averaged ensemble is *under*-confident, and α>1 corrects it |
| vs humans | FB-7-21 (498 questions): 0.1076 vs superforecasters 0.1110 vs public 0.1451 |
| **vs markets** | MarketLiquid (1,610 liquid questions): AIA **0.1258** vs market **0.1106**. Simplex regression blend: **0.106**. Weights: LLM **0.33** [0.12, 0.47] and market 0.67 on liquid markets; LLM 0.87 [0.42, 1.00] on FB-Market |
| Trading | None reported |

### 2.8 Kalshi bots on GitHub [R22, R23]
| | ryanfrigo/kalshi-ai-trading-bot (612★, pushed 2026-10-06) | OctagonAI/kalshi-trading-bot-cli (394★, pushed 2026-10-01) |
|---|---|---|
| LLM | One model per decision (default Claude Sonnet 4.5 through OpenRouter, T=0) with a fallback chain. The multi-model "debate" scaffolding is "not wired into the live trading path". Daily AI spend cap $10 | Octagon Research API, a paid market-anchored model; OpenAI, Anthropic, Google or xAI as the LLM |
| Gate | `verify`: research the catalyst → a SKEPTIC tries to refute it → a deterministic gate recomputes edge from the live book → `BUY_NO`/`PASS` | "5-gate risk engine": Kelly, liquidity, correlation, concentration, drawdown |
| Sizing | **Quarter-Kelly**, max 3% per position, 10 concurrent positions, sector cap 30%. Halts at 10% daily loss or 15% drawdown | **Half-Kelly** |
| Learning loop | `cli edge` reports Brier, log-loss and **edge-vs-book** (realised win rate − mean implied price). It refuses to claim edge with fewer than 10 forward trades. An `Edge Policy` blocks losing categories (minimum n=5 per group) | `calibration` (Brier and skill vs market per category); paper ledger with real quotes and settlements; flags "mechanical" edge |
| **Outcome** | **Real, n=55: won 58% vs 60% implied → edge −1.9 pts, Brier 0.2356.** YES side −22.7 pts (n=17), NO side +7.6 pts (n=37) | None published |

### 2.9 cryptuon/polybot [R24]
28★, created 2025-12, pushed 2026-07-16. Services connected over NNG, with FastAPI, a scanner, an executor and an analytics service. It ships 10 strategy plugins: arbitrage, AI-model, spread farming, copy trading, resolution arbitrage and others. An MCP server exposes typed tools, and agents *propose* trades that humans approve. **Shadow mode is the default**, and each strategy is promoted to live separately and reversibly. Platform-wide position, daily-loss and exposure caps apply. No Kelly or P&L is documented. Worth copying: the per-strategy shadow→live promotion. Worth skipping: the 10-strategy sprawl.

### 2.10 2026 papers (plus key older ones)
| Paper | Setup | Key numbers | Takeaway |
|---|---|---|---|
| **Prediction Arena** (2604.07355, 2026-03) [R25] | 6 frontier models, $10k real each. Kalshi 2026-01-12→03-09; Polymarket 02-09→03-09. Web search, notes, 15% concentration cap, no Kelly | Kalshi: glm-4.7 −16.0%, grok-4-20 −20.0%, gpt-5.2 −20.5%, claude-opus-4.5 −25.9%, gemini-3-pro −30.5%, grok-4.1-fast −30.8%. Polymarket average −1.1%. 71–97% of Kalshi settlements were in weather markets. "Research quantity shows no correlation with performance" | Autonomous LLM trading lost real money |
| **PolyBench** (2604.14199) [R26] | 38,666 Polymarket snapshots (2026-02-06→12) with CLOB state and Google News; trades only if confidence > 0.6; stake ∝ confidence; walks the book | Gemini-3-Flash: accuracy 75.0%, CWR +6.2%. MiMo-V2-Flash: 62.1%, +17.6%. The other 5 lost (−9.2% to −25.2%). Every model reported confidence of about 0.81–0.87 | Stated confidence is flat and uninformative. Profits in politics, losses in crypto |
| **Belief-to-Trade / Raven-Agent** (2607.03015) [R27] | Same archived forecasts (Brier 0.329), 5 trading policies, Polymarket replay | Flat-stake −10.7%; **edge-proportional −55.5%**; edge filter at 5 pp −9.3%; Raven fixed stake −4.7%; **Raven with ¼-Kelly, time-normalised ranking, liquidity clipping, stop-loss and drawdown halt +15.9%** (n=42) | "Sizing alone, without selection, is harmful". Keep the trade layer deterministic and outside the prompt |
| PolySwarm (2604.03888) [R28] | 50 LLM personas, confidence-weighted Bayesian combination with the market, ¼-Kelly, KL/JS divergence for finding inefficiencies | No trading results in the abstract | Design paper only; personas are an anti-pattern (R14, R31) |
| **KalshiBench** (2512.16030) [R30] | 300 Kalshi questions resolving after the training cutoff | All 5 models overconfident; **27% error when they state >90%**; ECE 0.120 (Opus 4.5) to **0.395 (GPT-5.2-XHigh)**; only 1 model has a positive Brier Skill Score | Raw single-shot LLM probabilities must not be used as they are |
| **Monoculture** (2606.26583) [R31] | Agents fine-tuned with DPO | Pairwise error correlation **ρ=0.70**. 10 agents ≈ **1.4 independent forecasters**. **Using different models cuts ρ from 0.68 to 0.40** | Many samples from one model add little; use several model families |
| Simulated Ignorance Fails (2601.13717) [R32] | "Pretend it's date X" prompts | 52% performance gap between simulated and true ignorance; CoT does not fix it | **No retrospective LLM backtests**; forward-test only |
| Pitfalls (Paleka et al., 2506.00723) [R38] | Methodology | Temporal leakage in many forms (retrieval dates, logical) | Freeze evidence with timestamps |
| **Foresight Arena** (2605.00420) [R33] | On-chain commit-reveal benchmark on Polymarket | Detecting a **Brier edge of 0.02 at 80% power takes about 350 resolved predictions**; an edge of 0.01 takes about 4× as many | Sizes our Phase F gate |
| WC2026-Agents (2607.17765) [R29] | 4 frontier agents, 104 World Cup matches, $100 virtual bets | No agent beat the bookmaker's Brier; ROI −18% to +10%; a flat bet on favourites beat all of them; identical top picks in 92% of matches | Liquid sports lines are not a target |
| Foresight-32B (Lightning Rod) [R34] | Qwen3-32B with RL; 251 Polymarket questions (2025-07-25→08-25) | Brier **0.199 vs market 0.170** (o3 about 0.208). Rule: "bet when edge > own ECE". Profit amounts not given | Even the best fine-tune trails the price |
| Mantic / Thinking Machines [R39, R40] | RL fine-tuning of gpt-oss-120b on about 10k binary questions | Mantic placed 8th of 549 in the Metaculus Cup (press; not re-verified). Future-as-Label: Qwen3-32B Brier −27% | Fine-tuning is a later option, not an MVP item |
| Halawi et al. 2024 (2402.18563) [R35] | RAG with up to 15 article summaries, fine-tuned plus base model, **trimmed mean of 6** | System **0.179** vs crowd **0.149** on 914 questions (300 of them from Polymarket); **weighted blend with the crowd at 4× weight: 0.146** | Blending with a dominant market weight is the right default |
| Schoenegger et al. 2024 (2402.19379) [R36] | Median of 12 LLMs vs 925 humans, 31 questions | LLM crowd median is statistically the same as the human crowd. **Acquiescence bias**: mean forecast > 50% although resolutions split about evenly. Feeding the human median into the LLM improved it 17–28%, but **less than simply averaging** the two | Blind LLM pass, then blend in code. Watch for YES-bias (compare ryanfrigo's YES side at −22.7 pts) |
| Beta-Bernoulli Calibrator (2605.27668) [R37] | Calibrates a point estimate using outcomes plus human/crowd forecasts | Reported better than post-hoc methods and forecasting fine-tunes; no figures in the abstract | Candidate for the C4 calibrator v2 |

---

## 3. Evidence: LLM vs the market (Brier, lower is better unless noted)

| Study | LLM alone | Market/crowd | LLM + market | Notes |
|---|---|---|---|---|
| AIA, MarketLiquid [R21] | 0.1258 | **0.1106** | **0.106** (LLM weight 0.33) | Liquid markets |
| AIA, FB-Market [R21] | — | — | LLM weight 0.87 | Less liquid/forecasting-site questions: the LLM gets more weight |
| Halawi [R35] | 0.179 | 0.149 | **0.146** (crowd 4×) | 914 questions |
| ForecastBench market questions [R10] | BI 60–63 (no price) | BI 71.3 (copy price) | BI up to 76.0 (price in context) | Higher BI is better |
| Prophet Arena paper [R8] | 0.184 (GPT-5, sees price) | 0.187 | — | Sim return 0.943 < 1 |
| Prophet Arena live [R9] | 1−B 0.9488 (sees price) | 1−B 0.9458 | — | Higher is better; sim returns 90–132% |
| Foresight-32B [R34] | 0.199 | **0.170** | — | n=251 |
| WC2026 [R29] | worse than bookmaker | — | — | n=104 |

**derived:** on liquid markets the market price beats any LLM alone by about 10–20% in Brier. A blend with the market at about ⅔–⅘ weight improves on the market by about 2–4%. That is the size of the "edge" to expect, before spread and fees.

---

## 4. Real-money outcomes (all we could find)

| Who | Period | n | Result | Type |
|---|---|---|---|---|
| Prediction Arena, 6 models [R25] | 57 days (Kalshi), ~1 month (Polymarket), 2026 | 129–886 trades/model | **−16% to −31%** (Kalshi); **−1.1% average** (Polymarket) | real, controlled |
| Olas Polystrat fleet [R5] | 2026-01-16 → 2026-10-07 | 138 agents (15 daily active) | Trading ROI **−7% lifetime**; +10% over 30d; +3% over 90d; −2% total ROI including staking | real, claimed |
| Olas Omenstrat fleet [R5] | 2025-11-22 → 2026-10-07 | 1,208 agents | Trading ROI **−5% lifetime**; total ROI +69% (staking subsidy) | real (play-ish xDAI), claimed |
| ryanfrigo Kalshi [R22] | to 2026-10-02 | 55 settled | Edge **−1.9 pts** vs book; Brier 0.2356 | real |
| Gnosis Prophet agents [R2] | 2024–2026 | — | Several variants marked unprofitable in code; no public totals | real, unpublished |
| Raven-Agent live [R27] | 2026 | 20 predictions | +$53.92 | real, n too small |

---

## 5. Recommendations

### (a) Best practices to adopt in our forecaster (C2–C6)

| # | Practice | Concrete setting for us | Evidence |
|---|---|---|---|
| 1 | **Blind pass 1** | The LLM never sees price, volume or comments in pass 1. The market enters only in C4, in code | R11, R36, R21 |
| 2 | **Prompt structure** | XML sections: `<question>`, `<resolution_criteria>` (verbatim), `<fine_print>`, `<today>`, `<close_time>`, `<evidence id=… date=… source=…>`. Reasoning steps in order: (1) restate exactly what resolves YES; (2) time left; (3) status quo if nothing changes; (4) reference class and base rate (outside view); (5) YES path and NO path (inside view); (6) evidence for and against, citing ids; (7) adjust the base rate; (8) check "is this already resolved or ambiguous?"; then strict JSON `{verdict, p_yes, key_evidence_ids, …}` as in `02-architecture.md` §4 | R14, R17, R19 |
| 3 | **Samples: raise K from 3 to ≥ 5–6** | K = 6, for example 2–3 samples × 2–3 model families. The biggest gain comes from 1 to 5 samples. Samples from one model are highly correlated (ρ≈0.7) | R21, R14, R31 |
| 4 | **Models** | Reasoning models first: reasoning won all 8 of 8 paired comparisons. Free-tier candidates are whatever reasoning models are actually free, e.g. Gemini Flash-class, DeepSeek-R1/V3.x, Qwen3-235B and Grok-fast-class through OpenRouter or provider free tiers. Expect a handicap against frontier bots, so record the model id with every forecast | R12, R14 |
| 5 | **Retrieval** | An agentic loop with at most 2 rounds: generate queries → search → name the gaps → search again. **At least 2 distinct sources**, e.g. a news API plus web search or grounding, plus structured data (FRED, Yahoo, official sites) where it applies. Keep the published timestamp on every item and drop items dated after the forecast time. Quality matters more than quantity | R16, R18, R25 |
| 6 | **Aggregation** | The median of the K probabilities, or the mean of their log-odds; they perform about the same. Do not let an LLM merge the forecasts, and do not use personas. A supervisor re-search on high disagreement is optional, for v2 only | R14, R21 |
| 7 | **Clip** | Clip raw p to [0.03, 0.97] before calibration. Final q may reach [0.01, 0.99] only through the blend | R17, R30 |
| 8 | **Calibration** | Use 1–2-parameter **Platt** on logit(p), fitted on our own resolved forward record. Isotonic needs far more data and is a v2 option once N ≥ 300 (**derived**). Fit the direction from data: KalshiBench finds single shots overconfident, while AIA's averaged ensemble was underconfident (α=√3). Until N is reached, keep `q := market` as the design already says | R15, R21, R30 |
| 9 | **Market blend** | Blend in log-odds: `logit q = w·logit p_cal + (1−w)·logit m`. Prior **w = 0.25–0.33** for liquid markets (AIA 0.33; Halawi 0.2), allowed to rise toward 0.5+ only for illiquid or long-horizon markets once our own regression supports it. Fit w by constrained regression on our record | R21, R35 |
| 10 | **Abstain** | Skip a trade if: the verdict is not FORECAST; the IQR across samples is above a set width (e.g. 0.15, our proposal); `edge < max(min_edge, ECE_own)` (Olas default min_edge 0.03; Lightning Rod's rule is edge > ECE); **`edge > max_edge`** (a huge disagreement with a liquid price is more likely our error; Olas exposes this band); less than about 24–48h to resolution (markets absorb news faster there, per Prophet Arena); or the market is short-horizon crypto or sports | R6, R8, R26, R34 |
| 11 | **Sizing** | Use Olas-style **execution-aware Kelly**: maximise E[log wealth] over a grid of spends, walking the live ask book (VWAP) and subtracting fees. Then take **¼-Kelly** and apply the caps in `04` §2. Never let the LLM output a size | R6, R27, R22, R1 |
| 12 | **Edge provenance** | Store `edge_source ∈ {llm_disagreement, calibration_shift, blend_artifact}` per decision. If q differs from m only because the calibrator moved it, flag the trade as mechanical (Octagon's idea) | R23 |
| 13 | **Measurement** | The primary metric is **edge-vs-book** (realised win rate − mean price paid) plus paired Brier against the mid, with splits by side (YES/NO), category and horizon. Do not use win rate or "accuracy" on their own | R22, R5 |
| 14 | **Power** | Detecting a 0.02 Brier edge at 80% power needs about **350 resolved forecasts**. Treat ≥60 as a smoke test and set G2/kill at about 300–350, or accept a weaker test with explicit CIs. *Owner decision.* | R33 |
| 15 | **Budget arithmetic** | **derived:** about 2 calls for query generation and evidence summary, plus K = 6 forecasts, is about 8 LLM calls per market. With the 100-calls/day cap that allows about 12 markets/day. Tournament winners used about 28 calls per question, so screen hard in C1 rather than cutting K | R17 |
| 16 | **No retro LLM backtests** | Forward-only Phase F with frozen evidence hashes. Retrospective prompts like "pretend it's date X" leak 52% of the gap | R32, R38 |

### (b) Anti-patterns seen

| Anti-pattern | Where | Why it fails |
|---|---|---|
| LLM chooses the position size (fraction of funds via regex) | Polymarket/agents [R1] | Uncalibrated, unbounded, can't be audited |
| Fully autonomous LLM trader (it researches, decides, sizes and executes) | Prediction Arena [R25] | −16% to −31% real |
| Sizing proportional to edge or confidence with no selection | Raven baseline [R27]; PolyBench CWR [R26] | −55.5% ROI; stated confidences sit at a flat 0.81–0.87 |
| Treating accuracy or win rate as skill | Olas tool board, Polystrat PR [R5, R7] | 56% accuracy at 60¢ prices loses money |
| Profit that is really a token subsidy | Omenstrat (+69% total vs −5% trading) [R5] | The model has no edge |
| Selected-subset statistics ("60% of agents with ≥10 trades positive") | Olas [R5] | Survivorship |
| Persona swarms and many samples from one model | PolySwarm [R28]; bot advice [R14]; monoculture [R31] | ρ≈0.7: 10 agents ≈ 1.4 independent |
| Single-shot, T=0, one model | Polymarket/agents [R1]; template cost-cutting [R14] | High variance; Brier gets clearly worse |
| Giving the LLM the price and asking for an adjustment | ForecastBench entries [R11] | 442/527 entries fell below copying the price |
| Retrospective backtests with "simulated ignorance" | Common in repos [R32] | Leakage inflates results |
| Tuning bet parameters per agent on small historical samples | Gnosis `max_price_impact` per agent [R2] | Overfitting (values in code: 0.068, 0.7, 1.38) |
| Short-horizon or near-resolution trading | Prophet Arena [R8]; PolyBench crypto [R26] | Markets update faster than an LLM loop |
| Trading liquid sports lines against a sharp book | WC2026 [R29]; Prophet Arena (76% sports) [R8] | No model beats the bookmaker |
| Claiming edge from a handful of settlements | ryanfrigo's explicit gate [R22] | Need n ≥ 10 for any statement and ~350 for a Brier edge |

### (c) Honest assessment: is there documented evidence that LLM agents make money on real-money markets after fees?

**No, not robustly, as of 2026-10-07.**
- Every controlled real-money test we found is negative: Prediction Arena on Kalshi and Polymarket [R25].
- The one public fleet-scale real-money record, Olas, is **negative over its lifetime** on trading. Only its recent 30–90-day windows and its token-subsidised totals are positive, and all of it is self-reported [R5].
- The most transparent individual operator publishes "no measured edge" at n=55 [R22]. Gnosis's code marks several of its agents as unprofitable [R2].
- Every positive result is **simulated or tiny**: Prophet Arena's live sims (fees not stated, sports-heavy, models see the price) [R9]; 2 of 7 PolyBench models over 1 week [R26]; Raven-Agent at n=42 backtest and n=20 live [R27]; Foresight-32B with no figures [R34].
- What *is* well documented is narrower. An LLM ensemble **blended with the market price** beats the market price on Brier by a few percent [R21, R35]. On less liquid or longer-horizon questions the LLM earns more weight [R21, R8].

Whether a few percent of Brier survives spread, taker fees and the 5-share minimum on long-tail Polymarket markets is **unproven anywhere**. P1 is therefore an experiment and not a known edge. Treat the $50–150 bankroll as tuition for a forward record.
- **derived:** no documented LLM forecasting agent comes near $5–10k/month. Polystrat's best recent window, +10% over 30 days, would be about $5–15 a month on our bankroll.

---

## 6. Deltas to our design docs (for the planner)

| Doc | Current | Proposed | Evidence |
|---|---|---|---|
| `02-architecture.md` §4.3 | K = 3, ≥2 families, median | **K ≥ 5–6, ≥ 2 (ideally 3) families**, median or mean of logit; IQR gate | R21, R31, R14 |
| `02-architecture.md` §4.4 | Isotonic or Platt | **Platt first**; isotonic only at N ≥ ~300; clip raw p to [0.03, 0.97] | R15, R17 |
| `02-architecture.md` §4 blend | "blend with market" (weight unspecified) | log-odds blend, prior w = 0.25–0.33, fitted by constrained regression on our own record | R21, R35 |
| `02-architecture.md` sizer | `f_kelly` at the best ask | **Book-walking (VWAP) log-utility Kelly** incl. fee, then ¼-Kelly, then caps; add a **max_edge** reject | R6, R27 |
| `03-strategies.md` P1 kill | ≥ 60 resolved forecasts | Smoke test at 60; **decision at about 300–350** (power for a 0.02 Brier edge); metric = edge-vs-book + paired Brier | R33, R22 |
| C2 evidence | unspecified | ≥ 2 distinct sources; agentic 2-round search; timestamp filter | R16, R18 |
| Market filter | — | Exclude markets < 24–48h to resolution and liquid sports/crypto lines for P1 | R8, R26, R29 |

---

## 7. Sources (all accessed 2026-10-07)

| ID | Source |
|---|---|
| R1 | Polymarket/agents — https://github.com/Polymarket/agents (API: archived, pushed 2024-11-05); code `agents/application/{trade,executor,prompts}.py` via raw.githubusercontent.com |
| R2 | gnosis/prediction-market-agent — https://github.com/gnosis/prediction-market-agent ; `prediction_market_agent/agents/prophet_agent/deploy.py` (l.122–128, 417, 493, 767, 810); Dune: https://dune.com/gnosischain_team/ai-agents-overview-omen-prediction-markets (not rendered); docs mirror https://mintlify.wiki/gnosis/prediction-market-agent/agents/prophet-agents (third-party) |
| R3 | gnosis/prediction-market-agent-tooling — `prediction_market_agent_tooling/tools/betting_strategies/kelly_criterion.py`, `deploy/betting_strategy.py`, `scripts/optimize_betting_strategy.py` |
| R5 | Olas Predict metrics page — https://olas.network/agent-economies/predict (data "as of 7 October 2026 08:20 UTC") |
| R6 | valory-xyz/trader — https://github.com/valory-xyz/trader ; `packages/valory/customs/kelly_criterion/kelly_criterion.py`; `packages/valory/skills/decision_maker_abci/policy.py` |
| R7 | Polystrat press coverage (claimed) — https://www.mexc.com/news/932530 ; https://www.edgen.tech/es/news/post/ai-agents-score-376-return-rewriting-prediction-markets |
| R8 | Prophet Arena paper — https://arxiv.org/html/2510.17638v2 (ICLR 2026) |
| R9 | Prophet Arena live leaderboard — https://www.prophetarena.co/leaderboard (last scored 2026-10-05) |
| R10 | ForecastBench leaderboards — https://www.forecastbench.org/leaderboards/ (data: `/assets/js/leaderboard_tournament_full.js`, `leaderboard_baseline_full.js`, "last updated 2026-10-07") |
| R11 | "The copier line: what ForecastBench's market scores actually…" (2026-08-21) — https://forum.nunosempere.com/posts/8YuG9EuhyYqaNBt7F/the-copier-line-what-forecastbench-s-market-scores-actually |
| R12 | FutureEval Spring 2026 results (2026-09-09) — https://www.lesswrong.com/posts/wZBbDqzfBjYG58CxK/futureeval-spring-results-pros-beat-bots-but-the-gap-is |
| R13 | Q2 2025 AI Benchmark results — https://www.lesswrong.com/posts/Surnjh8A4WjgtQTkZ/q2-ai-benchmark-results-pros-maintain-clear-lead |
| R14 | Metaculus "Bot advice (Fall 2025)" — https://www.metaculus.com/notebooks/43357/bot-advice-fall-2025/ |
| R15 | Metaculus calibration-adjustment analysis — https://www.metaculus.com/notebooks/43356/calibration-adjustment-analysis/ |
| R16 | Metaculus search-provider analysis — https://www.metaculus.com/notebooks/43361/search-provider-analysis/ |
| R17 | Metaculus "AI forecasting in 2026" (2026-05-16) — https://www.metaculus.com/notebooks/43363/ai-forecasting-in-2026/ |
| R18 | Fall 2025 FutureEval survey — https://www.metaculus.com/notebooks/43337/fall-2025-futureeval-survey/ |
| R19 | Metaculus/metac-bot-template `main.py` — https://github.com/Metaculus/metac-bot-template ; median: `forecasting_tools/data_models/binary_report.py` in https://github.com/Metaculus/forecasting-tools |
| R20 | nostream, "On writing a bitter-pilled Metaculus bot" (2026-05-03) — https://nostream.substack.com/p/on-writing-a-bitter-pilled-metaculus |
| R21 | AIA Forecaster technical report — https://arxiv.org/html/2511.07678 |
| R22 | ryanfrigo/kalshi-ai-trading-bot README — https://github.com/ryanfrigo/kalshi-ai-trading-bot |
| R23 | OctagonAI/kalshi-trading-bot-cli README — https://github.com/OctagonAI/kalshi-trading-bot-cli |
| R24 | cryptuon/polybot — https://github.com/cryptuon/polybot |
| R25 | Prediction Arena (2604.07355) — https://arxiv.org/html/2604.07355 |
| R26 | PolyBench (2604.14199) — https://arxiv.org/html/2604.14199v1 |
| R27 | Beyond Forecasting: Belief-to-Trade Layer (2607.03015) — https://arxiv.org/html/2607.03015 |
| R28 | PolySwarm (2604.03888) — https://arxiv.org/abs/2604.03888 |
| R29 | WC2026-Agents (2607.17765) — https://arxiv.org/abs/2607.17765 |
| R30 | KalshiBench (2512.16030) — https://arxiv.org/abs/2512.16030 |
| R31 | Preference Optimization Drives Monoculture in LLM Prediction Markets (2606.26583) — https://arxiv.org/abs/2606.26583 |
| R32 | Simulated Ignorance Fails (2601.13717) — https://arxiv.org/abs/2601.13717 |
| R33 | Foresight Arena (arXiv 2605.00420) — https://econpapers.repec.org/paper/arxpapers/2605.00420.htm |
| R34 | Lightning Rod, Foresight-32B on live Polymarket — https://blog.lightningrod.ai/p/foresight-32b-beats-frontier-llms-on-live-polymarket-predictions |
| R35 | Halawi et al., Approaching Human-Level Forecasting — https://arxiv.org/html/2402.18563 |
| R36 | Schoenegger et al., Wisdom of the Silicon Crowd — https://arxiv.org/abs/2402.19379 |
| R37 | Beta-Bernoulli Calibrator (2605.27668) — https://arxiv.org/abs/2605.27668 |
| R38 | Paleka et al., Pitfalls in Evaluating LM Forecasters — https://arxiv.org/abs/2506.00723 |
| R39 | Thinking Machines × Mantic — https://www.thinkingmachines.ai/news/training-llms-to-predict-world-events/ ; Mantic 8th/549 (press, unverified): https://www.thestar.com.my/tech/tech-news/2026/09/18/ai-startup-mantic-raises-25-million-for-superhuman-forecasting |
| R40 | Future-as-Label / Foresight Learning (2601.06336) — https://arxiv.org/abs/2601.06336 |
| R41 | smingers, "One shot is not enough" (2026-04-16) — https://thonks.substack.com/p/one-shot-is-not-enough |
| R42 | "AI forecasting in 2026: what 11 analyses say" — https://www.lesswrong.com/posts/a82q6yd8zKpYk56cF/ai-forecasting-in-2026-what-11-analyses-say |

---

## Verification (2026-10-07)

An adversarial re-check of the 10 headline claims, re-run independently on 2026-10-07; it replaces an earlier pass from the same day. Each cited source was re-opened as a raw artifact: arXiv HTML, raw GitHub files plus the commits API, the ForecastBench leaderboard JS and source code, the Olas page HTML, and the Metaculus notebook payload. Each claim was also checked against at least one independent source or code path. **Access date for every URL below: 2026-10-07.** Verdicts are confirmed, partly, unverifiable or refuted. **No claim was refuted**, so no inline markers were added. **derived** means the verifier computed the number. **vendor/self-reported** means it cannot be checked independently.

| # | Claim (short) | Verdict | Note | Better source |
|---|---|---|---|---|
| 1 | Prediction Arena: Kalshi −16.0% to −30.8% over 57 days; Polymarket average −1.1% | confirmed | The abstract and §7 match. Kalshi results: glm-4.7 −16.0% and grok-4-1-fast-reasoning −30.8%. The mean of the six models is **−23.95%** (derived). The paper's "concurrent" Kalshi average is −22.6%. **The Polymarket leg covered only Feb 9 to Mar 9 (~1 month).** Per-model Polymarket returns ranged from −1.85% to −0.09%, on 18 to 155 trades. The $10k starting capital is stated for Kalshi; the paper gives no separate Polymarket stake. It is a single non-peer-reviewed preprint, v1 dated 28 Mar 2026. Independent coverage (econpapers, roboticscenter.ai) only restates the abstract | https://arxiv.org/abs/2604.07355 (Table 3, §7.4) |
| 2 | Olas Polystrat −7% trading ROI since 16 Jan; 30d +10%; 90d +3%; Omenstrat −5% trading vs +69% total | confirmed (vendor-reported) | The page is stamped "7 October 2026 08:20 UTC" and matches all six numbers. The Omenstrat lifetime window starts on **22 Nov 2025**. Other figures on the page: Polystrat total ROI since 16 Jan is −2%, and the 7d trading ROI is +27%, so short windows are noisy. The page's own lifetime histogram puts **81.6% of 157 "qualifying" Polystrat agents (≥10 trades) at negative trading ROI** (derived by summing the bins). Data-quality flag: 157 qualifying agents exceeds the 138 "Total Agents" figure. Press coverage quotes Olas as "not yet at a fleet-wide break even", which is consistent with these figures. There has been no on-chain audit | https://olas.network/agent-economies/predict ; https://laikalabs.ai/news/ai-agents-polymarket-wallets-polystrat-376-percent-trade (press, vendor-sourced) |
| 3 | ryanfrigo: NO MEASURED EDGE, n=55, 58% vs 60%, −1.9 pts, Brier 0.2356; YES −22.7 (n=17), NO +7.6 (n=37) | confirmed (self-reported) | The README matches. `docs/TRACK_RECORD.md` was auto-generated on 2026-10-02 in commit `40857555`. Caveats: **the Brier and log-loss figures use n=43 of the 55 trades.** The side splits (17+37=54) appear only in the README, not in TRACK_RECORD. The account is also traded manually by its operator: equity is $929, which is 68.8% below its $2,975 peak. Across all 200 settlements the account is at −$156.59 net, and YES positions are −$188.82 at a 44% win rate. Commit `f0235cc0` (2026-07-04) had claimed all-time realized P&L of "+$330". None of this can be verified against Kalshi | https://github.com/ryanfrigo/kalshi-ai-trading-bot/blob/main/docs/TRACK_RECORD.md |
| 4 | AIA: 0.1258 vs market 0.1106 on 1,610 questions; blend 0.106, LLM weight 0.33 [0.12, 0.47]; 10 forecasts; Platt α=√3; FB 0.1076 vs supers 0.1110 | confirmed | Tables 2, 7, 9, 11 and 13–15 match. Caveats: the 1,610 questions are 322 markets × 5 dates, so they are correlated. **The AIA-vs-market p-value is 0.0567** (Table 15). The 0.106 figure is leave-one-out, and the market weight is 0.67 [0.53, 0.80]. The FB-7-21 comparison with superforecasters has p=0.152. **On FB-Market (market questions), superforecasters score 0.0740 and beat AIA's 0.0753.** The value d=√3 was fixed in advance from Neyman & Roughgarden (2022), not fitted. The source is an industry technical report (Nov 2025, v1), not peer reviewed | https://arxiv.org/abs/2511.07678 |
| 5 | ForecastBench: naive forecaster "copies the market freeze value" and scores 71.3; baseline LLMs ~60–63; Cassi 76.0; supers 74.6 | partly | All four numbers match the leaderboard JS (stamped "last updated 2026-10-07"). Baseline LLMs top out at 62.8 (o3-scratchpad). **The wording is wrong: the Naive Forecaster does not use the freeze value.** `_impute.py` fills its market forecasts with `market_value_on_due_date_minus_one`, the t−1 price, while the freeze value is taken `FREEZE_WINDOW_IN_DAYS = 10` days earlier (`question_curation.py`). A forecaster that copies the freeze value would score **≈70.1** (derived: Imputed (due-date) line 71.4 → C=0.0818, then add δ=0.0076 from R11, giving (1−√0.0894)×100). Cassi's 76.0 is the highest market score on the board, but it rests on N=109 with CI [72.8, 79.9], and Cassi ranks only 6th overall. The superforecasters' 74.6 rests on 57 questions and appears only on the tournament board; **the baseline board shows them at 72.0** because it uses a different difficulty adjustment | https://github.com/forecastingresearch/forecastbench/blob/main/src/resolve/_impute.py ; https://www.forecastbench.org/assets/js/leaderboard_tournament_full.js |
| 6 | 442 of 527 FB entries are significantly below copying the price shown in their prompt | partly | The figures 527 and 442 match the post. It is by the pseudonymous "Dominus", has karma +1, uses data as of 11 Aug 2026 and was posted 21 Aug; it is not peer reviewed. **However, the test is against the due-date price**, which nobody saw, not the price shown in the prompt. The post's TL;DR effect for Cassi (−0.0220) equals the gap to the due-date line, not to the shown-price line (≈−0.030, derived). The post itself reports the shown price at Brier 0.0843 vs 0.0767 for the due-date price, and finds **204/273 (75%)** at or below the shown-price line by point estimate only. Our recount on the 2026-10-07 board (419 non-reference entries, uncorrected 95% CIs, derived) gives two results. Against the due-date line (71.4), **206** entries are wholly below and 10 wholly above. Against the shown-price line (~70.1), **159** are wholly below and **60 wholly above**. The direction holds, but the magnitude is overstated | Post (above) + https://www.forecastbench.org/assets/js/leaderboard_tournament_full.js |
| 7 | FutureEval Spring 2026: −1.25 [−4.87, 2.37], p=0.247, vs −20.03 in Q2 2025; reasoning beats non-reasoning 8/8; teams of 4–10 best | partly | The head-to-head figures are confirmed: 99 Pro questions, one-sided t-test, post dated 2026-09-09, and the EA Forum cross-post matches. Two parts are mis-stated. (a) The 8 pairs compared **a higher-reasoning-effort version against the standard version of the same model**, for example gpt-5.1-high at 11.32 vs gpt-5.1 at 4.56. The sign test gives p=0.004, but the pairs share the same questions. (b) **Teams of 2 to 10 all scored about −2.** A 4-bot team had the best point estimate, but Metaculus says the noise prevents ranking team sizes. Performance dips above 10 bots and plateaus around 50. The best single bot scored −4.51 | https://forum.effectivealtruism.org/posts/iJ3qBDwpWCfNy9ZGd/pros-beat-bots-but-the-gap-is-nearly-gone |
| 8 | Platt recalibration of Metaculus bots: binary Brier −0.016 (p=0.0005); MC −0.005 | confirmed | The notebook table gives binary mean Δ 0.016426, p=0.000523, raw 0.1840 → 0.1675. Multiple choice gives Δ 0.005357, p=0.000053. Scope: bots from the 2025 Q1–Q2 AI Benchmarking tournament with **≥500 forecasts each**, one fit per forecaster, a chronological 70/30 split and a one-sided Wilcoxon test. Fits fail (and the bot is excluded) when the slope is ≤0 or the training data has only one class. The code exists in `Metaculus/forecasting-tools/forecasting_tools/calibration_adjustment/logistic_recalibration_adjuster.py` | Same notebook (LukeBeukelman, published 2026-05-01) |
| 9 | Raven: edge-proportional −55.5%, flat −10.7%, Raven (selection + ¼-Kelly + limits) +15.9% on 42 trades, CI +3.8% to +30.0% | confirmed | Tables 1 and 3 match: Raven-full staked $820 for +$131, against $580 stakes for the baselines. **Key caveat: with the same 42 selected trades at a fixed stake, Raven scores −4.7%** (CI −20.6% to +11.5%). The positive result therefore comes from sizing on top of selection, on a single 42-trade replay. The full-archive forecaster Brier is **0.329, worse than a 0.25 coin flip**. Unresolved positions are marked to market. On the fused 132-row archive at a fixed stake, ROI is +5.9% (App. E). It is a single preprint (3 Jul 2026). Independent checks: the repo Alchemist-X/predict-raven exists (75★, last pushed 2026-09-29). The paper's wallet `0x6664…614e` shows 42 trades (2026-03-16 to 06-27) and $138.10 of current value in data-api. That shows the deployment is real, not the replay ROI | https://arxiv.org/abs/2607.03015 ; https://data-api.polymarket.com/value?user=0x6664e32f79aee42639f73633e40b5a842b07614e |
| 10 | Detecting a 0.02 Brier edge at 80% power needs ~350 predictions; 0.01 needs ~4× | confirmed | Paper Table: α*=0.02 → 348 (50 rounds × 7); 0.01 → 1,392; 0.005 → 5,567. Assumptions: one-sided 5%, q̄=0.5, mean \|b−p\|=0.15, giving n ≳ 0.139/α*². Re-derived independently: Var(α_i)=4q(1−q)(b−p)², so n=((1.645+0.842)·0.15/0.02)² ≈ 348. Because n scales with (\|b−p\|/α*)², the threshold depends on how far we deviate from the market. "Edge" here means Brier gain over the market, not P&L. For frontier LLMs vs market (\|α*\| 0.003–0.008) the paper says 2,200 to 15,500 predictions are needed. The paper's simulations are seed-controlled and there are no live results. econpapers is only a mirror | https://arxiv.org/abs/2605.00420 (§3.6.2) |

**Corrections for the planner** (these supersede the matching text above; the rest of the report is unchanged):
- §2.5 "copy the market freeze value": the 71.3 baseline copies the **t−1 (due-date-minus-one)** price. A true freeze-value copier scores about 70.1 (derived). The direction of TL;DR #2 and §3 is unchanged.
- TL;DR #4 and §5b "442/527 below copying": the reference is a fresher price that no entry saw. Read it as "most entries add nothing measurable over the market price; about 60 of 419 sit clearly above the shown price". The rule of running a blind pass and then blending with the market in code still stands.
- §2.6 and §5a #4: replace "reasoning beat non-reasoning" with "**higher reasoning effort** beat standard effort, 8/8 pairs". Replace "bot teams of 4–10 best" with "teams of 2–10 about equal; >10 worse".
- TL;DR #2–3, §2.7, §3 and §5a #9 (AIA): the LLM-vs-market gap is not significant (p=0.057). The blend weight to use as a prior is **LLM ≈ 0.33, market ≈ 0.67**. On market-sourced FB questions, superforecasters still beat AIA.
- §1 #13, §2.10 and §5b (Raven): selection alone at a fixed stake was −4.7%. Do not cite Raven as evidence that selection works. It is weak, n=42 evidence that ¼-Kelly sizing on top of selection helped on one replay.
- §5a #8: Metaculus measured the Platt gain at ≥500 forecasts per bot. Until our forward record is near that size, use a fixed transform (identity, or AIA's d=√3 on an averaged ensemble) rather than a fitted one.
- §5a #14: the ~350 figure assumes mean |b−p| = 0.15. Recompute it from our real |q−m| distribution once Phase F has data.
- §1 #8 and §4: ryanfrigo's Brier is computed on n=43, and the account mixes in manual trades. Cite it as "no edge on the scored subset", not as strategy P&L.
