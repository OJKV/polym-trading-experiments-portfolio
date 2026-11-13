# 03 — Polymarket strategies that actually work (evidence review, 2025–2026)

> Research track, written 2026-10-07. Read-only: public APIs (Gamma, Data API, CLOB GET) and web pages only. No logins, no orders, and no third-party code was run.
> Labels: **[V]** = verified today against a primary source or public API. **[D]** = derived (my arithmetic on cited numbers). **[U]** = unverified or claimed (secondary source, vendor, or self-report).
> All access dates are **2026-10-07** unless stated. Source keys `[S#]` resolve in §8.
> Scope: our constraints are US$50–150, free-tier LLMs, Python, a remote isolated server, and an hourly decision cadence.

---

## 0. TL;DR

1. **Fees changed the game in 2026.** Taker fees now apply to every category except Geopolitics. The fee is `C·rate·p·(1−p)`. As a share of notional that is **rate·(1−p)**, so buying cheap tails costs the most: **4.75% at 5¢, but 0.25% at 95¢** (weather/other rate 0.05) [S1][D]. Makers pay nothing and earn rebates. **Our default execution should therefore be maker-only (post-only) limit orders.**
2. **The well-known speed bots are dead or dying.** The crypto up/down bots **0x8dxd** (+$2.05M lifetime) and **gabagool22** (+$813k) earned almost nothing after March–April 2026 [S13][V]. That followed taker fees on all crypto markets (Mar 6), Fee Structure V2 (Mar 30), CLOB V2 (Apr 28), TWAP resolution (Aug 7) and a 150 ms taker delay (Sep 4) [S2][V]. Do not build speed strategies.
3. **Weather has the best evidence for a small, slow bot**, but it is decaying.
   - The owner's own reference bot, `@automatedaitradingbot`, is a weather bot: +$88k on $3.49M volume. Its **last trade was 2026-07-14**, and its PnL has been flat since May 2026 [S13][V].
   - **ColdMath** made +$137k, flat since May, and **now trades sports** [V].
   - The weather category still paid its top 50 a combined **$412k in the last 30 days**. Rank #10 made $11.9k [S12][V]. Edge remains, but the easy money has gone.
4. **Liquidity rewards look under-rated in our design pack.**
   - [V] 18,477 markets carry rewards, totalling **$162k/day**. The median market pays $3/day with a 20-share minimum [S6].
   - [D] My crude sample of order books suggests that one minimum-size two-sided quote captures **≈$0.9–1.3/day gross** in low-competition markets. That figure ignores adverse selection, so measure before believing it.
5. **Arbitrage is mostly gone at our scale.**
   - [V] Saguillo et al. measured $39.6M of arbitrage, but Apr 2024–Apr 2025 was before fees. 73% of it ($29.0M) came from neg-risk rebalancing, and the top 10 accounts took more than $7.2M [S15].
   - [V/D] In today's live scan, only **2 of 279** neg-risk events cleared fees at top of book. Those had depths of 2–16 shares and were long-dated.
   - [D] Intra-market "YES+NO<1" is impossible at a single moment, because the book is unified (`NO ask = 1 − YES bid`).
6. **Skill exists and persists, but leaderboard PnL is a bad proxy for it.** Only **3.14%** of accounts are "skilled", and 44% of those stay skilled out of sample. Yet only 12% of top earners fall in the skilled group [S17][V via The Block]. So copy-trading by PnL is weak. A skill-filtered wallet signal could feed P1.
7. **LLMs have a structural edge in four places, not in speed:**
   - reading rules and resolution sources (P2, disputes, weather station rules, matching contracts across venues);
   - non-English and regional sources;
   - breadth (screening thousands of thin markets: 74% of esports events and about 70% of closed markets traded under $10k) [S14][S26];
   - transcript base-rates for mention markets.
8. **Recommended order.**
   - Primary strategies: **P1 (LLM long-tail)**, **P2 (rules lawyer and near-resolution bonds)** and **W (weather ensembles, maker-only)**.
   - Measure-only scanners, all free: **R (reward farming)**, **N (neg-risk)** and **C (skilled-wallet signal)**.
   - Everything speed-based is **never**.

---

## 1. Platform facts that every strategy depends on (state on 2026-10-07)

| Fact | Value | Src |
|---|---|---|
| Taker fee | `fee = C × rate × p × (1−p)`. Makers pay 0. Rates: Crypto 0.07; Sports 0.05; Econ/Culture/Weather/Other 0.05; Politics/Finance/Mentions/Tech 0.04; **Geopolitics 0 (fee-free)** | [S1][V] |
| Fee as % of notional | `rate × (1−p)`. At rate 0.05: 4.75% @5¢, 2.5% @50¢, 0.25% @95¢. At 0.04: 3.8% / 2.0% / 0.2%. At 0.07: 6.65% / 3.5% / 0.35% | [D] |
| Fee timeline | 15-min crypto (2026-01-05); 5-min crypto (02-12); NCAAB/Serie A (02-18); all crypto (03-06); **Fee Structure V2, all categories (03-30)**; Sports 0.03→0.05 (07-10) | [S2][V] |
| Maker rebates | Paid daily from taker fees, weighted by the fee curve. 20% Crypto, 15% Sports, 25% others. $1 minimum payout | [S3][V] |
| Taker rebate tiers | Live 2026-05-28. Bronze needs $2k "weighted volume" in 30 days for 3%; Obsidian needs $10M for 50%. **Irrelevant at our size** | [S4][V] |
| Liquidity rewards | Score `((v−s)/v)²` summed over size. Two-sided quotes are favoured: one-sided counts ÷3 inside mid 0.10–0.90, and only two-sided counts outside it. Sampled every minute at a random offset, 1,440 samples/day. $1 minimum payout | [S5][V] |
| Reward universe today | **18,477 markets, $162,401/day** configured. Median $3/day, p90 $17/day, 9 markets above $1k/day. Min size 20 shares in 79% of markets. Max spread 4.5¢ in 79% | [S6][V]; [D] ≈ $4.9M/month |
| Delays | Crypto taker delay **150 ms** since 2026-09-04 (was 250 → 50 → 150). Sports have a "game delay" window near live play. Orders cannot be cancelled while delayed | [S2][S9][V] |
| CLOB V2 | Live 2026-04-28. pUSD collateral, new exchange contracts, V1 SDKs dead, `py-clob-client-v2` | [S2][V] |
| Crypto up/down resolution | Chainlink TWAP since 2026-08-07: 30 s window on 5-min markets, then 60 s from 08-14; 60 s on 15-min and 4-hour markets. No UMA | [S2][S7][V] |
| UMA resolution | Bond "typically $750". 2 h challenge. Disputed markets take 4–6 days. Can resolve 50/50. Proposers have been whitelisted since UMA's MOOV2 change (Aug 2025), so **we cannot be proposers** | [S7][V]; [S32] |
| Holding rewards | 3.25% APR (variable) on 17 listed events. Position sampled hourly, paid daily | [S11][V]; [D] $100 earns ≈ $3.25/year |

The design consequence: **post-only maker orders are the default for every directional strategy.** Taking liquidity is allowed only when `edge > rate·(1−p) + slippage + margin`.

---

## 2. Who actually makes money: verified wallet evidence

All rows come from the public Data API (`/v2/user-stats`, `/v2/user-pnl`, `/activity`) and the leaderboard API [S12][S13]. Margin = PnL ÷ volume [D].

| Wallet (niche) | Lifetime PnL | Volume | Margin | Fees paid / maker rebate | Status on 2026-10-07 [V] |
|---|---|---|---|---|---|
| `automatedaitradingbot` 0xd8f8… (weather; owner's link) | **+$88.4k** | $3.49M | 2.5% | $672 / $89 | **Last trade 2026-07-14.** Bought cheap buckets (median fill price 4¢) in HK, Shenzhen, Seoul and London highs |
| ColdMath 0x594e… (weather → sports) | +$137k | $13.3M | 1.0% | $8.6k / $765 (mostly taker) | Active, but **now trades sports** |
| gopfan2 0xf2f6… (weather #1, then politics) | +$1.68M overall; **+$317k in weather** | $70.4M / $4.6M | 2.4% / 6.9% | $3.4k / $2.4k | Active. Recent trades are mostly **Russian Duma ("United Russia")**, NE independents and the Fed |
| 0x8dxd 0x63ce… (crypto 15-min) | +$2.05M | $200.6M | 1.0% | $322k / $71k | Last trade 2026-08-24. Flat since April |
| gabagool22 0x6031… (crypto both-sides) | +$813k | $144.8M | 0.56% | $55k / $50.5k | **Last trade 2026-02-20** |
| gabigol 0x8852… (crypto micro-clips) | +$351k | $31.3M | 1.1% | $16 / $34.8k | Active. +$4–11k/month since August |
| TeemuTeemuTeemu 0x5388… (esports in-play) | +$230k | $5.8M | 4.0% | n/a | Stats snapshot dated 2025-12-31, so it looks inactive |
| bama124 (mentions #1) | +$293k in mentions | $1.33M | 22% | n/a | Inactive since Feb 2025 |
| swisstony (sports #1) | +$18.4M | $1.85B | 1.0% | n/a | Large-scale sports market making or execution |

**Monthly PnL in $k, from end-of-month `economic_pnl` points [S13][D]:**

| Wallet | 2025-12 | 2026-01 | 02 | 03 | 04 | 05 | 06 | 07 | 08 | 09 |
|---|---|---|---|---|---|---|---|---|---|---|
| automatedaitradingbot | +11 | +10 | +8 | +6 | +8 | +1 | 0 | 0 | 0 | 0 |
| ColdMath | +10 | +13 | +29 | +41 | +44 | −3 | −2 | −1 | 0 | +6 |
| 0x8dxd | +322 | +536 | +772 | +503 | +25 | +3 | +7 | +5 | −1 | +1 |
| gabagool22 | +300 | +270 | +154 | +1 | 0 | 0 | 0 | 0 | 0 | 0 |
| gabigol | +1 | +2 | +16 | +78 | +110 | +41 | +53 | +30 | +4 | +11 |
| gopfan2 (all categories) | −121 | +15 | +215 | +238 | +74 | −34 | +63 | −108 | +21 | +163 |

**What the population data says:**

- **Skill is real but rare.** In 1.72M accounts, 210k markets and $13.76B of volume (2023–2025), 3.14% of accounts are "skilled winners" and 67% are losers. Skilled traders plus market makers take more than 30% of gains. Of those skilled in the training sample, 44% stay skilled out of sample (about 10% for mutual funds). Only about 12% of top PnL earners are skilled [S17][V via The Block, 2026-04-26; SSRN 6617059].
- **Pre-fee crypto 15-min (Feb 2–8, 2026).** Makers won 47% of fills yet netted +$728.5k for the week. Takers won 53% of fills and lost overall [S18][U: third-party analytics].
- **The long tail is bot-heavy, not empty.** About 70% of closed markets traded under $10k, and bots made more than 80% of activity in those markets. More than 45k markets never traded [S26][U: CNBC via Tangem].
- **Last-30-day leaderboards** [S12][V]:

| Category | Top-50 total | #1 | #10 | #25 | #50 |
|---|---|---|---|---|---|
| Weather | $412k | $43k | $11.9k | $4.8k | $2.9k |
| Mentions | $86k | $8.5k | $2.2k | n/a | $0.45k |
| Culture | $409k | $34.9k | $10k | n/a | $3.5k |
| Politics | $4.29M | $511k | $103k | n/a | $34.6k |
| Crypto | $4.93M | n/a | $127k | n/a | $39k |

These boards show winners only (survivorship bias); the losers are not listed.

---

## 3. Strategy cards

Fit is scored 1–5 against our constraints: $50–150, free LLMs, Python, remote isolated server, hourly cadence.

### A. Intra-market arbitrage (YES + NO < $1)

| Item | Finding |
|---|---|
| Mechanism | Buy YES and NO for less than $1 combined, then merge or redeem. |
| Evidence | $10.58M realised Apr 2024–Apr 2025, pre-fee: $5.90M from buying both below $1 and $4.68M from selling both above $1 [S15][V]. |
| Today | The book is unified, so `best NO ask = 1 − best YES bid`, and YES ask + NO ask = 1 + spread ≥ 1 [D, from CLOB mechanics]. Simultaneous arbitrage cannot happen. The surviving variant is **legging in** (gabagool-style): rest maker bids on both sides and get filled at different moments. That is inventory risk, not arbitrage. gabagool22 stopped on 2026-02-20 [V]. Open-source "gabagool" clones exist [U]; treat any repo of this kind as possible key-stealing malware. |
| Competition / capital / fees | Sub-second bots. Fees now apply per leg. |
| Fit | **1.** Never. |

### B. Neg-risk multi-outcome rebalancing

| Item | Finding |
|---|---|
| Mechanism | In a neg-risk event, exactly one YES pays $1. If Σ best YES asks + fees < 1, buy the full set. If Σ YES bids > 1, buy the NOs. Any NO converts into YES on all other outcomes [S8]. |
| Evidence | **$29.01M** (73% of all arbitrage), Apr 2024–Apr 2025, almost entirely before fees [S15][V]. The top account made $2.01M; the top 10 made more than $7.2M [V]. NBA, Feb–Mar 2026: 7 executable single-market episodes with a median life of 3.6 s (via design pack [P1]). |
| **Live scan today** [V/D] | Of 279 complete neg-risk events among the top 1,000 by 24h volume: median Σ asks 1.089; **6 had Σ asks < 1, and only 2 after fees**. Both were dust: *Maduro Prison Time?* (Σ 0.886, but one leg only 2 shares deep) and *OpenAI IPO market cap* (Σ 0.968, one leg 10.8 shares deep, resolves in 2027). 8 events had Σ bids > 1, with a maximum of 1.012. |
| Competition | Fast bots. Gaps that survive are thin and long-dated. |
| Capital / fees | The fee is charged on every leg. [D] Example from the design pack: 10 legs cost 4.5¢ per $1 set. Capital stays locked until resolution unless the legs are sold. |
| Data / tech | Free Gamma and CLOB book data. Needs a depth-aware scanner and atomic-ish multi-leg execution. Legs are not atomic. |
| Fit | **2.** Run the hourly scanner as observe-only (cheap). Trade only fee-clearing gaps deeper than $5 that resolve within 30 days. Watch for augmented neg-risk events with "Other" or placeholder outcomes. |
| Ruin risk | Low per trade. A partial fill leaves naked directional legs. |

### C. Cross-venue arbitrage (Polymarket vs Kalshi)

| Item | Finding |
|---|---|
| Mechanism | Buy YES on one venue and NO on the other when they sum to less than 1 after both fees. |
| Evidence | Polymarket leads Kalshi in price discovery, especially when liquidity is high ("economically meaningful arbitrage") [S30][V abstract]. Claimed spreads: 5–10% in 2024, **1–2% now** on major markets [S38][U]. No audited cross-venue PnL was found. |
| Problems | Capital is split across two venues ($25 each at our size). Both venues charge fees. **Resolution sources differ**, so a hedge can become one-sided. Withdrawals between venues are slow. |
| LLM angle | Structural, but defensive: an LLM can check whether two contracts' rules are truly equivalent before any pairing. |
| Fit | **1.** Revisit above $2k, as an observe-only gap logger. |

### D. Market making and liquidity rewards (CLOB V2 era)

| Item | Finding |
|---|---|
| Mechanism | (i) A professional two-sided market maker earns spread plus rebates. (ii) A **reward farmer** rests minimum-size quotes inside the reward band of low-competition markets. |
| Evidence (i) | Sports leaders run near 1% margin on $1B+ volume (swisstony) [V]. Crypto makers netted +$728k in one week pre-fee [S18][U]. warproxxx/poly-maker (1.5k★) warns: "competitive and can lose money" [S31][V]. CLOB V2 launch: $1M rewards program (Apr 2026); March Madness $2M+, with up to **$60k/day** on a live game moneyline [S2][V]. |
| Evidence (ii) [D, crude] | I sampled 47 reward markets, with books read now. Score = Σ size·((v−s)/v)² within the band. Our hypothetical two-sided min-size quote at mid±1¢, share = ours ÷ (all + ours): <br>• $1–5/day markets: median competing score 22, est. **$0.92/day** for about $20 of capital. <br>• $5–20/day: score 451, $1.33/day. <br>• $20–100/day: 2,715, $0.93/day for about $50. <br>• $100+/day: 8,393, $1.02/day for about $75. <br>Taken at face value, that is **1–5% of deployed capital per day**. That is implausible to persist, so expect adverse selection (fills just before news), inventory, the scoring's adjusted midpoint and size cutoffs, and minute-sampling gaps to eat most of it. **Unvalidated.** |
| Competition | Thin in $1–5/day markets, crowded above $20/day. |
| Capital | About $20–75 per market for a two-sided minimum-size quote. $100 covers 2–5 markets. |
| Fees | Makers pay 0 and receive rebates. Rewards are paid daily; the minimum payout is $1. |
| Data / tech | Free WebSocket book feeds. Needs a 1-minute quoting loop, **post-only GTD orders**, the heartbeat endpoint (auto-cancel on disconnect), and per-market news-risk exclusion. That is not an hourly loop, but it is not low-latency either. |
| LLM angle | Assistive: screen out markets with imminent catalysts (scheduled announcements, live games) where quotes get picked off. |
| Fit | **3** (could become 4 after measurement). |
| Ruin risk | Medium. One news fill can lose a resting side. Cap each market at about $25. |

### E. High-probability "bonds" near resolution (and rules-lawyer)

| Item | Finding |
|---|---|
| Mechanism | Buy 90–99¢ outcomes that are decided in substance but not yet settled. Hold to $1. |
| Evidence | 588M trades by 2.48M accounts: purchases at ≥90¢ earn **+0.83¢ per $** pre-fee, while longshots lose about 19.3¢ per $. The sign is sensitive to grouping by event [S16][V abstract]. "Nothing Ever Happens" bot: 73.3% of resolved markets resolve NO. It buys NO below 65¢; **no live PnL disclosed** [S27][U]. A copy-trader reported that copying 99.5¢ buyers failed because the edge is too thin [S28][U]. |
| Fees | [D] At 95¢, the taker fee is 0.20–0.25% of notional, which leaves about +0.6¢ per $ if the paper's average holds. As a maker it costs 0. Annualised return depends on days to resolution. |
| Competition | Many humans and bots. A 99¢ level is crowded; a 90–96¢ level with ambiguous rules is less so. |
| Capital / data / tech | Any size. Free Gamma rules plus `prices-history`. Hourly cadence is fine. |
| LLM angle | **Core and structural.** Decide RESOLVED_IN_SUBSTANCE vs AMBIGUOUS from the rules text, clarifications and the named source, and quote the clauses. |
| Fit | **4.** This is P2 + P5 in the design pack. |
| Ruin risk | Asymmetric: about −100% per loss against +1–5% per win. UMA misresolution or a 50/50 resolution is the dominant tail. The hit-rate gate must be ≥98% [design pack]. |

### F. UMA resolution and dispute plays

| Item | Finding |
|---|---|
| Mechanism | Trade markets during proposal and dispute windows, when prices dislocate. Estimate how the DVM vote will read the rules and clarifications. |
| Evidence | Dispute rate about 0.7–1.2% of proposals [U: UMA-sourced claim via S33]. The $7M Ukraine-minerals case resolved YES without a deal (design pack [S14]). Proposers are whitelisted (MOOV2), so the proposer bond play is closed to us [S32]. The Data API now exposes UMA settlement-time estimates on `/v2/resolutions` (2026-09-28) [S2][V]. |
| LLM angle | **Structural.** Read rules, clarifications and the UMA Discord evidence; predict the voters' reading. |
| Fit | **2.** Rare events, whale-vote risk. Run it as a P2 sub-mode with a hard cap of 5% of bankroll per dispute. |

### G. Copy-trading and whale following

| Item | Finding |
|---|---|
| Mechanism | Mirror wallets that have shown skill, with a delay. |
| Evidence | Skill persists (44%) but PnL rank is a poor proxy (12% overlap) [S17]. A self-reported +$650k in 7 months from copying niche specialists with fewer than 100 bets and an 80–90% hit rate; copying high-frequency traders and 99.5¢ buyers failed [S28][U]. Several paid copy-bot products exist; none publish audited results [U]. |
| Competition | Other copiers (crowding, front-running of followed wallets). Followed wallets can bait copiers. |
| Data / tech | Free: Data API `/activity`, `/positions`, `/v2/user-pnl`. Hourly is fine for slow niche specialists. |
| Fit | **2** as a standalone; **3** as a *feature* for P1: skilled-wallet net flow in a market. |
| Test | Replicate the sign-randomisation skill test on about 500 wallets. Simulate copying with a 1 h delay at recorded asks. |

### H. Weather markets (ensemble forecast vs bucket prices)

| Item | Finding |
|---|---|
| Universe [V] | **367 active weather events**: 146 daily highs, 128 daily lows, the rest climate or disaster. About 51 cities, about 11 buckets per event, all neg-risk. Median event volume **$1.5k**, p90 $41k. Fee rate 0.05 with a 25% maker rebate [S14]. |
| Resolution sources | NYC uses NOAA's LaGuardia station hourly "Temp" maximum (`weather.gov/wrh/timeseries?site=klga`). Hong Kong uses the HKO Daily Extract "Absolute Daily Max", in °C to 0.1 [S14][V]. Units, rounding and station choice differ by city, so read them per market. |
| Evidence | Verified weather PnL: gopfan2 +$317k, ColdMath +$134k, automatedaitradingbot +$88k [S12][S13][V]. Blog lists claim similar traders [S23][U]. **Decay:** two of the three bots were flat from May 2026 and stopped or switched [V], which coincides with Fee V2 (03-30) and CLOB V2 (04-28); cause not proven. The last-30-day top 50 still made $412k [V]. |
| Mechanism | Turn ensemble members (ECMWF IFS 51 members, GFS 31, ICON 40, MOGREPS 18, all free via Open-Meteo [S24]) into bucket probabilities. Correct station bias against the resolution station's history. Compare with prices. Later, use same-day observations (METAR) for the last hours. |
| Competition | Several bots. The final hours are a nowcast speed race, which we skip. The T−48h to T−12h window is slower. |
| Fees | Cheap tail buckets are where taker fees hurt most (4.75% at 5¢) [D]. The reference bot's median fill was 4¢, so post-only limit orders are a must. |
| Capital | Tiny markets suit $5–20 clips. Capacity is low, which is fine for us. |
| Data | Free: Open-Meteo (non-commercial terms, so check whether personal trading counts as commercial), NOAA GEFS/NOMADS (public domain), HKO, METAR. |
| LLM angle | Assistive, not core. Parse each market's rules (station, unit, rounding, "no data" fallback to the lowest bracket) into a typed spec. The forecasting itself is deterministic statistics. |
| Fit | **4.** Best evidence of all the candidates. Runs on an hourly to 6-hourly cadence, matching model updates. |
| Ruin risk | Low per market (independent daily events). Model error is correlated within a heatwave, so cap per city and per day. |

### I. Crypto short-horizon up/down (5-min, 15-min, hourly)

| Item | Finding |
|---|---|
| Evidence | 0x8dxd: claimed to start from $313 in Dec 2025 [S20][U]; verified **+$2.05M lifetime but +$1k to +$25k/month since April** [V]. gabagool22 stopped in Feb 2026. gabigol still earns +$4–11k/month on about 100k micro-clips: a dev.to analysis calls it taker-heavy at 94–99¢ and 3–30¢ [S19][U], but the API shows net fees of $16 and a $34.8k maker rebate, so it may be largely maker [V]. |
| What killed it | Taker fee 0.07 (3.5% of notional at 50¢); TWAP settlement replacing last-tick sniping (a claimed $7.6M lost to manipulation before) [S21][U]; a 150 ms uncancellable taker delay [S2]. |
| Competition | Co-located bots in eu-west-2 [S10]. |
| Fit | **1.** Never at our cadence. |

### J. Sports in-play, latency, and esports

| Item | Finding |
|---|---|
| Evidence | Sports is the largest PnL pool (swisstony +$18.4M at 1.0% margin) [V]. Esports in-play bot TeemuTeemuTeemu: +$230k verified [V]; "$900 → $208k in 3 months" is claimed [S25][U]. Edge comes from game-server data feeds against roughly 30 s stream delays. |
| Barriers | The sports "game delay" window, the 0.05 fee, paid low-latency feeds, and MM rewards concentrated around live games ($60k/day on March Madness moneylines) that pros farm. |
| LLM angle | None. |
| Fit | **1.** Exception: pre-game long-tail esports or lower-league lines could join P1's universe (74% of 754 esports events traded under $10k) [V]. |

### K. News-speed trading

| Item | Finding |
|---|---|
| Evidence | No verifiable retail PnL. The documented big "news" wins are **insider-like**: the Maduro capture (+$400k on a $32k bet, Jan 2026) and the Nobel leak (about $90k across 3 accounts, Oct 2025) [S35][V news]. Polymarket does not ban insider trading, but these cases draw probes. |
| Barriers | Hourly cadence, free LLM latency (seconds to minutes), taker fees. |
| Fit | **1.** Not a strategy for us. The LLM can still *ingest* news within P1 at an hourly cadence. That is slow news, not speed. |

### L. Mention markets ("What will X say…")

| Item | Finding |
|---|---|
| Universe [V] | 21 active events and 336 markets under the `mention-markets` tag, median volume about $4.5k (more exist under other tags). Fee rate 0.04. Example rules exclude background music and recap segments [S14]. |
| Evidence | bama124 made +$293k at a 22% margin, but is inactive since Feb 2025. The current top 50 made $86k in 30 days; #10 made $2.2k [S12][V]. The pool is small but not crowded by large funds. |
| Mechanism | Base rates from past transcripts of the same speaker and format, plus the scheduled context. |
| LLM angle | **Structural:** transcript retrieval, counting of phrase variants, and reading of the rule's exact-term clauses. |
| Fit | **3.** Small capacity suits $50–150. Needs a transcript corpus (YouTube captions, the White House site, CBS). |
| Ruin risk | Per-market binary outcomes; correlated within one speech, so cap per event. |

### M. Holding rewards

| Item | Finding |
|---|---|
| Finding | 3.25% APR on 17 long-dated events [S11][V]. [D] That is about $3/year on $100. |
| Fit | Not a strategy, only a tie-break bonus for long-dated P1 positions. Polymarket flags a "new reward and oracle-resolution system later this year" [S11], so watch the changelog. |

### N. Information-edge niches (local politics, non-English, esports, long tail)

| Item | Finding |
|---|---|
| Evidence | gopfan2's live book is concentrated in Russian Duma ("United Russia") markets [V]. Inference only: regional-language knowledge is part of the edge. Wolfe Research found that Polymarket earnings-miss calls hit 44% against an 18% base rate (Bloomberg, 2026-04-16) [U]. Geopolitics is fee-free, but its median event volume is **$164k** (only 13% under $10k) [V], so it is liquid and efficiently priced. |
| Counter-evidence | LLMs forecast less accurately than markets on liquid questions, and "aggregate information slower than markets when resolution nears" [S34]. Vendor claims (Foresight-32B profitable on 251 live questions [S36], "37% of Polystrat agents profitable") are **[U]**. |
| Fit | **4** as P1's universe filter: thin, text-heavy, non-English, 3–45 days to resolution. |

---

## 4. Where an LLM gives a structural (not cosmetic) advantage

| LLM capability | Strategies | Why code alone can't do it | Evidence strength |
|---|---|---|---|
| Reading rules, resolution sources and clarifications | E (bonds/P2), F (disputes), H (weather spec parser), C (venue equivalence), L (mention exclusions) | Rules are free text, and titles mislead ("rules define how it resolves") [S7] | Strong for need; no audited PnL |
| Non-English and regional sources | N, P1 | Local media reach markets late; few English-speaking bots read them | Anecdotal (gopfan2 [V], inference) |
| Breadth over many thin markets | P1, D (news-risk screen), B (outcome-set completeness check) | 18k reward markets, 7,998 esports markets, ~70% of markets under $10k [V][U] | Strong for opportunity size; bots already make up 80% of tail activity [U] |
| Transcript base rates | L | Fuzzy phrase matching against rule exclusions | Moderate |
| Forecasting itself | P1 | n/a | **Weak.** Leading LLMs trail superforecasters and are overconfident (design pack [L1][L5]). Use the LLM as an *input*, blended with market price |

---

## 5. Ranked backlog

Effort: S under 1 week, M 1–3 weeks, L over 3 weeks, for one developer with the shared core (journal, Gamma/CLOB clients, risk layer) already built.

| # | Strategy | Fit 1–5 | Expected edge source | Needs LLM? | Build effort | What to test first |
|---|---|---|---|---|---|---|
| 1 | **H. Weather ensembles, maker-only** | **4** | Ensemble-calibrated bucket probabilities vs crowd prices, plus station and rounding rules; zero fee as maker | Assist (rules → typed spec) | M | 4-week forecast-only log: P(bucket) from Open-Meteo ensembles at T−36h/T−12h vs mid. Score Brier against market mid. Record whether a post-only bid at our price would have filled (paper fill model) |
| 2 | **E. Rules-lawyer and near-resolution bonds (P2+P5)** | **4** | Settlement lag, ambiguity priced wrong, slight favourite bias (+0.83¢/$ pre-fee) | **Core** | S–M | Offline Gamma study of ≥90¢ at T−24h/6h/1h, split IS (before 2026-01-01) and OOS. In parallel, a forward log of LLM RESOLVED_IN_SUBSTANCE calls with quoted clauses; gate at ≥98% hit-rate |
| 3 | **P1 + N. LLM long-tail forecasting on thin, non-English, text-heavy markets** | **3** (the thesis) | Information the crowd doesn't read; breadth | **Core** | M–L (already designed) | Phase F forward log with two controls: market mid plus noise, and the "always NO below 65¢" prior. Kill per design-pack criteria |
| 4 | **D-ii. Liquidity-reward farming in low-competition markets** | **3** (→4 if measured) | Venue subsidy ($162k/day) in markets where nobody quotes | Assist (news-risk screen) | M | Read-only minute-level book recorder on about 50 reward markets for 2 weeks. Simulate our score share and the fills we would have taken, and mark them to market an hour later. Then go live with $20 on 2 markets |
| 5 | **L. Mention markets** | **3** | Transcript base rates against crowd intuition | **Core** | M | Build a transcript corpus for 3 recurring speakers. Run forward base-rate forecasts on every new mention event for 4 weeks |
| 6 | **G. Skilled-wallet flow signal** (feature for P1/H) | 2–3 | Persistent skill (44%) of a 3% minority | No | M | Replicate the sign-randomisation skill test on Data API histories. Check whether skilled net flow predicts the 24h price move in our universe |
| 7 | **B. Neg-risk scanner** | 2 | Stale legs in thin, long-dated sets | No (LLM only to check outcome-set completeness) | S | Hourly, observe-only, depth-aware log of fee-clearing executable size and duration. Trade only if ≥$5 executable at ≥2% net, resolving within 30 days |
| 8 | **F. UMA dispute plays** (P2 sub-mode) | 2 | Mispriced dispute outcomes | **Core** | S (on top of P2) | Log every disputed market: price at dispute, final ruling, and the LLM's predicted ruling |
| 9 | "Nothing ever happens" NO prior | 2 | NO-resolution base rate (73.3%) [U] | No | S | Offline only, as a **baseline that P1 must beat** |
| 10 | C. Cross-venue Polymarket↔Kalshi | 1 | Lead-lag gaps of 1–2% [U] | Assist (rules equivalence) | L | Nothing until bankroll exceeds $2k. Optionally a matched-pairs gap logger |
| 11 | D-i. Professional two-sided market making | 1 | Spread + rebates at scale | No | L | Defer (design pack: NautilusTrader, more than $1–2k) |
| 12 | J. Sports or esports in-play | 1 | Data-feed latency | No | L | Never at this budget and cadence |
| 13 | I. Crypto 5/15-min up/down | 1 | Was latency; now fees + TWAP + 150 ms delay | No | L | Never |
| 14 | K. News-speed | 1 | Speed or insider information | No | L | Never |
| 15 | M. Holding rewards | n/a | 3.25% APR carry | No | S | Add as a tie-break term in P1 sizing only |

---

## 6. What this changes in the design pack (`docs/03-strategies.md`)

1. **Add Weather (H) as a first-wave strategy** next to P1/P2. It is deterministic at its core, uses free data, and has the strongest verified PnL record for a slow bot. The decay data means we enter late, so the test must prove the edge still exists.
2. **Make post-only maker execution the default** for every strategy. Allow taker orders only when the fee-adjusted edge clears. Fee as % of notional is `rate·(1−p)`, which punishes cheap-tail buys.
3. **Re-rate P4 (liquidity rewards) from "poor, cents/day" to "measure first".**
   - Today's data: in the median rewarded market, nobody quotes inside the band.
   - The measurement is read-only and free.
   - The earlier verdict assumed a share of crowded markets.
4. **Geopolitics fee-free ≠ long-tail.** Geopolitics events are liquid (median $164k), so P1's long-tail universe must be chosen by volume and text-heaviness, not by fee category alone.
5. **The P1 control set gains a second baseline:** the "always NO below 65¢" prior, alongside market-mid-plus-noise.
6. **Re-verify platform facts monthly.** Fees, delays, rewards and the oracle changed at least nine times in 2026 [S2]. Add a monthly job that diffs the changelog.

---

## 7. Questions for the owner (to grill)

1. **Weather first or P1 first?** Weather has better evidence and needs a less capable LLM, but its edge is decaying and its capacity is small. P1 is the thesis but has weak evidence. Proposal: run both in Phase F in parallel, since the forecast-only logs cost $0.
2. **Is a 1-minute loop acceptable for reward farming?** It breaks the "hourly cadence" assumption, though not the "no low-latency" one.
3. **Which languages and regions can you sanity-check by hand** (Russian? Kazakh?)? This decides the N sub-universe.
4. **Open-Meteo licensing:** personal trading may count as "commercial". Fall back to NOAA GEFS/NOMADS (public domain), or pay for Open-Meteo, or accept the risk?
5. **Accept insider-adjacent markets?** Some profitable "news" markets are driven by insiders. Proposal: exclude single-decider markets (awards, military operations) from P1, because we will be on the wrong side of insiders.

---

## 8. Sources (accessed 2026-10-07)

| Key | Source |
|---|---|
| S1 | Polymarket Fees — https://docs.polymarket.com/trading/fees |
| S2 | Polymarket Changelog — https://docs.polymarket.com/changelog |
| S3 | Maker Rebates Program — https://docs.polymarket.com/programs/maker-rebates |
| S4 | Taker Rebate Program — https://docs.polymarket.com/programs/taker-rebates |
| S5 | Liquidity Rewards — https://docs.polymarket.com/programs/liquidity-rewards |
| S6 | CLOB rewards API (paged in full) — https://clob.polymarket.com/rewards/markets/current ; books via https://clob.polymarket.com/book |
| S7 | Resolution — https://docs.polymarket.com/concepts/resolution |
| S8 | Negative Risk — https://docs.polymarket.com/concepts/negative-risk |
| S9 | Order lifecycle — https://docs.polymarket.com/concepts/order-lifecycle |
| S10 | Geoblock — https://docs.polymarket.com/api-reference/geoblock |
| S11 | Holding Rewards — https://help.polymarket.com/en/articles/13364459-holding-rewards |
| S12 | Leaderboard API — https://data-api.polymarket.com/v1/leaderboard?category=WEATHER&timePeriod=MONTH&orderBy=PNL&limit=50 (and other categories) |
| S13 | Wallet data — https://data-api.polymarket.com/v2/user-stats , https://data-api.polymarket.com/v2/user-pnl , https://data-api.polymarket.com/activity . `automatedaitradingbot` wallet 0xd8f8c13644ea84d62e1ec88c5d1215e436eb0f11, inferred from https://polymarket.com/@automatedaitradingbot page data |
| S14 | Gamma events — https://gamma-api.polymarket.com/events (tags `weather`, `esports`, `geopolitics`, `mention-markets`; top 1,000 by 24h volume) |
| S15 | Saguillo, Ghafouri, Kiffer, Suarez-Tangil, "Unravelling the Probabilistic Forest" — https://arxiv.org/html/2508.03474 |
| S16 | Cardozo & Rivero-Wildemauwe, "The Favorite-Longshot Bias in Prediction Markets: Evidence from Polymarket" — https://arxiv.org/abs/2609.12878 |
| S17 | Gómez-Cram, Guo, Kung, Jensen (LBS/Yale), via The Block 2026-04-26 — https://www.theblock.co/post/398902/skilled-polymarket-traders-are-a-3-minority-and-everyone-else-funds-their-gains-study ; paper https://ssrn.com/abstract=6617059 |
| S18 | Telonex, top crypto traders 15m (2026-02-13) — https://telonex.io/research/top-crypto-traders-polymarket-15m [U] |
| S19 | dev.to, "Reverse-engineering gabigol" (2026-07-10) — https://dev.to/fatherson/reverse-engineering-gabigol-high-frequency-crypto-updown-farmer-on-polymarket-d0a [U] |
| S20 | BeInCrypto, arbitrage bots (0x8dxd $313 claim) — https://beincrypto.com/arbitrage-bots-polymarket-humans/ [U] |
| S21 | casatrick, "TWAP fixes latency is a half-truth" (2026-08-03) — https://casatrick.substack.com/p/polymarket-twap-latency-trading-bots [U] |
| S22 | The Block, taker fees on 15-min crypto (2026-01-06) — https://www.theblock.co/post/384461/polymarket-adds-taker-fees-to-15-minute-crypto-markets-to-fund-liquidity-rebates |
| S23 | dev.to, top weather traders — https://dev.to/xniiinx/my-top-7-most-profitable-weather-market-traders-on-polymarket-gb7 [U] |
| S24 | Open-Meteo Ensemble API — https://open-meteo.com/en/docs/ensemble-api |
| S25 | esports.net, TeemuTeemuTeemu (2025-12-30) — https://www.esports.net/news/polymarket-bot-makes-over-200k-in-3-months-with-lol-dota-2-esports/ [U] |
| S26 | Tangem summarising CNBC (2026-07-03) — https://tangem.com/en/news/markets/30950-45-000-polymarket-markets-see-zero-trades-activity-clustered/ [U] |
| S27 | Decrypt, "Nothing Ever Happens" bot (2026-04-15) — https://decrypt.co/364381/polymarket-bot-bets-no-has-a-point [U] |
| S28 | Blockonomi, $650k copy-trading report — https://blockonomi.com/crypto-trader-reports-650000-profit-through-polymarket-copy-trading-strategy/ [U] |
| S29 | Cointelegraph, Kalshi 140 countries — https://cointelegraph.com/news/kalshi-300-million-raise-prediction-markets-140-countries/ [U] |
| S30 | Ng, Peng, Tao, Zhou, "Price Discovery and Trading in Modern Prediction Markets" — https://papers.ssrn.com/abstract=5331995 |
| S31 | warproxxx/poly-maker README — https://github.com/warproxxx/poly-maker |
| S32 | The Block, UMA MOOV2 whitelisted proposers — https://www.theblock.co/post/366507/polymarket-uma-oracle-update |
| S33 | UMA dispute-rate claims (aggregator) — https://www.mexc.com/news/1192167 [U] |
| S34 | Yang et al., "LLM-as-a-Prophet / Prophet Arena" — https://arxiv.org/abs/2510.17638 |
| S35 | Maduro trade — https://www.ctpublic.org/2026-01-05/a-400-000-profit-on-maduros-capture-raises-insider-trading-questions-on-polymarket ; Nobel probe — https://www.theblock.co/post/374199/officials-probe-polymarket-over-suspicious-trades-predicting-nobel-peace-prize-winner-reports |
| S36 | Lightning Rod (vendor) Foresight claims — https://www.lightningrod.ai/solutions/prediction-markets [U] |
| S38 | NYC Servers arbitrage guide (spread claims) — https://newyorkcityservers.com/blog/prediction-market-arbitrage-guide [U] |
| P1 | Design pack `docs/03-strategies.md` (NBA arbitrage episodes, fee-per-leg example, kill criteria) |

**Method notes and caveats.**

- **Wallet PnL** is the Data API's `economic_pnl`. Monthly values are differences of the last daily point in each month.
- **Reward-share estimates** come from one snapshot of 47 books. My score omits the size-cutoff-adjusted midpoint and the `min(Q_one, Q_two)` two-sidedness. They are an upper-bound sketch, not a forecast.
- **Gamma `bestAsk`** can lag. Neg-risk candidates were re-checked against live `/book`.
- **Leaderboards** list winners only (survivorship bias).

---

## Verification (2026-10-07)

This is an adversarial re-check of 10 claims; it replaces this section from the earlier run. For each claim I re-opened the cited source and tried at least one independent source or a fresh API pull. All sources were accessed on 2026-10-07. Verdicts are **confirmed**, **partly** (the number or wording is off), **unverifiable** or **refuted**. No claim was refuted, so the report body has no inline ⚠️ markers. [D] marks figures I derived.

**Material corrections (read first)**

| # | Correction | Evidence |
|---|---|---|
| C1 | **Data API `volume` counts shares, never USD.** Cash volume is the `volume_usdc` field. Every "Volume" and "Margin" figure in §2 is therefore mislabelled. Cash-based margins on `economic_pnl` [D]:<br>• `automatedaitradingbot` $88.4k ÷ $365k ≈ **24%** (the table says 2.5%)<br>• 0x8dxd $2.17M ÷ $90.8M ≈ **2.4%** (the table says 1.0%)<br>• gabagool22 $0.865M ÷ $71.4M ≈ **1.2%** (the table says 0.56%) | The v2 OpenAPI describes `volume` as "Both-sides traded volume in shares; never USD": https://data-api.polymarket.com/v2/openapi.json . The changelog for 2026-09-04 says "bare fields are shares, `_usdc` fields are USD". |
| C2 | **Fee rate is a per-market property.** Read `feeSchedule.rate` and `feesEnabled` on each Gamma market; do not use the category table. In a 200-market sample of active markets (ranks 1–100 and 501–600 by 24h volume):<br>• 11 `sports_fees_v2` markets charge **0.03**. These are futures created before the 2026-07-10 change from 0.03 to 0.05.<br>• 3 `sports_fees_nfl_cfb_oct26` markets charge **0.03** with a 15% rebate.<br>• 18 markets have `feesEnabled=false`, all of them geopolitics (Iran, Israel, Yemen, Ukraine, Greenland).<br>• Every schedule has `takerOnly: true`. | https://gamma-api.polymarket.com/markets?active=true&closed=false&order=volume24hr&ascending=false ; changelog entry dated 2026-07-10 |
| C3 | **0x8dxd's "+$2.05M" is `position_pnl`.** Under this report's own method (`economic_pnl`) the lifetime figure is **$2.17M**. The difference includes $71.3k of maker rebates and $49.4k of referral income. | https://data-api.polymarket.com/v2/user-stats?user=0x63ce342161250d705dc0b16df89036c8e5f9ba9a |
| C4 | **Protocol V2 is in the docs**, and it is separate from CLOB V2.<br>• It brings ExchangeV3 (signing domain `"3"`), a new `CONDITIONAL-V2` asset type for balance and allowance calls, and PositionManager and Router contracts.<br>• Existing CTF holdings are not converted.<br>• Canary markets run until Oct 30. Net-new markets tentatively switch on **Nov 2, 2026**.<br>• It is not yet in the changelog.<br>Keep the order-signing and contract layer swappable, and verify it before going live. | https://docs.polymarket.com/migrate/polymarket-v2/overview.md ; [The Block, 2026-10-06](https://www.theblock.co/news/business/2026-10-06-polymarket-rolls-out-v2-upgrade-with-single-erc-1155-positions-contract-pusd-collateral-417804) |

| # | Claim | Verdict | Note (what was checked) | Better / independent source |
|---|---|---|---|---|
| 1 | Taker fee = `C·rate·p·(1−p)`. Rates: Crypto 0.07; Sports, Weather, Econ, Culture and Other 0.05; Politics, Finance, Mentions and Tech 0.04; Geopolitics 0. Makers never pay. | **partly** | **The docs match exactly**, including the formula `fee = C × feeRate × p × (1 - p)` and "Makers are never charged fees". Fees are rounded to 5 decimals, so tiny trades near 0 or 1 can pay nothing.<br><br>**Live markets deviate (C2):** some sports markets are still at 0.03, and geopolitics is shown as `feesEnabled=false`. All other categories in the sample match the table: politics 0.04, crypto 0.07, culture 0.05, weather 0.05, economics 0.05, tech 0.04, finance 0.04 and general 0.05. | Per-market Gamma `feeSchedule` (authoritative): https://gamma-api.polymarket.com/markets ; https://docs.polymarket.com/trading/fees.md |
| 2 | Fee Structure V2 extended taker fees to all categories on 2026-03-30. Crypto taker delay has been 150 ms since 2026-09-04. CLOB V2 (pUSD) went live 2026-04-28. Crypto up/down markets have resolved on a Chainlink TWAP since 2026-08-07. | **confirmed** | Every date and value is in the changelog:<br>• **Mar 30:** "Geopolitical and world events markets remain fee-free", so "all categories" means all except geopolitics.<br>• **Sep 4 (Data API v2 entry):** taker delay set to 150 ms from 50 ms, effective 14:00 UTC. The Aug 17 entry had cut it from 250 ms to 50 ms.<br>• **Apr 28:** CLOB V2 live, with pUSD replacing USDC.e and about 1 h of downtime.<br>• **Aug 7:** TWAP resolution with a 30 s window on 5-min markets, changed to 60 s on Aug 14. | https://docs.polymarket.com/changelog/predictions.md (primary). There is no independent source for the 150 ms delay; an X post is cited in the report [U]. |
| 3 | 18,477 markets carry liquidity rewards totalling $162,401/day. The median is $3/day, and the minimum size is 20 shares in 79% of markets. | **confirmed** (snapshot drift) | I re-paged the whole endpoint (37 pages):<br>• **18,301 unique markets** (−1% drift)<br>• Σ`native_daily_rate` $162,009 and Σ`total_daily_rate` $162,484<br>• median $3, p90 $18, 9 markets above $1k/day<br>• min_size 20 in **79.0%**<br>These are configured rates, not paid-out amounts. | https://clob.polymarket.com/rewards/markets/current (primary; no independent aggregate exists) |
| 4 | `automatedaitradingbot` (0xd8f8…0f11) is a weather-temperature bot with +$88.4k economic PnL on $3.49M volume. Its last trade was 2026-07-14, and its PnL has been flat since May 2026. | **partly** | **Confirmed:**<br>• The Gamma profile name `automatedAItradingbot` and bio "Meteorologist.IT engineer.Automated bot testing" match the wallet.<br>• `economic_pnl` is $88,376.88.<br>• The last `/trades` fill is 2026-07-14 12:03 UTC, a buy on KL/HK highest-temperature markets at 2–4¢.<br>• Monthly ΔPnL [D]: Apr +$8.2k, May +$0.8k, then Jun–Sep between −$73 and +$75.<br><br>**Wrong:** $3.49M is **shares**. Cash volume is **$365k** (C1). | https://gamma-api.polymarket.com/public-profile?address=0xd8f8c13644ea84d62e1ec88c5d1215e436eb0f11 ; https://data-api.polymarket.com/trades?user=0xd8f8c13644ea84d62e1ec88c5d1215e436eb0f11 |
| 5 | 0x8dxd made +$2.05M lifetime on $200.6M volume, but only +$25k in April and ≤ +$7k/month May–Sep 2026. gabagool22 last traded on 2026-02-20. | **partly** | **Monthly `economic_pnl` confirmed [D]:** Apr +$25.1k, May +$3.4k, Jun +$7.1k, Jul +$4.8k, Aug −$1.5k, Sep +$1.1k. The last trade was 2026-08-24 (BTC 5-min).<br><br>**Off:**<br>• $2.05M is `position_pnl`; `economic_pnl` is **$2.17M** (C3).<br>• $200.6M is shares; cash volume is **$90.8M**.<br><br>**gabagool22:** the profile resolves to 0x6031b6…f96d. The last fill was **2026-02-20 09:06 UTC** (confirmed). Only referral rewards continue, the latest on 2026-06-24. | https://data-api.polymarket.com/v2/user-stats?user=0x63ce342161250d705dc0b16df89036c8e5f9ba9a ; https://data-api.polymarket.com/trades?user=0x6031b6eed1c97e853c6e0f03ad3ce3529351f96d |
| 6 | On the Weather leaderboard for the last 30 days, the top 50 made $412k combined; #1 made $43k and #10 made $11.9k. | **confirmed** | The v1 and v2 boards are identical:<br>• Σ top 50 **$415.5k** (the rolling window drifts)<br>• #1 Bilberry $43.1k, #10 $11.9k, #25 $5.08k (the §2 table says $4.8k), #50 $2.9k<br>• The board's `vol` field is in shares.<br>Leaderboards are survivorship-biased. | https://data-api.polymarket.com/v2/leaderboard?category=weather&time_period=month&order_by=pnl&limit=50 |
| 7 | Arbitrage realised Apr 2024–Apr 2025 was about $39.6M: $10.58M single-condition, $29.01M NegRisk rebalancing and $95k combinatorial. The top account made $2.01M. | **confirmed** | The paper's text gives:<br>• single-condition: $5,899,287 + $4,682,075 = **$10.58M** [D]<br>• rebalancing: $11.09M + $0.61M + $0.004M + $17.31M = **$29.02M** [D]<br>• pairs: $60.2k + $18.5k + $15.8k + $0.6k = **$95.2k** [D]<br>• stated total **$39,587,585**; top user **$2,009,631.76**<br>• window: markets resolved from 2024-04-01 to 2025-04-01<br><br>**Caveats:**<br>• These are *estimates* that assume ε = $1 profit per trade.<br>• The components sum to $39.69M, against the paper's stated $39.59M [D].<br>• All of it predates taker fees. | [arXiv HTML](https://arxiv.org/html/2508.03474) ; [IMDEA PDF (AFT 2025)](https://suarez-tangil.networks.imdea.org/papers/2025aft-arbitrage.pdf) |
| 8 | Only 3.14% of 1.72M Polymarket accounts are skilled, and 44% of them stay skilled out of sample. Only about 12% of top PnL earners are in the skilled group. | **confirmed** | The Block (2026-04-26) gives all three figures. Yale Insights independently confirms the 1.72M accounts, "just 3%" and the 44% persistence.<br><br>I found the **12%** figure only in The Block; SSRN returned "Content Blocked". | [Yale Insights](https://insights.som.yale.edu/insights/wisdom-of-the-few-prediction-markets-are-driven-by-small-number-of-skilled-traders) ; [SSRN 6617059](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=6617059) |
| 9 | Contracts bought at ≥90¢ earn +0.83¢ per dollar before fees on Polymarket (588M trades); the sign is sensitive to event grouping. | **partly** | **Confirmed:** +0.83% pooled (95% CI 0.60–1.07) from 588M trades by 2.48M accounts. It is pre-fee: "The calculations do not subtract transaction-specific fees or include rebates."<br><br>**Misattributed:** the grouping sign-flip is for **longshots** (−6.3¢ with equal child weights vs +4.1¢ grouped by parent event). Favourites stay positive under all three weightings: +0.28% (equal child), +0.83% (pooled) and +0.39% (equal parent event).<br><br>**The favourite sign flips only with buyer action (Table 16):**<br>• *accepting* the seller's offer (taker) returns **−0.46%** with equal child weights<br>• *posting* the bid (maker) returns +0.64% with equal child weights and +0.82% pooled<br>• the fee-flagged pooled return is +0.33%, with a CI that includes 0<br>• the pattern is absent in Sports.<br>This supports maker-only bonds. | [arXiv 2609.12878 HTML §5.2, Table 16, Panel C](https://arxiv.org/html/2609.12878) |

**Method notes**
- Monthly `economic_pnl` figures are the change between the last daily points of consecutive months in `/v2/user-pnl?interval=all&fidelity=1d`.
- The Gamma sample covers ranks 1–100 and 501–600 by 24h volume: the endpoint caps each call at 100 rows, and I called offsets 0 and 500. CLOB rewards were paged in full.
- Raw pulls are in the session scratchpad (`v2/`). Python's default user agent gets HTTP 403 from the CLOB, so requests used curl.
