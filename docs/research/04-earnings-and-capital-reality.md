# 04 — Earnings and capital reality: what Polymarket traders and bots actually make

Access date for every source and API pull: **2026-10-07**. No logins, no orders, no third-party code run.
Only public read-only endpoints were called (`data-api.polymarket.com`, `user-pnl-api.polymarket.com`, `gamma-api.polymarket.com`, `clob.polymarket.com` GET).

**Labels:** **[V]** verified by us from Polymarket's public API or a primary paper · **[S]** secondary report (press or analyst), not re-verified · **[C]** claimed or unverified (marketing, social posts, vendors) · **[D]** derived by us (arithmetic or judgment on the above).

---

## 0. Bottom line

1. **Most wallets lose money.** Four independent on-chain studies put the losing share at **68–84%**. Profits are extremely concentrated: the top 1% of winning users take **76.5%** of profits ([V] Akey et al., CEPR DP21615). Fewer than 0.04% of addresses took **>70%** of realised profit ([S] DeFi Oasis).
2. **Earning $5k–10k in one month puts you in roughly the top 1,400–2,400 wallets on the platform.** [V] Live leaderboard: rank 1,396 earned ≥$10k and rank 2,372 earned ≥$5k in the trailing month. **Earning ≥$5k for 4 months in a row: 0.015% of addresses** ([S] Sergeenkov, 2.5M wallets).
3. **Bots win as a class, but not most bots, and not for long.** Bots are 0.54% of wallets but place 63% of trades and 28% of matched volume ([S] Bitquery). In one measured week, the bot class as a whole **lost $2.8M after fees** ([S] Bitquery, 11–19 Aug 2026).
4. **The famous small-bankroll bots really did make millions, and their edges then died.** We pulled monthly PnL for 5 crypto "Up or Down" bots. Each made **$112k–$880k per month** from Dec 2025 to Mar 2026, then **about $0 per month from April 2026 on** [V]. The top weather traders show the same pattern: $8k–51k per month from Dec 2025 to Apr 2026, then roughly flat or negative from May to Jul 2026 [V]. Niche edges lasted about **4–5 months**.
5. **No published evidence shows an LLM forecasting bot making sustained profit after fees.** In a live 57-day test with $10k per model, every model lost 16–31% on Kalshi and averaged −1.1% on Polymarket ([V] Prediction Arena, arXiv 2604.07355).
6. **Capital arithmetic [D]:** $5k per month needs $25k of capital at a 20% monthly net return, $50k at 10%, or $100k at 5%. The only evidence for ≥10% per month comes from niches that decayed within months, and the capital behind it is unobservable.
7. **With $50–150, the realistic 3-month outcome is between −$100 and +$75.** Even 20% per month compounded gives +$73 on $100. The bankroll is a **measurement budget** to test whether the bot has an edge, not an income source. Reaching $5k per month needs proven edge **and** about $25k–100k of capital. Compounding from $150 does not get there: about 30 months at a sustained 20% per month, a rate nobody has documented.

---

## 1. How trader profitability is distributed (studies)

| Study | Data / period | Losing share | Concentration | Other thresholds | Label / source |
|---|---|---|---|---|---|
| Akey, Grégoire, Harvie, Martineau, "Who Wins and Who Loses in Prediction Markets? Evidence from Polymarket" (CEPR DP21615, 12 Jun 2026) | 588M trades, $67B volume | 68.8% unprofitable ([S] press summary) | **Top 1% of winners take 76.5% of profits** | Winners **provide liquidity with limit orders**; losers **take liquidity with market orders**; monthly persistence is modest | [V] abstract: https://cepr.org/publications/dp21615 |
| Sergeenkov (Dune, on-chain, realised PnL incl. splits/merges) | Apr 2024–Apr 2026, 2.5M addresses | **84.1%** | — | >$1k lifetime 2%; >$10k 0.32% (~8,000); >$100k 0.033% (~840). **Average ≥$5k/month: 0.26%; ≥$10k/month: 0.13%.** Of those averaging ≥$5k/month, 53% were active only 1 month. **≥$5k in 1 month 0.98%, 2 consecutive 0.1%, 4 consecutive 0.015%** | [S] https://sergeenkov.com/polymarket-profitability/ |
| DeFi Oasis (via Yahoo/Cointelegraph) | to Dec 2025, 1.7M addresses | ~70% | **<0.04% of addresses took >70% of profits ($3.7B)**; 668 addresses >$1M = 71% of gains | 63.5% lost $0–1k; 149 addresses lost >$1M each; ~462,600 monthly active traders | [S] https://finance.yahoo.com/news/70-polymarket-traders-lost-money-192327162.html |
| Bloomberg analysis (28 Apr 2026) | every wallet active Jan 2025–Apr 2026 | >100,000 accounts lost ≥$1k, **about 2× the number that made ≥$1k** | ~800 bot-like accounts made >$100k each, **≈$131M**, which roughly mirrors retail losses | Della Vedova: bots win by entering earlier at better prices, not by predicting better | [S] https://news.bgov.com/crypto/prediction-market-users-suffer-broad-losses-as-bots-reap-gains |
| Solidus Labs (vendor of surveillance tools) | politics markets, Dec 2025–Feb 2026 | — | 0.55% of profitable makers took 50% of maker gains; 0.26% of winning takers took ~50% of taker gains (~$8M each) | ~15% of volume in some markets resembled wash trading | [S] https://www.coindesk.com/markets/2026/04/29/a-tiny-group-is-winning-on-polymarket-as-under-1-of-wallets-take-half-the-profits |
| WSJ (via Yahoo, 5 May 2026) | 1.6M accounts | 1.1M unprofitable (~69%) | **0.1% of accounts took 67% of profits** | — | [S] https://finance.yahoo.com/markets/crypto/articles/two-thirds-polymarket-profits-just-100500195.html |
| "95M transactions" report | Apr 2024–Dec 2025 | — | — | 0.51% of wallets made >$1k | [S] https://www.htx.com/es-la/news/344363/ |

**Takeaways:** results vary with method: whether splits and merges are counted, realised versus unrealised PnL, and address versus person. Under any method, about 70–84% of wallets lose and about 1% of winners take three-quarters of profits. The most decision-relevant finding is that **profits come from posting limit orders, and losses from crossing the spread** (Akey et al.).

## 2. How much volume and profit bots capture

| Metric | Value | Period | Label / source |
|---|---|---|---|
| Bot wallets (24-hour activity, median gap <60s) | 12,685 of 2,347,464 (**0.54%**) | Sep 2025–Aug 2026 | [S] Bitquery https://bitquery.io/investigations/how-much-of-polymarket-is-bots |
| Bot share of trades / matched $ / fees | **63.1% / 28.3% / 36.6%** of 1.65B trades, $34B, $345.9M | same | [S] Bitquery |
| Bot share of matched $ over time | ~16% (Sep 2025) → 30.8% (Aug 2026); **48.6% with a machine on either side** | same | [S] Bitquery |
| Bot share by market duration | <15 min: 55.3%; 15 min–1 h: 31.1%; 1 h–1 d: 26.1%; >1 week: 16.2% | same | [S] Bitquery |
| **Bot class PnL in one week** | **before fees −$0.47M; fees $2.35M; after fees −$2.82M; win rate 34.4%** (non-bots −$3.86M after fees) | 11–19 Aug 2026 | [S] Bitquery |
| Bot share of volume in markets with <$10k total volume | **>80%** (bot = >50 trades/day or >1,000 trades) | Nov 2022–Feb 2026 | [S] Della Vedova via CNBC https://www.cnbc.com/2026/07/02/prediction-markets-mostly-have-thinly-traded-contracts-.html |
| **Bot profit in shallow (<$10k) markets vs deep (>$10M) markets** | **~$1.2M vs ~$35.1M** | Nov 2022–Feb 2026 | [S] Della Vedova via CNBC (SSRN 6191618) |
| Bot profit in shallow markets per month, all bots combined | **~$31k/month** ($1.2M / ~39 months) | — | [D] |
| Closed markets with <$10k lifetime volume | ~70%; >45,000 markets (~5%) had zero volume | 2021–May 2026 | [S] CNBC |

**Takeaway:** bots carry most of the long-tail flow, but **the whole long-tail profit pool is tiny.** All bots combined made about $31k per month from shallow markets [D]. A newcomer competes with established bots for a share of that pool.

## 3. Official leaderboard snapshot (pulled 2026-10-07)

Source: `https://data-api.polymarket.com/v1/leaderboard?timePeriod={month|week|all}&orderBy=PNL&limit=50&offset=N` (offset cap 10,000). "Month" is a trailing window. The PnL definition is not documented; it probably includes unrealised PnL.

### 3a. What rank a given monthly PnL buys you [V]

| Trailing-month PnL | Rank reached | Trailing-week rank → PnL |
|---|---|---|
| $3.65M | #1 (00gringo00) | #1 → $2.00M |
| $1.18M | #10 | #100 → $59k |
| $154k | #100 | #500 → $11.9k |
| $30k | #500 | #1,000 → $5.5k |
| $15k | #1,000 | #2,500 → $1.7k |
| **≥$10k** | **#1,396** | #5,000 → $580 |
| **≥$5k** | **#2,372** | #10,000 → $168 |
| ≥$1k | #6,659 | |
| $478 | #10,000 | |

- Denominator: the month leaderboard ranks at least **526,962** wallets. We queried a known wallet and it ranked #526,962 [V]. DeFi Oasis counts about 462,600 monthly active traders [S]. **$5k/month ≈ top 0.45%; $10k/month ≈ top 0.27% of active wallets** [D].
- The top 1,000 wallets made a combined **$80.7M** in the month. The top 10 took 22.9% of that [V].

### 3b. Who sits at the top: PnL / volume tells the style [V]

| Window | Rank | Wallet | PnL | Volume | PnL/vol | Style (inferred) |
|---|---|---|---|---|---|---|
| Month | 1 | 00gringo00 | $3.65M | $28.2M | 13.0% | directional |
| Month | 5 | e46m3 | $1.77M | $2.5M | 71.6% | concentrated bet |
| Month | 10 | UpTheBlues | $1.18M | $123.9M | 1.0% | high-turnover bot/MM |
| Month | 13 | HomeRunHazard | $0.79M | $112.2M | 0.7% | high-turnover bot/MM |
| All | 1 | Theo4 | $22.1M | $43.0M | 51.3% | 2024 election whale |
| All | 2 | swisstony | $18.4M | $1,850.6M | 1.0% | sports bot (Bitquery: 5.7M trades, $6.2M fees) |
| All | 4 | RN1 | $12.7M | $1,339.1M | 0.95% | high-turnover sports |
| All | 5 | kch123 | $11.4M | $293.4M | 3.9% | sports |

- Among the top 1,000 by monthly PnL, the median PnL/volume is 6.7%. 18% of them earn under 2% of volume (bot or market-maker style) and 20% earn over 20% (big directional bets) [V].
- Over all time, the median is 2.8% and 41% earn under 2% [V]. **The high-turnover machines earn about 0.5–1% of the volume they trade.**

### 3c. Data-quality warning [V]

For high-turnover wallets, the leaderboard PnL and `user-pnl-api` can **disagree even in sign**:

- ferrariChampions2026: leaderboard month PnL −$42.8M, but `user-pnl-api` shows +$0.4M to +0.8M over the same window.
- RN1: leaderboard month −$3.44M versus +$0.73M from `user-pnl-api`.

For directional wallets the two sources roughly agree (00gringo00: $3.65M vs $2.97M). **Use the `user-pnl-api` time series to track competitors.** Do not trust leaderboard PnL for market-maker wallets.

## 4. Documented bot earnings: monthly PnL pulled from `user-pnl-api` [V]

Method: `https://user-pnl-api.polymarket.com/user-pnl?user_address=<wallet>&interval=all&fidelity=1d`, taking the difference at each month end [D]. Usernames were resolved through `gamma-api.polymarket.com/public-search`. Each wallet's market type was confirmed from `data-api.polymarket.com/activity`.

### 4a. Crypto "Up or Down" bots (5-min, 15-min and hourly BTC/ETH/XRP markets)

| Wallet | Lifetime PnL / volume | Monthly PnL | Now |
|---|---|---|---|
| **0x8dxd** (0x63ce…ba9a) | $2.05M / $200.6M (1.0%) | Dec-25 +$322k · Jan +$590k · Feb +$880k · Mar +$591k · **Apr +$0.2k** · May–Oct ≈ $0 | still trades BTC Up/Down (median $29 trade), earns ≈ $0 |
| gabagool22 (0x6031…96d) | $0.81M / $144.8M (0.56%) | Nov +$134k · Dec +$300k · Jan +$277k · Feb +$152k · **Mar–Oct $0** | stopped |
| distinct-baguette (0xe007…548c) | $0.70M / $72.4M (0.97%) | Nov–Mar +$112k to +$227k each month · **Apr–Oct $0** | stopped |
| Account88888 (0x7f69…6162) | $0.62M / $120.4M (0.52%) | Dec +$378k · Jan +$268k · then ≈ $0 | trades $0.6 clips, ≈ $0 |
| k9Q2mX4L8A7ZP3R (0xd0d6…93aa) | $1.39M / $208.9M (0.67%) | active to Apr 2026 | stopped |

- **The "$313 → $414k in a month" story.** The $313 start, $4–5k bets and 98% win rate are **[C]**: they come from a social post by "Dexter's Lab" relayed through press (https://finbold.com/polymarket-trading-bot-that-turned-300-into-400000-is-now-up-this-much/). The PnL itself is **[V]**: +$322k in Dec 2025 and +$590k in Jan 2026.
- **Why the edge died.** Polymarket introduced dynamic taker fees on 15-minute crypto markets on 5 Jan 2026 and extended them to all crypto timeframes on 6 Mar 2026 ([S] https://pineanalytics.substack.com/p/polymarket-fee-rollout). The crypto fee rate is 0.07, which costs **$1.75 per $100 at p = 0.5** ⚠️ refuted, see Verification ([V] https://docs.polymarket.com/trading/fees). All five bots went to about $0 per month by April 2026. We see the timing match but cannot prove the fees caused it, because competition also rose [D].
- **Other claims.** "$63 → $131k in a month" (elitetrader forum) is **[C]**.

### 4b. Weather (daily highest-temperature) traders

Wallets are taken from a "top 7 weather traders" list ([C] for the starting capitals: https://dev.to/xniiinx/my-top-7-most-profitable-weather-market-traders-on-polymarket-gb7). The PnL series below are [V].

| Wallet | Lifetime (pnl-api) | Boom months | After 2026-03-30 weather fees | Claimed start |
|---|---|---|---|---|
| ColdMath | $145k | Jan +$14k · Feb +$29k · **Mar +$42k · Apr +$45k** | May −$1.8k · Jun −$1.3k · Jul −$0.6k · Aug +$0.2k · Sep +$9.1k. **Now mostly trades sports.** | ~$2,000 [C] |
| VibeTrader | $117k | Dec +$51k · Jan +$49k | Apr −$8.0k · then −$1k to −$4.8k per month through Sep | ~$3,000 [C] |
| BeefSlayer | $87k | Jan +$12k · Mar +$21k · Apr +$16k | May −$0.1k · Jun +$3.0k · Aug +$11.8k · Sep +$7.5k | ~$170 [C] |
| HenryTheAtmoPhD | $64k | Mar +$11k · Apr +$21k | May +$0.4k · Jun +$2.0k · then ≈ $0 | ~$1,500 [C] |
| HondaCivic | $69k | Feb +$20k · Mar +$28k | Apr +$8.2k · May +$2.5k · Aug +$9.4k · Sep +$1.1k | ~$1 [C] |
| **automatedAItradingbot** (user's link; 0xd8f8…0f11) | **$88k on $3.49M volume (2.5%)** | Nov-25 +$21k · Dec–Apr +$6k to +$10k per month | **May +$0.8k · Jun +$0.2k · Jul +$0.1k · Aug–Oct ≈ $0** | — |

automatedAItradingbot details [V]:

- It trades weather markets (Hong Kong, Seoul, Shenzhen, Moscow) with a **median trade of $1.79** and a largest trade of $257.
- The wallet now holds **$10.92**, and its last activity was 2026-09-28.
- **Its edge ran from about Feb 2025 to Apr 2026 (~14 months). It peaked at about $21k per month and is now dead.**

**Weather niche today [V]:** 1,143 temperature markets traded in the last 24h. The median market traded $335 a day with $2.9k of liquidity, and all of them together traded about $1.36M a day (gamma snapshot).

Marketing claims we could not verify:

- "$1k → $24k London weather bot" (https://blog.devgenius.io/found-the-weather-trading-bots-quietly-making-24-000-on-polymarket-and-built-one-myself-for-free-120bd34d6f09) **[C]**
- "$6k/month blueprint" pages **[C]**

### 4c. Other classes

| Class | Evidence | Label |
|---|---|---|
| Structural arbitrage (one market's outcomes summing ≠ 1; logically linked markets) | **$39.59M extracted Apr 2024–Apr 2025, all arbitrageurs combined.** Within one condition $10.58M; across a multi-outcome market $28.28M. Top account **$2.01M over 4,049 trades**; #2 $1.27M; #3 $1.09M | [V] Saguillo et al., arXiv 2508.03474 https://arxiv.org/html/2508.03474v1 |
| Arbitrage after 2026 | Fees now apply to every leg. A 10-outcome full set costs about 4.5¢ per $1 at rate 0.05 | [D] in docs/03-strategies.md |
| Market making / liquidity rewards | **Total daily liquidity-reward pool $162,481 across 18,477 markets. Median market $3/day, p90 $17/day; only 208 markets ≥$100/day** (CLOB `/rewards/markets/current`). Makers pay no fee and receive 15–25% of taker fees as rebates | [V] API; [V] https://docs.polymarket.com/trading/fees |
| Market making, open-source | poly-maker README warns that market making is competitive and can lose money | [V] https://github.com/warproxxx/poly-maker |
| LLM bots, live | **Prediction Arena** (Arcada Labs/Harvard), $10k per model, live 12 Jan–9 Mar 2026. **Kalshi: every model −16.0% to −30.8%. Polymarket: −1.1% on average.** Best paper result +6.02% over 3 days (gemini-3.1-pro). The amount of research a model did did not correlate with returns | [V] https://arxiv.org/html/2604.07355v1 |
| LLM bots, simulated | **PolyBench**: 7 LLMs, 38,666 markets, 6–12 Feb 2026. **Only 2 of 7 were profitable** (+17.6%, +6.2%) | [V] https://arxiv.org/abs/2604.14199 |
| LLM bots, vendor | Olas Polystrat: 4,200+ trades in its first month, 56–69% accuracy, about 37% of instances PnL-positive, **fleet not yet at break-even** | [C] vendor statements via press https://www.mexc.com/news/997996 |
| LLM bots, anecdote | One tester ran 7 bots on $9,200 for 4 months and ended −$3,840 (−42%) | [C] https://prediction-market.medium.com/polymarket-ai-bot-review-2026-heres-what-actually-worked-efb8e5d75740 |
| High-probability "bond" (buy ≥90¢) | Pooled +0.83% before fees, but the sign flips across weightings | [S] cited in docs/03-strategies.md (arXiv 2609.12878); not re-verified here |

## 5. Realistic monthly returns by strategy class

| Strategy class | Verified profits | Return on capital | Survivorship / decay | Fit for $50–150 + free LLM |
|---|---|---|---|---|
| Crypto Up/Down latency / complete-set bots | $0.1–0.9M per month per bot, Dec-25 to Mar-26 [V] | **Unknown.** Capital is not observable. "$313 start" [C] | **All 5 at ≈ $0 since Apr 2026** [V]; fee is 1.75% at p = 0.5 ⚠️ refuted, see Verification | No. It is a millisecond race and the edge is gone |
| High-frequency sports trading / MM | swisstony $18.4M, RN1 $12.7M lifetime at ≈1% of volume [V] | Unknown. Likely 7–8-figure capital [D] | Big wallets can show large losses (see §6) | No. Needs size and speed |
| Weather specialists | $8–51k per month in the boom [V] | Unknown. Trade sizes and claimed starts suggest $1–20k bankrolls [C/D] | Boom Dec-25 to Apr-26, then flat or negative May–Jul; 2 of 6 partly recovered Aug–Sep [V] | Possible as a research niche. Crowded; edge is model-driven, not LLM-driven |
| Structural arbitrage | $39.6M per year across all players, 2024–25 [V] | n/a | Fee-eroded since 2026 [D] | Monitor only |
| Liquidity rewards / maker | Pool $162k/day across the platform [V] | Long-tail market ≈ $3/day shared by all LPs [V] | Adverse selection | Small rebates only. **Maker-first execution is the one evidence-backed habit** (Akey et al.) |
| LLM directional forecasting | **None verified.** Live average −1.1% (Polymarket) and −16% to −31% (Kalshi) [V] | ≤0 on average; occasional +6% over days | Benchmarks are short and unrepeated | **Unproven. This is our hypothesis to test, not a known edge** |
| Big directional bets (whales) | Theo4 $22M [V] | One-off | Bossoskil1 lost $2.36M in 8 days [S] | No |

**Survivorship bias.** Every named winner above was picked because they won. The unbiased base rates are in §1: about 70–84% lose. Of the ~6,600 addresses that averaged ≥$5k per month, 53% did it for a single month only [S].

## 6. What failed

| Case | What happened | Label / source |
|---|---|---|
| Bot class, one week | −$2.82M after fees; fees ($2.35M) were 5× the pre-fee loss | [S] Bitquery |
| **suntori** (bot; 3.38M trades, $3.0M fees) | Cumulative **−$8.2M** (pnl-api), never above $0. Losses grew from −$0.07M (Mar-26) to −$3.35M (Jul-26). Trades crypto Up/Down, LoL, tennis | [V] pnl-api; [S] Bitquery |
| Crypto latency bots | Monthly PnL went from $0.1–0.9M to $0 within about 3 months of fees and competition | [V] §4a |
| Weather boom | Top specialists went from +$20–50k per month to slightly negative after the 2026-03-30 fee rollout | [V] §4b |
| automatedAItradingbot | 14-month edge, decayed to $0; wallet now holds $10.92 | [V] |
| "Nothing Ever Happens" (always buys NO ≤65¢) | 73% of markets resolve NO, but at typical NO prices the margin is too thin, so the creator keeps losing | [S] https://protos.com/this-bot-only-bets-no-on-polymarket-and-its-creator-keeps-losing-money/ |
| LLM agents | Every model lost on Kalshi over 57 days | [V] Prediction Arena |
| Bossoskil1 | −$2.36M in 8 days on sports spreads; 47% win rate, $200k+ bets, no hedging | [S] https://finbold.com/how-this-crypto-trader-blew-over-2-million-in-8-days-on-polymarket/ |
| Oracle risk | "Ukraine mineral deal" market ($7M volume) resolved YES with no deal. A UMA holder with about 25% of votes was suspected; Polymarket refused refunds | [S] https://www.coindesk.com/markets/2025/03/27/polymarket-uma-communities-lock-horns-after-usd7m-ukraine-bet-resolves |
| **Malware** | More than 20 malicious "Polymarket bot" repos in a hijacked GitHub org (`dev-protocol/polymarket-copytrading-bot-sport` and others) steal `.env` keys. A DPRK npm package also targets Polymarket bot developers | [S] https://www.stepsecurity.io/blog/malicious-polymarket-bot-hides-in-hijacked-dev-protocol-github-org-and-steals-wallet-keys ; https://panther.com/blog/polymarket-trader-funds-at-risk-dprk-npm-package-steals-wallet-keys |

## 7. Capacity: how much money thin books can absorb

Snapshot of every active market with 24h volume above zero, via `gamma-api /markets/keyset?order=volume24hr` [V]:

| Metric | Value |
|---|---|
| Active markets with 24h volume > 0 | 15,274 (gamma reported $30.2M in 24h volume; this field undercounts compared with headline figures) |
| Markets ≥$1k / ≥$10k / ≥$100k in 24h | 2,197 / 449 / 48 |
| Share of 24h volume in the top 10 / top 100 / top 1,000 markets | 18.9% / 52.2% / 86.3% |
| **Long-tail markets (24h volume $100–10k)** | **6,813 markets: median $361/day volume, $4.5k liquidity, 1.8¢ spread (p75 4¢)** |
| Liquidity of traded markets, p25 / median / p75 / p90 | $516 / $2.9k / $15.7k / $36.9k |

**Capacity arithmetic [D]:**

- **Per-market limit.** If we take ≤10% of a market's daily volume to avoid moving the price, a median long-tail market absorbs about **$36 a day**.
- **Fill volume needed.** $5k per month at a **5% net edge per dollar traded** needs $100k of fills a month, about $3.3k a day. That means trading about **90 median long-tail markets every day, each with a live edge**. Holding to resolution with 2× monthly capital turnover would tie up about **$50k**.
- **Pool size.** Della Vedova measured all bots combined making about **$31k per month** in shallow markets (§2). **The long-tail pool cannot support a $5–10k/month newcomer** unless the edge is unusually large or the bot moves into deeper markets, where competition is harder.

## 8. Derived: capital needed for $5k and $10k per month

| Monthly net return | Capital for $5k/mo | Capital for $10k/mo | Months to grow $100 into the $5k capital at that rate | Classes with evidence at this return | Evidence at that capital size? |
|---|---|---|---|---|---|
| 1% | $500,000 | $1,000,000 | 856 | High-turnover sports/MM (≈1% of *volume*, which is more than 1% of capital if capital turns over); "bond" strategy (+0.83% pre-fee, fragile) [V/S] | Yes, top-100 and top-1,000 markets only (449 markets ≥$10k/day). Needs speed and risk infrastructure |
| 2% | $250,000 | $500,000 | 395 | Same, plus maker rebates and rewards | Same |
| 5% | $100,000 | $200,000 | 142 | Top crypto bots and weather specialists **at their peaks** (capital unobservable). **No LLM evidence** | No verified case at $100k in long-tail markets. Long-tail capacity binds (§7) |
| 10% | $50,000 | $100,000 | 65 | Only transient niches (crypto Up/Down Dec-25 to Mar-26; weather Dec-25 to Apr-26) | Edges decayed within 4–5 months [V] |
| 20% | $25,000 | $50,000 | 30 | Same transient niches, small bankrolls only | **$25k exceeds what thin books absorb at edge** [D] |
| 50% | $10,000 | $20,000 | 11 | Only anecdotes such as "$313 → $414k" [C], which is survivorship | Every sampled small-start bot fell to ≈$0/month within ≤5 months [V] |

**Reading the table.**

- **The return rates that make small capital work (≥10% per month) appear only in temporary niches.** Those niches were sharply crowded and fee-taxed in 2026.
- **The return rates with durable evidence (≈1–2% per month, MM-style) need six to seven figures of capital and low-latency infrastructure.**
- **There is no evidence point anywhere for "LLM long-tail bot earns 5–20% a month at $25–100k."**

## 9. A $50–150 bankroll over 3 months: realistic outcomes

Dollar ceiling on $100 [D]:

| Monthly return | 1% | 2% | 5% | 10% | 20% | 50% |
|---|---|---|---|---|---|---|
| 3-month gain on $100 | +$3 | +$6 | +$16 | +$33 | +$73 | +$238 |

**Outcome distribution [D, judgment].** Inputs: base rates in §1 (70–84% of wallets lose), Prediction Arena (live LLM average −1.1% to −31%), fees up to 1.75% per taker fill outside geopolitics ⚠️ refuted, see Verification, and long-tail spreads of about 2–4¢.

| Outcome after 3 months (live, $100) | Probability (judgment) |
|---|---|
| Paper phase shows no edge, so we never go live or stop early (a **good** outcome; costs $0) | 35–50% |
| Live: −20% to −100% (fees, adverse selection, correlated LLM errors) | 20–30% |
| Live: −20% to +10% (noise around zero) | 15–25% |
| Live: +10% to +50% | 5–15% |
| Live: > +50% | ≤5% (mostly luck from a few correlated bets) |
| On track for $5k/month | ≈0% |
| Tail: total loss from a key leak or malware | low, but only if we never run third-party bot code and keep keys isolated |

**Statistical power [D].** Number of resolved bets needed to show a positive mean return at one-sided 95% confidence:

| Edge per $ | p = 0.3 | p = 0.5 | p = 0.8 |
|---|---|---|---|
| 3% | 7,133 | 3,004 | 681 |
| 5% | 2,595 | 1,080 | 227 |
| 10% | 665 | 268 | 45 |

Three months of long-tail trading resolves only a few hundred bets, because markets take weeks to resolve. **Live PnL over 3 months cannot prove an edge.** Measure edge on **paper forecasts across thousands of resolved markets**, using Brier score or log-loss against the market price at forecast time. Paper forecasts cost nothing and scale.

## 10. Implications for the build (for docs 02 / 03 / 05)

1. **Treat the first $50–150 as tuition and a measurement budget.** The 3-month objective is to show positive Brier skill against the market on ≥1,000 resolved paper forecasts, plus paper PnL after fees with depth-checked fills. Income is not the objective.
2. **Execute maker-first.** Winners post limit orders and losers cross the spread (Akey et al.). Makers pay no fee and earn rebates. Cross the spread only in fee-free **geopolitics** markets, or when edge exceeds fee + spread + slippage by a margin.
3. **Cap each order by capacity:** ≤10% of the market's trailing 24h volume and ≤ the depth available inside the edge. Log rejected opportunities so we can measure capacity.
4. **Assume any edge decays in 3–6 months.** Monitor rolling 30-day realised edge and Brier score per category, and auto-pause a category when it turns negative.
5. **Benchmark competitors with `user-pnl-api` time series**, not leaderboard PnL (§3c).
6. **Set the scaling rule now.** Add capital only after the paper and small-live edge is statistically supported. $5k/month needs about $25–100k of capital *plus* proven capacity, so it comes from deliberate capital injection after evidence, not from compounding $150.
7. **Security:** never install third-party "Polymarket bot" repos or npm packages without review. Use a dedicated hot wallet that holds only the trading float.

---

### Raw pulls (reproducible, read-only)

- Leaderboard: `curl "https://data-api.polymarket.com/v1/leaderboard?timePeriod=month&orderBy=PNL&limit=50&offset=0"`. The `user=<proxyWallet>` parameter returns a single wallet's rank.
- PnL series: `curl "https://user-pnl-api.polymarket.com/user-pnl?user_address=<wallet>&interval=all&fidelity=1d"`
- Trades: `curl "https://data-api.polymarket.com/activity?user=<wallet>&limit=300&type=TRADE"`
- Markets by 24h volume: `https://gamma-api.polymarket.com/markets/keyset?active=true&closed=false&limit=500&order=volume24hr&ascending=false` (follow `next_cursor`)
- Reward pools: `https://clob.polymarket.com/rewards/markets/current` (follow `next_cursor`)
- Wallets: 0x8dxd `0x63ce342161250d705dc0b16df89036c8e5f9ba9a`; gabagool22 `0x6031b6eed1c97e853c6e0f03ad3ce3529351f96d`; Account88888 `0x7f69983eb28245bba0d5083502a78744a8f66162`; distinct-baguette `0xe00740bce98a594e26861838885ab310ec3b548c`; automatedAItradingbot `0xd8f8c13644ea84d62e1ec88c5d1215e436eb0f11`; ColdMath `0x594edb9112f526fa6a80b8f858a6379c8a2c1c11`; suntori `0xe9076a87c5ed90ef16e6fe6529c943baeca0cff6`; swisstony `0x204f72f35326db932158cba6adff0b9a1da95e14`.

---

## Verification (2026-10-07)

Adversarial re-check of 10 headline claims. Every source below was opened on **2026-10-07**; API figures were re-pulled the same day (read-only GETs). Verdicts: **confirmed** · **partly** (a number or wording is off) · **unverifiable** · **refuted**.

| # | Claim | Verdict | Note | Better / independent source |
|---|---|---|---|---|
| 1 | Top 1% of profitable users take 76.5% of profits; 588M trades, $67B; winners post limit orders, losers cross with market orders | **confirmed** | The abstract matches word for word, and "top 1%" means the top 1% of users *with positive PnL*. The independent coverage puts profitable users at 764,988 of 2,469,589 (≈31%, derived), which matches the 68.8% losing share in §1 | https://cepr.org/publications/dp21615 ; https://econpapers.repec.org/RePEc:cpr:ceprdp:21615 ; https://gigazine.net/gsc_news/en/20260525-polymarket-win-or-lose/ |
| 2 | 84.1% of 2.5M addresses in the red (Apr 2024–Apr 2026); ≥$5k in 4 consecutive months 0.015%; 53.4% of ≥$5k/month earners active 1 month | **confirmed** | The source states all three numbers. The data runs to 1 Apr 2026 and the post is dated 6 Apr 2026. It counts realised PnL only, including splits and merges, from Dune. This is one analyst's work and is not peer-reviewed. Press coverage repeats the same figures | https://sergeenkov.com/polymarket-profitability/ ; https://thedefiant.io/news/research-and-opinion/polymarket-profitability-report-april-2026 |
| 3 | Month leaderboard: ≥$10k ≈ rank 1,396; ≥$5k ≈ rank 2,372; rank 1,000 ≈ $15k; rank 1 $3.65M | **confirmed** | Re-pulled: #1 00gringo00 $3,654,335; #1,000 $15,066; the last wallet ≥$10k is #1,399 ($10,003); the last ≥$5k is #2,378 ($5,005). The ranks drift by a few places as the window moves. **Caveat:** #1's "month" PnL ($3.65M) is larger than its *all-time* leaderboard PnL ($3.51M), while `user-pnl-api` gives +$2.97M over 30 days. The wallet's PnL series starts 2026-09-09. The leaderboard's PnL definition is undocumented and not internally consistent | `data-api.polymarket.com/v1/leaderboard?timePeriod={month,all}&user=0x5169…1842` ; `user-pnl-api…?interval=1m` |
| 4 | Bots are 0.54% of wallets, 63.1% of trades, 28.3% of matched $, 36.6% of fees (Sep 2025–Aug 2026); bot PnL 11–19 Aug 2026 −$2.82M after fees | **confirmed** (vendor figure) | Matches the source: 12,685 of 2,347,464 wallets; window 1 Sep 2025–19 Aug 2026; −$472,989 before fees, $2,349,133 fees, −$2,822,122 after fees, win rate 34.4%. "Bot" is Bitquery's own timing heuristic, and no independent replication was found. The PnL window is 9 days, not exactly a week | https://bitquery.io/investigations/how-much-of-polymarket-is-bots (no independent source found) |
| 5 | 0x8dxd: Dec +$322k, Jan +$590k, Feb +$880k, Mar +$591k, then ≈$0 from Apr 2026; lifetime $2.05M on $200.6M; "$313 start" unverified | **partly** | The monthly figures reproduce exactly from `user-pnl-api` (+321.7k / +589.8k / +880.1k / +591.2k / Apr +0.2k / May–Oct within ±$0.4k). **But "$2.05M / $200.6M" is the all-time *leaderboard* figure.** The cited `user-pnl-api` gives a cumulative **$2.38M**, which is the sum of the months listed. The "$313 start" stays [C]. Finbold's 9 Jan 2026 piece says "$300". Derived and speculative: the pnl-api series *opens* at a cumulative PnL of $313.51 on 2025-12-05, so "$313" may be day-1 PnL misread as a deposit. The first trades on 2025-12-04 were $1–14 | `data-api…/v1/leaderboard?timePeriod=all&user=0x63ce…ba9a` (pnl $2,050,412, vol $200,578,645); https://finbold.com/polymarket-trading-bot-that-turned-300-into-400000-is-now-up-this-much/ |
| 6 | automatedAItradingbot: weather, median trade $1.79, $88k lifetime on $3.49M, peak ~$21k (Nov 2025), ≈$0 from Jun 2026, wallet holds $10.92 | **partly** | Confirmed: leaderboard all-time $88,259 on $3,486,030; pnl-api $88,950; Nov-25 +$21.4k; May +$0.8k, Jun +$0.2k, Jul +$0.1k, then ≈$0; trades are weather markets (Seoul, HK, Moscow, Qingdao, London, Shenzhen). **Off:** the median trade does not reproduce. It is $1.96 over the last 300 trades, $1.87 over the last 500 and $2.24 over the last 3,500 (largest of the last 300 is $256.89 ✓). **$10.92 is the value of open positions** (`/value` = $10.915), not cash; its on-chain USDC.e balance is 0 and its pUSD balance was not checked. The **last trade was 2026-07-14**; the 2026-09-28 activity is a $1.09 REDEEM | `data-api.polymarket.com/activity?user=0xd8f8…0f11&type=TRADE` ; `data-api.polymarket.com/value?user=0xd8f8…0f11` |
| 7 | Prediction Arena ($10k per model, 12 Jan–9 Mar 2026): every Cohort 1 model lost 16.0–30.8% on Kalshi; Polymarket averaged −1.1% | **confirmed** | The numbers match the abstract and tables. **Caveats:** the $10k funded the *Kalshi* accounts. The Polymarket live leg ran only **9 Feb–9 Mar** (28 days), with individual returns from −2.68% to 0.00%, and at least one model showed "minimal activity". So −1.1% mostly reflects low activity and is not evidence of near break-even skill | https://arxiv.org/abs/2604.07355 ; https://econpapers.repec.org/paper/arxpapers/2604.07355.htm |
| 8 | Arbitrageurs extracted ~$39.59M (Apr 2024–Apr 2025); top account $2.01M over 4,049 transactions | **confirmed** | Exact figures: $39,587,585.02 total, which the paper says "assum[es] ε = $1 profit per trade", so it is a heuristic estimate; top account $2,009,631.76 over 4,049 txs; window 1 Apr 2024–1 Apr 2025. Side note on §4c: the multi-condition legs sum to **$29.02M** (11.09 + 0.61 + 0.004 + 17.31, derived), not $28.28M | https://arxiv.org/html/2508.03474v1 ; peer-reviewed version AFT 2025 (LIPIcs 354, paper 27): https://drops.dagstuhl.de/storage/00lipics/lipics-vol354-aft2025/html/LIPIcs.AFT.2025.27/LIPIcs.AFT.2025.27.html |
| 9 | Bots made ~$1.2M in <$10k markets vs ~$35.1M in >$10M markets (Nov 2022–Feb 2026); >80% of sub-$10k volume is bots; ~70% of closed markets had <$10k volume | **confirmed** | CNBC states all of it. Bots are defined as >50 trades/day or >1,000 trades in total. "~70%" covers 2021 to end-May 2026 and uses double-counted "reported" Gamma volume. Della Vedova's SSRN abstract has bots earning **$133M in total** and retail losing $79M over 222M trades, so the shallow and deep buckets together ($36.3M) are only about 27% of bot profit (derived) | https://www.cnbc.com/2026/07/02/prediction-markets-mostly-have-thinly-traded-contracts-.html ; https://papers.ssrn.com/sol3/papers.cfm?abstract_id=6191618 |
| 10 | Taker rates: crypto 0.07 (**$1.75 per $100 at p=0.5**), sports/weather/culture/economics 0.05, politics/finance/tech 0.04; geopolitics free; makers pay 0 and get 15–25% rebates | **partly** — the "$1.75 per $100" element is **refuted** | The rates match the docs table (also Other 0.05, Mentions 0.04). **Wrong:** the docs' $1.75 is for **100 shares at $0.50, i.e. $50 of trade value**. Per $100 of notional at p = 0.5 the crypto fee is **$3.50 (3.5%)**. Fee as a share of notional = feeRate × (1 − p) (derived from `fee = C·rate·p·(1−p)`), so it is **highest on cheap contracts**: a 4¢ weather bucket pays ≈4.8% and a crypto one ≈6.7%. The §5 and §9 "1.75%" figures are therefore understated about 2× at p = 0.5, and more for long-tail buys. Rebates are **15–25% of the taker fees collected** (crypto 20%, sports 15%, others 25%), paid daily in pUSD to makers by their share of liquidity taken, not a rebate on maker volume. Sports was 0.03 at the 30 Mar 2026 rollout and is now 0.05 | https://docs.polymarket.com/trading/fees (fee table, 100-share example) ; https://docs.polymarket.com/programs/maker-rebates ; https://www.crowdfundinsider.com/2026/03/268884-polymarket-to-impose-taker-fees-on-nearly-all-trading-categories/ |

**Impact on this report.** The fee correction (claim 10) changes the edge arithmetic. Use `fee/notional = rate × (1 − p)` in the edge-after-fees code, not a flat 1.75%. All other headline conclusions stand (most wallets lose; profit sits with makers; niche bot edges decayed within about 4–5 months). Re-pull commands are in "Raw pulls" above, plus `data-api.polymarket.com/value?user=<wallet>`.
