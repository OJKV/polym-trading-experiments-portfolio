# Strategies

> Canonical. Supersedes `docs/design/03-strategies.md` (frozen input). Evidence lives in
> [`research/03`](research/03-strategies-that-work.md) (strategy cards, wallet evidence) and
> [`research/02`](research/02-llm-forecasting-agents.md) (LLM agents). IDs are identities, not
> ranks; pack IDs P1–P5 are kept. Every strategy implements the interface in ARCHITECTURE §7 and
> goes paper → own gate → live. Nothing here has a forward record yet.

**Rules for every strategy.** Positions are binary and sized as a total loss. Maker-first execution. Hold to resolution unless the exit rule fires. PnL is always reported after fees, spread, LLM cost and data cost. Thresholds are pre-registered before data exists.

## Pipeline

| ID | Strategy | LLM role | State | Fit (research/03) |
|---|---|---|---|---|
| **P1** | Market-anchored LLM forecast on long-tail markets | core | **build now** (Phase 2) | 3 — the thesis |
| **P6** | Weather ensembles, maker-only | assist (rules → typed spec) | **paper in Phase 2** (D-022) | 4 — best verified record |
| P3 | Neg-risk sum-of-asks observer | none | observe-only from Phase 1 | 2 |
| P2 | Rules-lawyer + near-resolution bonds (absorbs pack P5) | core | design ready; offline study first | 4 |
| P4 | Liquidity rewards in uncontested markets | assist | measure first (read-only recorder) | 3 → 4 if measured |
| P7 | Mention markets (transcript base rates) | core | backlog | 3 |
| P8 | Skilled-wallet flow signal (a *feature* for P1/P6) | none | backlog | 2–3 |
| P9 | UMA dispute plays (P2 sub-mode) | core | backlog, log-only | 2 |
| P10 | Sports rating / odds-consensus model (NFL, CFB, MLB, NHL, top soccer) | none (assist: rules) | backlog; paper after P6 | 3 (strong free data, research/09) |
| P11 | Esports rating model (LoL, CS2, Dota 2, Valorant) | none | backlog | 2–3 (largest sport by volume; data thinner) |
| P12 | Crypto price-ladder model from options-implied distributions | none | backlog | 2–3 (keyless Deribit/Binance; 5/15-min up/down excluded) |
| — | Not pursued: cross-venue arb, pro market making, sports/esports in-play, crypto up/down, news-speed | — | rejected at this budget | 1 |

---

## P1 — Market-anchored LLM forecast on long-tail markets (build now)

**Hypothesis.** On thin, text-heavy markets that few informed traders read, a calibrated LLM ensemble blended with the market price beats the market price alone by enough to pay spread and costs. Evidence is weak: an LLM alone loses to liquid markets; blending in code beats the market slightly ([research/02](research/02-llm-forecasting-agents.md)). This is a test, not an assumption.

**Universe (scanner filters).**
- Binary markets only in v1 (neg-risk legs need joint forecasting; backlog).
- Resolution in 3–45 days; scoring prefers ≤ 21 days for faster evidence.
- Long tail by **volume and text-heaviness, not by fee category**: geopolitics is fee-free but liquid (median event volume ~$164k, research/03 §6), so it is a slice, not the universe.
- Depth at our price for ≥ 2× the 5-share minimum.
- Exclude (deterministic rules in `config/areas.yaml`, plus labels from Jev or the owner once D-035 is approved): areas owned by other strategies (sports and esports → P10/P11, crypto price ladders → P12, weather → P6), single-decider / insider-driven markets (awards, military operations, one-person corporate decisions, documented insider cases), subjective resolution sources, `restricted`. **v2 markets are included** in scoring and paper; live trading of them waits for the canary (D-030), and `version` is recorded as a stratum.
- **Slices** tracked separately in every report: geopolitics, politics, non-English-source markets, owner niche (D-019, when named).

**Forecast, calibration, blend, re-forecast triggers and capacity:** exactly as in [ARCHITECTURE §8](ARCHITECTURE.md) (one calibrator on the aggregate; `w_prior` fixed per RISK §2; gated cohorts pinned).


**Entry and exit:** maker-first entry per [ARCHITECTURE §9](ARCHITECTURE.md) (non-crossing maker price, taker only when the fee-adjusted edge clears); hold to resolution, and exit only by the ARCHITECTURE §11.4 rule (fee-inclusive, two consecutive forecasts, `min_hold_hours`). All values live in RISK §2.


**Controls (stored in the `controls` table, never traded):**
1. **Permutation null.** Shuffle the cohort's calibrated LLM probabilities across markets (same dispersion), re-run the full pipeline and the same fill model, 1,000 reps. This gives a p-value for the observed paper PnL. It replaces the pack's "mid + noise" control, which the blend shrinks to ~0, so beating it proves nothing.
2. **"Nothing ever happens":** always NO below 65¢ (research/03 baseline; ~73% of markets resolve NO, unverified).
P1 must beat both (BUILD_PLAN §G2), not just the market.

**Gate.** G2 as pre-registered in [BUILD_PLAN §Gate G2](BUILD_PLAN.md). Numbers are defined only there.

**Kill.** Any G2 failure; paper PnL ≤ 0 after all costs; ECE worse than market; three re-tuning cycles on the same data (R10).

---

## P6 — Weather ensembles, maker-only (paper in Phase 2)

**Hypothesis.** Bucket probabilities from bias-corrected ensemble forecasts beat crowd prices on daily temperature markets in the T−48 h to T−12 h window. That window is slower than the final-hours nowcast race. Evidence: three verified weather wallets made +$88k to +$317k, **but the verified small-ticket earner traded observation-day nowcasts** (92% of its London buys were on the day), so its record does not transfer to our window. Paper logs both horizons (below). Two of them went flat after the spring 2026 fee and CLOB changes, while the top 50 still made $412k in the last 30 days (research/03 §H). We enter late, so the test must prove the edge still exists.

**Universe.** Daily-high and daily-low temperature events (neg-risk sets of ~11 buckets) in cities whose resolution station and data source are machine-readable (e.g. NYC = NOAA LaGuardia hourly max; Hong Kong = HKO Daily Extract). There are ~367 active weather events, median event volume $1.5k.

**Rules spec.** The LLM parses each market's rules once per city template into a typed spec: station, unit, rounding, time window, "no data" fallback. A human checks each new city template once. Code uses only the typed spec.

**Horizon (unresolved in research).** The verified earner bought on the observation day (a nowcast), while our plan is T−36 h / T−12 h. That evidence does not transfer. Paper logs all three horizons; only the primary horizon (T−12 h) is gated.

**Model.**
1. Ensemble members: NOAA GEFS (31 members, public domain) and ECMWF ENS (51 members). Source licensing is an **open decision**: Open-Meteo's free API is non-commercial-only, and personal trading may count as commercial. NOAA NOMADS is public domain.
2. Station bias correction from the resolution station's own history (observations from NOAA/METAR/HKO).
3. P(bucket) = smoothed share of corrected members in the bucket (kernel dressing), then calibrated on our forward record.
4. Fair value per bucket = `blend(model, market)` with weight learned like P1 (prior 0.5: the model is not an LLM, so its prior weight differs; pre-registered).

**Sizing.** Every bucket intent carries `cluster_id` = city-day, so mutually exclusive buckets are sized together at their worst case (ARCHITECTURE §9).

**Execution.** Post-only maker only. The weather fee rate is 0.05 with a 25% maker rebate, and taker fees hurt cheap tail buckets most (4.75% at 5¢). Clips of $1–5. Caps per city per day and per heatwave cluster, because model error is correlated across cities in one weather regime.

**Gate (own G2, pre-registered before paper starts).** Primary horizon T−12 h; T−36 h and observation-day are exploratory. N comes from a P6 power simulation run before paper starts (default ≥ 300 resolved events, ≈ 3–4 weeks at ~15 events a day). The market-implied distribution is normalised to sum to 1 across the event's buckets before scoring. City templates are verified with `pythia weather verify <template>` (owner step):
- multiclass Brier / ranked probability score vs the market-implied distribution, paired bootstrap CI entirely below 0;
- paper PnL after fees and rebates > 0 under the trade-through maker fill model;
- maker fill rate and post-fill drift reported.

**Kill.** Gate failure, or the edge decays below costs for 4 consecutive weeks once live.

**Build effort.** M (1–3 weeks) on the shared core.

---

## P3 — Neg-risk sum-of-asks observer (observe-only, Phase 1)

In a neg-risk event exactly one YES pays $1. Log every moment where Σ(best YES asks) plus fees < 1, with executable depth and duration. On 2026-10-07 only 2 of 279 multi-outcome events cleared fees, at dust depth (research/03 §B). **Promote only if** ≥ $5 is executable at ≥ 2% net within 30-day resolution, and the episode outlasts our measured order round trip. Otherwise it stays a free data feed.

## P2 — Rules-lawyer and near-resolution bonds (design ready; offline study first)

- **(a) Resolved in substance.** The event has happened per allowlisted sources, but the market trades below ~0.97 while resolution is pending.
- **(b) Title-vs-rules mismatch.** The rules or resolution source say something different from the title.
- **LLM role:** reading comprehension. It returns RESOLVED_IN_SUBSTANCE or AMBIGUOUS with quoted clauses.
- **First experiment ($0):** an offline Gamma + prices-history study of ≥ 90¢ prices at T−24 h/6 h/1 h (in-sample before 2026-01-01, out-of-sample untouched), plus a forward log of the LLM's calls from P1's forecasts.
- **Evidence:** +0.83¢ per $ pooled before fees on ≥ 90¢ contracts (Cardozo & Rivero-Wildemauwe); sign fragile.
- **Gate:** ≥ 98% hit rate on RESOLVED_IN_SUBSTANCE calls (each loss is ~−100% against ~+3% gains).
- **Caps:** 5% equity for (b); any single loss triggers a mandatory review.
- **Main risk:** the UMA oracle.

## P4 — Liquidity rewards in uncontested markets (measure first)

18,477 markets pay ~$162k/day in total. In the median rewarded market nobody quotes inside the reward band (research/03 §D), so the pack's "cents per day" verdict assumed crowded markets.
- **First experiment ($0):** a read-only minute-level book recorder on ~50 reward markets for 2 weeks. Simulate our score share and the fills we would have taken, and mark them to market 1 h later.
- **Then:** $20 live on 2 markets.
- **Needs:** a 1-minute loop (owner question) and post-only GTD re-quoting. The LLM screens markets for news risk.
- **Risk:** adverse selection on news.

## P7 — Mention markets (backlog)

"Will X say Y" markets. Build a transcript corpus for 3 recurring speakers. Forecast from base rates with LLM phrase matching against the rule exclusions. 4-week forward log before any money.

## P8 — Skilled-wallet flow signal (backlog, feature)

Replicate the sign-randomisation skill test on Data API histories. If the net flow of skilled wallets predicts 24 h price moves in our universe, add it as an input feature to P1/P6. It is never traded alone (copy-trading as a strategy is rejected).

## P9 — UMA dispute plays (backlog, log-only)

Log every disputed market: price at dispute, final ruling, the LLM's predicted ruling. Trade only with a long forward record under P2's caps.

## P10 — Sports rating / odds-consensus model (backlog)

Pre-game only, never in-play. Fair value comes from open rating data and bookmaker consensus:
- **NFL:** nflverse (CC BY, lines back to 1999).
- **College football:** CollegeFootballData (free key).
- **MLB:** StatsAPI (non-commercial terms, owner decision).
- **Top soccer leagues:** football-data.co.uk historical odds plus CC0 fixtures.
- **Odds consensus:** The Odds API free tier. Pinnacle's public API closed on 2025-07-23, so de-vig a multi-book consensus instead.

A deterministic backtest on historical odds and results **is** valid here, because no LLM is involved, so the gate is an out-of-sample backtest followed by a paper forward test. Fees are 0.03–0.05 on sports, so maker-only. Details in [research/09](research/09-areas-and-sources.md) §3.

## P11 — Esports rating model (backlog)

Esports is 16% of Polymarket volume ($6.30M per 24 h), the most of any sport.
- **Data:** PandaScore (free token), Leaguepedia (bot password), OpenDota.
- **Excluded sources:** Riot's API (its terms ban betting products), HLTV and vlr.gg (no API).
- **Model:** team ratings (Elo/Glicko) from match history → match-winner fair value.
- **Same gate path as P10.**

## P12 — Crypto price-ladder model (backlog)

"Will BTC be above X on date D" ladders, priced from the options-implied distribution (Deribit public data) and spot (Binance, Coinbase, Kraken; keyless). This is deterministic, and the backtest is valid because the data is historical prices. The 5- and 15-minute up/down markets stay excluded (latency, fees).

## Area coverage (D-037)

All 29 areas in [research/09](research/09-areas-and-sources.md) are **tagged and scored** from Phase 1. Each area trades only through a strategy whose gate has passed:

| Areas | Strategy |
|---|---|
| US and world politics, geopolitics, economy events, tech/AI, culture long tail, science, health, finance events | **P1** (LLM on evidence) |
| Weather, disasters | **P6** |
| NFL, CFB, NBA, MLB, NHL, soccer, tennis, golf, F1, MMA, cricket | **P10** (backlog) |
| Esports | **P11** (backlog) |
| Other sports (table tennis, darts, rugby, chess, cycling, Olympics; area 15) | **P10** (backlog), never P1 |
| Crypto price ladders | **P12** (backlog) |
| Mention and post-count markets | **P7** (backlog) |
| Insider-contaminated markets (documented: Nobel Peace Prize Oct 2025 leak, Google Year in Search CFTC/SDNY case May 2026) and single-decider markets | **excluded** |

## Rejected at this budget (revisit only with a reason)

| Strategy | Why not |
|---|---|
| Crypto 5/15-min up/down | Famous bots earned ~$0/month after spring 2026: fees, Chainlink TWAP resolution, taker delay (research/04) |
| Sports / esports in-play, news-speed | Latency races and paid feeds; an hourly LLM loop cannot win |
| Professional two-sided market making | Needs a book-level engine (NautilusTrader 2.x) and > $1–2k |
| Copy-trading | Survivorship bias; followers pay the spread the leader collected; kept only as feature P8 |
| Holding rewards | 3.25% APR carry; only a tie-break term in sizing |
