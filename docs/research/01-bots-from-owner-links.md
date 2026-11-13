# 01 · Bots from the owner's links: what they do, what is verified, what to adopt

> Research track 1 of the design pack. Prepared 2026-10-07. Access date for every source is **2026-10-07** unless a row says otherwise.
> Method: read-only. Public APIs (`gamma-api`, `data-api`) via curl. Code was read through the GitHub API, never cloned or run. No logins, no orders.
> Labels: **verified** = pulled from Polymarket public API or read in source code; **claimed** = vendor or catalog text, not checked; **derived** = my computation from verified data; **unverified** = could not confirm.
> Source keys like `[S5]` resolve in the table at the end.

## 0. TL;DR

1. **Only two bots in this set have verifiable profits, and neither is an LLM bot.**
   - `@automatedAItradingbot` is a **weather-nowcasting bot** run by a self-described meteorologist: **+$88.3k all-time PnL** over 20 months, with 75% of realized PnL from temperature markets **(verified)** [S20–S24].
   - PolymTradeBot's public account `@98euf98a` is a **5/15-min crypto microstructure bot**: **+$64.4k on $9.0M volume** since 2026-05-07 **(verified)** [S29].
   - No bot here has a verified track record of LLM-driven profit.
2. **`MrFadiAi/Polymarket-bot` is gone.** The account and repo return 404 (GitHub user id 73393700) **(verified)** [S10]. I reviewed its code through forks.
   - The code has **no LLM**: it does copy-trading, YES/NO-sum arbitrage, "DipArb" and Binance trend-following.
   - Its own audit file admits that until 2026-09 the risk layers **never fired for copy-trading**, because copy-trade PnL was always recorded as $0, and that real arbitrage opportunities were **zero** [S13].
   - No key exfiltration was found in the fork I read. Do not run it anyway.
3. **The Polymarket-bot ecosystem on GitHub and npm is hostile.** Known campaigns since 2026-02 include 20+ fake bot repos with typosquatted npm stealers [S40] and 9 key-stealing npm packages published within 2 minutes [S42]. The Polycule Telegram bot was hacked for $230k [S43]. Two repos from the owner's link set have also been deleted (MrFadiAi, Trust412).
4. **The catalog and "top-10" pages are mostly SEO and affiliate content.**
   - The PolyBot Pro listing contradicts the real product's pricing, and every link on it is a Telegram referral [S3][S31].
   - QuickNode's list includes a Kalshi-only tool and a Solana DEX aggregator. Its "73% of arb profits" figure has no source [S1].
5. **Weather markets now carry taker fees.** The market schema shows `feeType: weather_fees` with rate 0.05, `takerOnly` and a 25% maker rebate **(verified)** [S27]. At 1–2¢ entries, the fee is about 4.9% of notional (**derived** from the `fee = C·rate·p(1−p)` formula in `05-venue-technical-facts.md`). This hurts the longshot-taker style that made `@automatedAItradingbot` money. Its activity fell 91% from April to May 2026 (**verified** counts). The cause is **unverified**.
6. **Best practices worth taking** come from the few serious codebases: poly-maker [S16], Chainstack PolyClaw/Alphapoly [S14][S15], Win Weather Bot's published method [S33], and the *post-mortem* of MrFadiAi's bugs. The ranked list is in §6.

---

## 1. Summary matrix (all bots reached from the owner's links)

| Bot | Listed in | Type / strategy | Open source | LLM | Paper/demo | Custody | Pricing | Performance | Red flags |
|---|---|---|---|---|---|---|---|---|---|
| **@automatedAItradingbot** | owner link 5 | Weather temperature-bucket nowcasting, plus esports/UFC/cricket | No (wallet only) | **No evidence** (name only) | n/a | self | n/a | **Verified**: +$88.3k all-time, 18/18 positive months Feb-25→Jul-26 [S22][S24] | Inactive since 2026-07-14. Value now $10.92 [S21] |
| **MrFadiAi/Polymarket-bot** | owner link 4 | Copy-trade + YES/NO arb + DipArb + trend | Was MIT; **repo deleted** | **None in code** | `DRY_RUN=true` default. Paper PnL booked at detection | raw private key in `.env` | Free | None. Own audit: "Zero real arb opportunities" [S13] | Account deleted. Risk layers broken until v3.2 (§3.2) |
| **PolyBot Pro** (= @TradePolyBot / polybot.trading) | owner link 2 | Telegram copy-trading + TP/SL/trailing rules | No | Catalog says "AI-powered predictions"; vendor site mentions none | Not mentioned | Safe wallet, owner key exportable (claimed) | Catalog: Free / $29 / $99 per month. Vendor: **1% of successful trades** [S3][S31] | None | Listing ≠ product. All links are referral `ref_5sqbcpu` |
| PolyCop | QuickNode, PolyCatalog | Telegram sub-second wallet copy, limit-copy | No | No | No | Key "generated within the user's Telegram session" (claimed) | **0.5%** per trade [S5] | None | Copy latency vs paid fees |
| Polycool | QuickNode | Telegram copy-trading + whale/insider alerts | No | No | No | Not stated | Not stated | None [S6] | Opaque custody |
| Polygun | QuickNode | Telegram trade/copy/limit, multichain bridging | No | No | No | Non-custodial (claimed) | **1%** per tx [S7] | None | — |
| Stand.trade | QuickNode, PolyCatalog | Kalshi+Polymarket terminal, whale tracking/copy | No | No | No | Deposit into platform | Not stated | "$200M in annualized trading volume" (claimed) [S9] | Custodial-style deposits |
| Fireplace | QuickNode | Multi-venue terminal, smart order routing | No | No | No | "Enclave Money" wallets | Free | None. "$1.5 million pre-seed" (claimed) [S9b] | — |
| FORS | QuickNode | **Solana DEX** aggregator, TP/SL, copy | No | No | No | n/a | Not stated | None [S9c] | Not a Polymarket bot |
| **PolyClaw (Chainstack)** | QuickNode | OpenClaw skill: browse/trade + **LLM hedge discovery** | **Yes**, Apache-2.0, 360★ | **Yes**, OpenRouter LLM for logical-implication pairs | No | key in env, "small amounts only" | Free | None | Agent framework, unaudited (README says so) [S14] |
| Polybot (QuickNode entry) | QuickNode | Listing describes a **CEX** grid/scalp bot, but its X handle is @TradePolyBot | No | No | No | API keys | Not stated | None [S9d] | Listing inconsistent with PolyBot Pro |
| **poly-maker** (warproxxx) | PolyCatalog | Maker-only MM on political markets, CLOB V2 | **Yes**, MIT, 1,505★ | No | **`--paper`** full pipeline on live feed | own key | Free | README: "can lose money" [S16] | None found (metadata only, code not audited) |
| **Win Weather Bot** | PolyCatalog | Weather-market edge: 138-run model ensemble vs mid | No (signed Windows binary) | No | "Mock data" preview | Local keys (claimed) | $59/mo … $39/mo annual [S33] | One testimonial "bankroll up 14%" (claimed) | No track-record wallet |
| **PolymTradeBot** | PolyCatalog | 5/15-min crypto Up/Down: directional, maker, two-sided | Source sold ($499/$999) | No | **Paper on live markets** (claimed) | local key | $499 one-time / $999 Pro [S28] | **Verified account** @98euf98a: +$64.4k on $9.0M volume [S29] | Live account not proven to run the sold code |
| PolyGoat | PolyCatalog | 5/15-min crypto "temporal arbitrage", ~142 ms | **"Open source" = README only (8 KB, 1 file)** | No | No | **Custodial** (Fireblocks vaults) | 10% of profits [S30] | None | Custodial + fake open-source |
| Polytragent | PolyCatalog AI agents | LLM (Claude Sonnet) research per sub-market vs Manifold, calibration correction | No | **Yes** | No | n/a (advice only) | $99/mo | "78% prediction win rate" (claimed) [S34] | Win rate without price context is meaningless |
| Polymarket Prediction Bot (Kir Chain Labs) | PolyCatalog | "AI" signals on 5m/15m/1h/1d | No | "AI" | Free demo | Unknown | Contact sales | "80–90% accuracy" (claimed) [S35] | Implausible accuracy claim |
| ArbitradePro | PolyCatalog | Web bot: **Martingale**, Soros, trailing stops | No | "AI auto-trade" | Unknown | Unknown | 0.95% "trading tax" [S36] | None | Martingale |
| Polymarket Copy Trading Bot v1 (Trust412) | PolyCatalog open-source | Copy one target wallet | Was on GitHub; **account now 404** | No | Unknown | key | Free | None [S37] | Deleted account |
| Polycule | PolyCatalog | Telegram trading bot | No | No | No | **Custodial; hacked** | — | — | **$230k stolen, 2026-01-07** [S43] |
| Others (no deep dive): Polyspark, PolyFocus, VictorAI, PolyxBot, BetMoar, Bankr, Ultramarkets (10× leverage), PolySpike Trader, Poly Sniper (14-day trial), TradeFox, Bullpen, trade.fun, TatorTrader | PolyCatalog | Telegram execution, copy-trading, terminals, leverage | Mostly no | Mostly no | Mostly no | Telegram/custodial | Fee-based | None | Same pattern: closed, fee-taking, no verifiable PnL [S2] |

PolyCatalog counters are unlabeled. They appear to be vote score and page views, **not** user counts; for example "0 / 307" for TradeFox [S2]. The category page header says "31 projects found" while its meta says 21 [S2].

---

## 2. Owner link 5: `@automatedAItradingbot` (verified on public APIs)

### 2.1 Identity and headline numbers (verified)

| Field | Value | Source |
|---|---|---|
| Proxy wallet | `0xd8f8c13644ea84d62e1ec88c5d1215e436eb0f11` | profile page embedded data [S20], gamma public-profile [S21] |
| Bio | "Meteorologist.IT engineer.Automated bot testing" | [S21] |
| Account created | 2025-01-31 | [S21] |
| Profile stats | `trades` 3,433 (= markets traded; `/traded` also returns 3,433), `largestWin` $8,741.30, `volume` $3,486,030, `headlinePnl` $88,934.83, views 92,387 | [S20], [S21b] |
| Leaderboard ALL | rank 2,126, PnL **$88,259.38**, vol $3,486,030 | [S22] |
| Leaderboard MONTH / WEEK / DAY | −$3.82 / +$0.18 / +$0.00 (inactive) | [S22] |
| Current portfolio value | **$10.92** | [S21c] |
| Taker tier | "Tier 0" | [S21] |
| Twin account `automatedAItradingbot2` (`0x30d6…95b8`) | 0 markets traded, so not relevant | [S21], [S22] |

### 2.2 Activity profile (verified raw, derived aggregates)

I pulled the full `/activity` history by paging backwards on `end=` timestamps, because offset caps at about 5,000. That gave **36,058 events from 2025-01-31 to 2026-09-28** [S23]. Counts below are **derived** from that pull.

| Event type | Count | USDC |
|---|---|---|
| TRADE | 32,185 (24,411 BUY / 7,774 SELL) | $365,129 (buy $249,933, sell $115,195) |
| REDEEM | 1,729 | $157,024 |
| SPLIT / MERGE / CONVERSION | 582 / 657 / 824 | $135,025 / $190,599 / — |
| MAKER_REBATE + REWARD + YIELD + TAKER_REBATE | 25 + 10 + 45 + 1 | $113.90 total |

- The headline "volume" ($3.49M) is about **9.5×** the TRADE USDC actually exchanged ($365k). It sits close to total shares bought on closed positions (3.76M) (**derived**). Treat Polymarket's profile "volume" as a share or notional count, not cash turnover (**unverified** definition).
- Trading days: 468. Median **35 fills/day**, maximum **1,450 fills/day**. Fills spread across all 24 UTC hours, which is consistent with automation (**derived**).

**Monthly trades and realized PnL** (derived; realized PnL bucketed by the closed-position timestamp [S24]):

| Month | Trades | Buy+sell USDC | Realized PnL |
|---|---|---|---|
| 2025-02 | 1,055 | 15,709 | +4,343 |
| 2025-03 | 1,355 | 21,044 | +6,409 |
| 2025-04 | 141 | 1,292 | +364 |
| 2025-05 | 1,451 | 13,621 | +4,999 |
| 2025-06 | 806 | 5,452 | +6,785 |
| 2025-07 | 833 | 12,885 | +284 |
| 2025-08 | 1,468 | 28,867 | +3,918 |
| 2025-09 | 542 | 2,490 | +2,244 |
| 2025-10 | 734 | 4,122 | +8,203 |
| 2025-11 | 1,751 | 24,219 | +11,096 |
| 2025-12 | 1,415 | 15,093 | +12,310 |
| 2026-01 | 4,175 | 48,957 | +3,033 |
| 2026-02 | 4,932 | 54,206 | +8,845 |
| 2026-03 | 3,692 | 36,826 | +6,933 |
| 2026-04 | 5,086 | 61,123 | +7,784 |
| 2026-05 | 461 | 3,383 | +401 |
| 2026-06 | 1,913 | 13,125 | +4,556 |
| 2026-07 | 374 | 2,688 | +763 |
| 2026-08 / 09 | 0 / 0 | 0 / 0 | −119 / −473 |

- **18 of 18 trading months were positive.** The average was **≈ $5.2k/month realized** (derived: $93.3k ÷ 18).
- Activity collapsed after April 2026, from 5,086 trades to 461. That coincides with the **CLOB V2 / pUSD cutover on 2026-04-28** [S15] and with weather markets now being fee-bearing [S27]. The causal link is **unverified**, and I could not find when weather fees started.

### 2.3 What it trades (derived; keyword classifier on `eventSlug`)

| Category | Trades | Trade USDC | Closed positions | Realized PnL |
|---|---|---|---|---|
| **Weather** (temperature buckets) | 17,722 | $160,276 (44%) | 6,170 | **+$69,125 (75%)** |
| Other (Honor of Kings esports, UFC, R6 Siege, SC2, macro) | 9,379 | $158,141 | 1,132 | +$17,448 |
| Cricket (incl. "who wins the toss") | 1,155 | $16,379 | 175 | +$3,212 |
| Esports (CS2/LoL/Dota/Valorant/MLBB) | 476 | $5,364 | 52 | +$1,151 |
| US sports | 647 | $11,443 | 66 | +$947 |
| Politics/geo | 1,267 | $5,289 | 87 | +$95 |
| Tennis / econ / crypto / soccer | 1,407 | $8,206 | 93 | +$700 |

- **Cities:** 42 in total. London 5,823 trades, NYC 5,609, Seoul 2,473, Hong Kong 802, Tokyo 480, then Amsterdam, Moscow, Qingdao, Paris, Shenzhen.
- **Timing:** this is nowcasting, not multi-day forecasting.
  - **92% of London buys (4,367 / 4,723) happen on the observation day itself.** 346 happen the day before.
  - Buys peak between 07:00 and 15:00 UTC, i.e. while the daily maximum is forming.
  - The bot is trading on live station observations plus intraday model runs.

### 2.4 Prices, sizes, win rate, edge (derived)

| Metric | Value |
|---|---|
| BUY price deciles | 0.002, 0.005, 0.011, 0.029, 0.07, 0.12, 0.50, 0.90, 0.91 |
| Share of buys < $0.10 / ≥ $0.90 | 55% (13,459) / 23% (5,734) |
| BUY size (USDC) | median **$1.60**, mean $10.24, p90 $18, max $1,994 |
| Weather BUY YES | median price 1.1¢ |
| Weather BUY NO | median price 3¢ |
| Weather SELL YES | median 9.1¢: sells into repricing before resolution |
| Closed positions | 7,775. Profitable 1,661, losing 4,908, so **25.3% position win rate** |
| Average win / average loss | **+$127.92 / −$24.41** (payoff ratio 5.2) |
| Realized total / cost basis | +$92,678 / $371,245, so **ROI on cost 25%** |
| **Weather longshots (entry < 10¢)** | 4,359 positions. Cost $41,791. **PnL +$65,272 (+156%)**. Cost-weighted entry **1.54¢**. **7.3% resolved YES** (319) |
| Weather favourites (entry ≥ 90¢) | 51 positions. Cost $47,703. PnL **−$500**: no edge buying favourites |
| Entry = 0.50 bucket | 922 positions, −$18,476. These are artifacts of SPLIT, where the YES and NO legs book at 0.50. Example: Hong Kong 2026-03-13 made +$8,741 on one leg and −$6,960 on the other. Read them as pairs, not as losses. |

### 2.5 Inferred strategy

The strategy is inferred from behaviour; no code is public.

1. Several times a day, on the observation day, it builds a probability distribution for the daily maximum temperature at each market's resolution station. It probably uses model ensembles plus the running station observations. Its bio and the WinWeatherBot method [S33] support this, but it is **unverified**.
2. It **buys tail buckets priced 0.2–5¢** whose model probability is several times the price. It also buys NO on buckets the observations have already killed.
3. Tickets are tiny (median $1.60), spread across many buckets and cities.
4. It **sells into repricing**: median weather YES sell price is 9.1¢ against a median buy of 1.1¢. Otherwise it redeems at resolution.
5. It also uses SPLIT/MERGE/CONVERSION on neg-risk events.
6. **No LLM is needed or evident.** The "AI" in the name is branding.

**Relevance to us:**

- This is the strongest verified evidence in this track that **domain-specific nowcasting with small tickets works on Polymarket at $1–20 sizes**.
- That size range fits a $50–150 budget better than anything else here.
- However, the 2026 weather taker fee (≈ 4.9% of notional at 1–2¢, **derived**) and the 0.01 tick on many weather books [S27] cut into the longshot-taker edge. **Maker-first entry** would be needed.

---

## 3. Owner link 4: `MrFadiAi/Polymarket-bot` (code review, not run)

### 3.1 Provenance

| Item | Finding | Source |
|---|---|---|
| Upstream status | `github.com/MrFadiAi` and `/Polymarket-bot`, plus `api.github.com/user/73393700`, all return **404 on 2026-10-07**. Firecrawl's cached copy was already a 404 on 2026-10-06. Reason unknown. | [S10] |
| Last known upstream state | v3.2 "Execution Safety & Backtesting", Sep 2026. 6 contributors. Author on X: @Mr_CryptoYT | [S11] |
| Fork reviewed | `khalefbleydjadja-hub/Polymarket-bot` at HEAD `3a04fc84` (2026-09-13). Its commits are authored "Fadi" and match the upstream history. Three other forks (`fabwebdev`, `developeralexandre`, `mamadwinx`) sit at `82647014` (2026-01-11, v3.1). | [S12] |
| What it really is | `package.json` name is `@catalyst-team/poly-sdk` 0.4.3 (MIT): a repackage of a Chinese team's TypeScript SDK (npm maintainer `hhh_qc`, created 2025-12-23), plus `bot-with-dashboard.ts` (70 KB) and a React dashboard. | [S12], [S12b] |

### 3.2 Strategy, LLM, paper mode, risk

| Aspect | Finding (verified in fork source) |
|---|---|
| **Strategies** | 1. **Smart-money copy-trading.** Picks leaderboard top-20 that pass 6 filters: win rate ≥ 60%, PnL ≥ $500, ≥ 30 trades, profit factor ≥ 1.5, consistency ≥ 0.7, and no single trade above 30% of PnL. Size scale 0.1; max per trade min($15, 2% of capital); 3% max slippage; 500 ms delay. **Enabled by default** (`SMARTMONEY_ENABLED !== 'false'`). The hard-coded follow list holds `beachboy4` (#18 all-time, $5.83M PnL) and a wallet with **0 trades** [S22]. <br>2. **YES+NO sum arbitrage.** ≥ 1% threshold, $20–100 size, min 24h volume $5k, net profit ≥ $0.50. <br>3. **DipArb on 15-min crypto Up/Down.** Leg 1 on a ≥ 15% ask drop inside a 3-second window. Hedge leg 2 when the pair sum ≤ 0.92, with a 60 s timeout and 20% stop. <br>4. **Direct trend.** Binance 15m klines; SL 15% / TP 25% / trail 10%; max hold 7 days. |
| **LLM use** | **None.** A grep of the bot entry points finds no openai/anthropic/gemini/groq/LLM calls. The "AI" is branding. |
| **Paper mode** | `DRY_RUN=true` by default. **Anti-pattern:** in dry run the arbitrage path books `estimatedProfit = size × profitPercent` **at detection**, with no fill, queue or partial-fill simulation. Paper PnL is therefore systematically optimistic. A dashboard button flips DRY↔LIVE at runtime. It is token-gated and bound to localhost by default. |
| **Risk controls (v3.2)** | Daily loss 5%; monthly 15%; drawdown from peak 25%; **total loss 40%, which permanently halts**; pause after 6 losses in a row; total exposure cap 30%; per-market caps. Dynamic sizing at 1–5% of capital: −20% per loss beyond 2, **+10% per win beyond 3** (a hot-hand boost). Other controls: PnL-drift reconciliation (|liquid + open − (baseline + tracked PnL)| > max($5, 2%)), POL/MATIC gas monitor, wallet circuit breaker after 3 failures, skip copies older than 5 s, worst-price caps from the live book, sequential arb legs with unwind on partial fill. |
| **Its own audit** | `docs/AUDIT_FIXES.md` tracks upstream issue #2, "AUDIT: Critical Profitability & Risk Management Issues". Until fixes on 2026-09-09 to 13 [S13]: **copy-trade profit was always $0**, so daily/monthly/drawdown limits could never trip for the main strategy. `canTrade()` only protected the Direct strategy. `calculatePositionSize()` was never called in production. Profit ignored fees and gas. There was no on-chain reconciliation. And: "**Zero real arb opportunities in testing** … pair cost ≥ 1.01 … treat as market-efficiency evidence." |
| **Backtest** | JSONL order-book replay with fee and gas modelling (`src/backtest/*`), added in v3.2. No published results. |

### 3.3 Security red-flag check (static only, 224 text files from the fork)

| Check | Result |
|---|---|
| Outbound hosts in `.ts`/`.tsx` | Polymarket (`gamma-api`, `clob`, `data-api`, `bridge`); Polygon RPCs (polygon-rpc.com, publicnode, drpc, ankr, llamarpc, Alchemy); Goldsky subgraph; `api.binance.com`; `api.monid.ai`, an optional third-party leaderboard provider called with *its own* API key. **No unknown POST endpoints, no Vercel/Workers C2 patterns.** |
| Obfuscation / dynamic exec | No `eval`, `new Function`, `atob`, `fromCharCode` or hex blobs. `child_process.spawn` appears only in `scripts/benchmark/run-all.ts` (spawns `npx tsx`). |
| Dependencies | Lockfile has 225 packages, **all** resolved from `registry.npmjs.org`. Install scripts only in `esbuild`, `fsevents` and vite's `esbuild`, all standard. `@catalyst-team/cache` 0.2.0 has no install script and depends only on `ioredis` [S12b]. No typosquats seen. |
| Key handling | Raw `POLYMARKET_PRIVATE_KEY` in `.env`. The README tells users to "Export from MetaMask". Not passed in dry run. |
| Verdict | **No exfiltration found in this fork.** The upstream is deleted, a fork is not the upstream, and forks of this repo name are a known malware vector [S40][S41]. **Read for ideas; never install.** |

**Features worth adopting from it:**

- The `RiskIntent → PreExecutionGuard` choke point.
- PnL-drift reconciliation.
- Stale-signal skip.
- Worst-price caps.
- Sizing floor (skip dust).
- Gas-balance pause.

Above all, take the lesson from its audit: **a risk limit you have not tested end-to-end against real PnL accounting is decoration.**

---

## 4. Owner links 1–3: catalog pages

### 4.1 QuickNode "Top 10 Polymarket Trading Bots 2026" [S1]

- **No publish date.** The article sells QuickNode RPC: "Start for free" calls to action, and "Tools using dedicated Polygon RPC nodes offer a significant advantage".
- It claims "73% of arbitrage profits are captured by sub-100ms execution bots" with **no source (unverified)**.
- 2 of its 10 entries are not Polymarket bots: TurbineFi is Kalshi, FORS is a Solana DEX aggregator [S8][S9c].
- Its useful content is generic: non-custodial custody, Safe wallets, oracle/UMA settlement risk, and the warning "No 'Undo' Button".
- The one substantive entry is **PolyClaw** (§4.3).

### 4.2 PolyCatalog: PolyBot Pro and the trading-bots category [S2][S3]

**PolyBot Pro listing:**

- Launch date "June 15, 2024".
- Rating "4.8/5" from **4 reviews**.
- Pricing "Free: up to 10 trades/day · Pro $29/month · Enterprise $99/month".
- Features "AI-powered predictions".
- **Every** external link (website, GitHub, docs, X, Discord) points to `t.me/TradePolyBot?start=ref_5sqbcpu`, which is an affiliate referral [S3].

**The real product behind @TradePolyBot (polybot.trading)** [S31]:

- Charges a **"1% flat fee on successful trades"**.
- Offers Telegram copy-trading, TP/SL/trailing and limit orders.
- Uses a Safe wallet with an exportable owner key (claimed).
- Has **no AI** and no performance numbers.

**Conclusion:** the listing is inaccurate, and no verifiable performance exists.

**Category observations:**

- About 2/3 of entries are Telegram execution or copy-trading front-ends that take 0.5–1% per trade or 10% of profits.
- Open-source entries are few: poly-maker, the Trust412 copy bot (deleted), and PolyGoat. PolyGoat's "open source" repo is a single 8 KB README [S30].

### 4.3 Notable entries worth a closer look

| Entry | Why it matters to us | Key facts |
|---|---|---|
| **poly-maker** [S16] | Best-engineered OSS reference on the list. V2-aware, Python, uv, tests. | Maker-only (post-only) quotes. Fair value: reservation price `r = FV − skew(inventory)`; half-spread `δ = base + c_vol·σ + c_tox·toxicity`. Regime machine: QUIET / TRENDING / REDUCE_ONLY near end date / HALTED on stale data or kill switch. Risk: per-market notional cap, **neg-risk event-group worst-case cap**, total exposure cap, daily-loss kill switch, WS-staleness halt. **Exchange heartbeat dead-man switch.** Periodic REST reconcile. Raw WS/order events journaled for replay. `run --paper` runs the full pipeline against the live feed. 83 tests, mypy strict. Ranks markets by liquidity rewards and **maker rebates**. |
| **Chainstack PolyClaw / Alphapoly** [S14][S15] | LLM used where it is strong (logic), not for forecasting. | An LLM extracts **logical implications** between markets, and a second LLM pass validates them at market level. "Only logically necessary implications are accepted — correlations and 'likely' relationships are rejected." Coverage tiers T1 ≥ 95%, T2 90–95%, T3 85–90%. Trades by split-then-sell-the-unwanted-side. Alphapoly README: V1 (USDC.e) retired 2026-04-28. Uses **deposit wallets (signature type 3)** and a gasless relayer. "Bare EOAs can't trade directly" on the V2 CLOB. |
| **Win Weather Bot** [S33] | Commercial version of what @automatedAItradingbot does. | "138 total numerical weather prediction runs: four ensemble models (GFS, ECMWF, ICON, NOAA) plus 134 deterministic runs". Ingest every 5 min. Models weighted by historical accuracy per region and horizon. Edge = ensemble probability − mid > threshold. "Hard caps per city, per region, per direction". **Circuit breaker that "halts new entries above 3σ ensemble spread"**. 67+ cities. Markets keyed by resolution station (e.g. LFPB). No verified track record. |
| **PolymTradeBot** [S28][S29] | Verified example of what microstructure bots earn per dollar of volume. | @98euf98a (`0xdf4c…4f22`), joined 2026-05-07. 54,347 trades. Profile `headlinePnl` $86,823 vs leaderboard ALL **$64,401** on **$9.0M volume** (≈ 0.7% margin, **derived**). Last 30 days: **+$10,602 on $1.05M**. Portfolio value $3,829. Taker tier 6. That this account runs the $499 code is **unverified**. |
| **Polytragent** [S34] | Closest commercial analogue to our P1 (LLM research per sub-market). | "Claude Sonnet AI synthesis" in an 8-step pipeline. Cross-venue anchor: Manifold. "Calibration bias correction". Self-tracks predictions and resolutions every 6 h. "78% prediction win rate" is **claimed**, and win rate without entry-price context says nothing about edge. |

---

## 5. Security and integrity landscape (affects how we build)

| Incident | Date | Facts | Source |
|---|---|---|---|
| Hijacked `dev-protocol` GitHub org | reported 2026-03-15 | 20+ fake Polymarket bot repos with inflated stars. Typosquats `ts-bign` and `big-nunber` pull in `levex-refa` (steals `.env` and wallet files) and `lint-builder` (postinstall SSH backdoor, `ufw allow 22/tcp`). C2 hosts: `cloudflareguard.vercel.app`, `cloudflareinsights.vercel.app`. | [S40] |
| "Why almost every Polymarket bot on GitHub is malicious" | 2026-05-27 | Red flags: typosquatted deps, postinstall scripts, star farms, deleted warning issues, polished branding. | [S41] |
| npm campaign `texsellix` / `polymarketdev` | 2026-05-20 | 9 packages published in about 2 minutes, e.g. `polymarket-claude-code` (MAL-2026-4212). Reads `PRIVATE_KEY` or prompts for it, then sends it to a Cloudflare Worker. Targets **AI-assisted coders** who accept package suggestions. `texsellix/polymarket-trading-bot` also appears on skill aggregators with a `paperTrading: true` lure [S44]. | [S42] |
| Polycule Telegram bot hack | 2026-01-07 | $230k of user funds lost. Withdrawals frozen; rug accusations by 2026-01-12. | [S43] |
| Deleted accounts in this track | by 2026-10-07 | MrFadiAi (repo of owner link 4) and Trust412 (PolyCatalog "Copy Trading Bot v1"). | [S10], [S37] |

**Design consequences (for `04-risk-and-custody.md`):**

1. Use only official SDKs: `polymarket` py-sdk / `py-clob-client-v2`.
2. Lock dependency hashes (`uv lock`) and **never install third-party "Polymarket bot" packages**, including ones an AI assistant suggests.
3. Run `pip`/`npm` with no install scripts on the key-holding isolated server.
4. Hold only a float in the wallet.
5. Keep the signing key in a root-only file, never in the repo or argv.
6. Keep Telegram for alerts only, never for custody.

---

## 6. Ranked features and practices to adopt

Ranked by (impact on not losing money + impact on finding edge) ÷ build cost for a $50–150 bot.

| # | Feature / practice | Evidence (who does it, what broke without it) | Where it lands in our design |
|---|---|---|---|
| 1 | **Supply-chain and key hygiene.** Official SDK only, hash-locked deps, no third-party bot code on the key host, float-only deposit wallet, key outside the repo. | §5 campaigns. MrFadiAi and Trust412 deleted. Polycule hack. | `04` §1, `CLAUDE.md` hard rules |
| 2 | **One pre-execution risk choke point** that every order passes through: `OrderIntent → allow/block(reason)`. Covers *all* strategies, plus **tests proving each limit trips on real PnL accounting**. | MrFadiAi: copy PnL was $0, so limits never fired, and `canTrade()` guarded only 1 of 4 strategies [S13]. | `02` engine/risk module, test plan |
| 3 | **Honest paper trading.** Run the live-feed pipeline and simulate fills by walking the book: fees from `feeSchedule`, 5-share minimum, tick, latency re-snapshot, partial fills. **Never book PnL at signal time.** | MrFadiAi dry-run books PnL at detection. poly-maker `--paper` runs the full pipeline. No official sandbox exists (`05` F1). | `05` Phase 1 paper engine |
| 4 | **Fee-aware net-edge gate plus maker-first execution.** Post-only limit at fair value minus margin; taker only if net edge > fee + spread + buffer. Count maker rebates and liquidity rewards as a bonus, not a reason to trade. | Weather/econ fee rate 0.05 taker-only, 25% maker rebate [S27]. At 1–2¢ the taker fee is ≈ 4.9% of notional (derived). poly-maker scores rebates. | `02` edge calculator, order policy |
| 5 | **Reconciliation and staleness halts.** Compare tracked vs on-chain/venue PnL (drift limit). Halt on WS staleness. Periodic REST reconcile of orders and positions. | MrFadiAi v3.2 `checkPnlDrift`. poly-maker HALTED regime. | `02` reconciler |
| 6 | **Correlation-aware exposure caps:** per event / neg-risk group (worst case across mutually exclusive outcomes), per theme, per direction, plus a total cap. | poly-maker neg-risk group cap. WinWeatherBot per city/region/direction. | `04` sizing caps |
| 7 | **Exchange heartbeat dead-man switch.** Cancel-all on disconnect or crash; REDUCE_ONLY near end date. | poly-maker. | `02` executor |
| 8 | **Uncertainty circuit breaker.** No new entries when forecast dispersion is high: ensemble spread > 3σ, or LLM-ensemble disagreement above a threshold. | WinWeatherBot 3σ rule. Maps directly to our multi-LLM dispersion. | `02` gating |
| 9 | **Forward calibration loop.** Log every forecast with market price at that time; resolve outcomes; refit calibration (Platt/isotonic); report Brier and log-loss vs the market. A win rate alone means nothing. | Polytragent claims calibration correction. Win-rate marketing (78%, 80–90%) shows why win rate misleads. Our design already plans this. | `03` P1 test method |
| 10 | **Journal raw market and order events to disk** for deterministic replay backtests. | poly-maker journal. MrFadiAi JSONL replay. | `02` storage |
| 11 | **Stale-signal and worst-price protection.** Drop any signal older than N s; every order carries a limit price derived from the current book. | MrFadiAi v3.2 (5 s stale skip, price caps). | `02` executor |
| 12 | **Sizing floor and dust skip.** If the Kelly size is below the venue minimum (5 shares), skip the trade instead of rounding up. | MrFadiAi v3.2. `orderMinSize: 5` [S27]. | `02` sizing |
| 13 | **Candidate strategy "W1: weather nowcasting"** (deterministic, LLM only for parsing rules and stations). Small tickets across many buckets, observation-day entries, maker-first, sell into repricing. | The only verified profitable small-ticket pattern here: +$69k on weather, median ticket $1.60 (§2). Commercial clone exists [S33]. | `03` new strategy, design and test later |
| 14 | **Candidate strategy "L1: LLM logical-implication hedges".** The LLM finds *necessary* implications between markets; deterministic code prices the covering portfolio. | Chainstack PolyClaw/Alphapoly [S14][S15]. Uses the LLM for logic, which is its strength. | `03` deferred strategy |
| 15 | **Do NOT adopt:** win-streak size boosts or Martingale; leaderboard copy-trading through fee-taking Telegram bots (latency plus 0.5–1% fees); custodial "execution layers"; YES+NO-sum arbitrage at retail latency (MrFadiAi found zero opportunities). | §3.2, §4.2, [S30], [S36] | `03` rejected list |

## 7. What this means for the owner's goals

- **Is there evidence that LLM bots make money?** Not in this link set. The verified earners are a domain specialist (weather) and a high-volume microstructure bot. Every "AI" claim with a number (78%, 80–90%) is unverified and uses win rate, which says nothing about edge. P1's LLM edge has to be **proven by our own forward calibration**, not borrowed from marketing.
- **How large is "$5–10k/month"?**
  - @automatedAItradingbot averaged ≈ $5.2k/month realized on ≈ $250k of buys over 20 months. Its capital is unknown, because the data API does not expose deposits.
  - @98euf98a made ≈ $10.6k in 30 days on $1.05M of volume (≈ 1% margin) (**derived**).
  - Both imply five- to six-figure turnover per month or a real information edge. Neither is reachable from $50–150 in the near term. The realistic first goal is **positive net edge per trade with calibrated forecasts**, then scaling.
- **Platform facts that change the build** (detail in `05-venue-technical-facts.md`):
  - CLOB V2 with pUSD collateral since 2026-04-28.
  - Deposit wallets (signature type 3) via the relayer.
  - Fee schedules per category: weather/econ 0.05, taker-only, 25% rebate.
  - Minimum order 5 shares.
  - No official paper mode.

---

## Sources (all accessed 2026-10-07 unless noted)

| Key | URL / endpoint | Notes |
|---|---|---|
| S1 | https://www.quicknode.com/builders-guide/best/top-10-polymarket-trading-bots | No date. "73% of arbitrage profits…" unsourced |
| S2 | https://www.polycatalog.io/polymarket-trading-bots (and `?page=2`, `?page=3`, `/polymarket-ai-agents`, `/polymarket-trading-bots/open-source`) | Counters unlabeled |
| S3 | https://www.polycatalog.io/project/polybot-pro | Pricing $29/$99, 4 reviews, referral links |
| S4 | https://www.quicknode.com/builders-guide/tools/predictengine-by-predictengine | |
| S5 | https://www.quicknode.com/builders-guide/tools/polycop-by-polycop | 0.5% fee |
| S6 | https://www.quicknode.com/builders-guide/tools/polycool-by-polycool | |
| S7 | https://www.quicknode.com/builders-guide/tools/polygun-by-polygun | 1% fee |
| S8 | https://www.quicknode.com/builders-guide/tools/turbinefi-by-turbinefi | Kalshi |
| S9 | https://www.quicknode.com/builders-guide/tools/stand-by-stand | $200M annualized (claimed) |
| S9b | https://www.quicknode.com/builders-guide/tools/fireplace-by-fireplace | |
| S9c | https://www.quicknode.com/builders-guide/tools/fors-by-fors-market | Solana |
| S9d | https://www.quicknode.com/builders-guide/tools/polybot-by-polybot | |
| S10 | https://github.com/MrFadiAi/Polymarket-bot, https://github.com/MrFadiAi, https://api.github.com/user/73393700 | All HTTP 404. Firecrawl cache 404 at 2026-10-06T18:56Z |
| S11 | Firecrawl search index excerpt of the upstream README ("v3.2 - Execution Safety & Backtesting", "Contributors 6"); https://trendshift.io/repositories/35549 | Pre-deletion snapshot |
| S12 | https://github.com/khalefbleydjadja-hub/Polymarket-bot (HEAD 3a04fc84); https://github.com/fabwebdev/Polymarket-bot (82647014) | Read via GitHub API; files `bot-with-dashboard.ts`, `src/utils/risk.ts`, `.env.example`, `package.json`, `package-lock.json` |
| S12b | https://registry.npmjs.org/@catalyst-team/poly-sdk, https://registry.npmjs.org/@catalyst-team/cache | Maintainer hhh_qc, no install scripts |
| S13 | `docs/AUDIT_FIXES.md` in S12 fork (refs upstream issue MrFadiAi/Polymarket-bot#2) | Audit items #1–#7 |
| S14 | https://github.com/chainstacklabs/polyclaw (README; 360★, Apache-2.0, pushed 2026-04-28) | |
| S15 | https://github.com/chainstacklabs/polymarket-alpha-bot (README; 180★, pushed 2026-08-05) | V2/pUSD, deposit wallets |
| S16 | https://github.com/warproxxx/poly-maker (README; 1,505★, MIT, pushed 2026-07-09) | |
| S20 | https://polymarket.com/@automatedaitradingbot (embedded `user-stats`, dataUpdatedAt 2026-10-07 08:20 UTC) | |
| S21 | https://gamma-api.polymarket.com/public-profile?address=0xd8f8c13644ea84d62e1ec88c5d1215e436eb0f11 | Also `0x30d66c51815e20d98580c7ac79357564f37d95b8` |
| S21b | https://data-api.polymarket.com/traded?user=0xd8f8…0f11 | 3,433 |
| S21c | https://data-api.polymarket.com/value?user=0xd8f8…0f11 | $10.915 |
| S22 | https://data-api.polymarket.com/v1/leaderboard?timePeriod={DAY,WEEK,MONTH,ALL}&orderBy=PNL&user=… | Also used for beachboy4 `0xc2e7…be51` and `0x58c3…de74` |
| S23 | https://data-api.polymarket.com/activity?user=0xd8f8…0f11&limit=500&sortBy=TIMESTAMP&sortDirection=DESC&end=… | 36,058 events, paged back by `end` |
| S24 | https://data-api.polymarket.com/closed-positions?user=0xd8f8…0f11 (7,775 rows); `/positions?sizeThreshold=0` (348 rows) | |
| S27 | https://gamma-api.polymarket.com/events?tag_slug=weather&closed=false (e.g. `highest-temperature-in-denver-on-october-9-2026`) | `feeType weather_fees`, `feeSchedule {rate 0.05, exponent 1, takerOnly, rebateRate 0.25}`, `orderMinSize 5`, tick 0.01, NOAA KBKF resolution with WU fallback |
| S28 | https://www.polycatalog.io/project/polymtradebot | $499/$999, paper mode claimed |
| S29 | https://polymarket.com/@98euf98a; gamma public-profile and data-api leaderboard/value for `0xdf4c6a942bd95bf903d6066b4ba7051e6f914f22` | |
| S30 | https://www.polycatalog.io/project/polygoat; https://github.com/cronuscode-cloud/polygoat-polymarket (1 file, README 8,417 B) | |
| S31 | https://polybot.trading/ | 1% on successful trades, Safe wallet |
| S33 | https://winweatherbot.com/; https://www.polycatalog.io/project/win-weather-bot | 138-run ensemble, 3σ breaker, pricing |
| S34 | https://www.polycatalog.io/project/polytragent | |
| S35 | https://www.polycatalog.io/project/polymarket-prediction-bot | |
| S36 | https://www.polycatalog.io/project/arbitradepro | |
| S37 | https://www.polycatalog.io/project/polymarket-copy-trading-bot-v1; https://github.com/Trust412/polymarket-copy-trading-bot-v1 (404) | |
| S40 | https://www.stepsecurity.io/blog/malicious-polymarket-bot-hides-in-hijacked-dev-protocol-github-org-and-steals-wallet-keys | 2026-03-15 |
| S41 | https://dev.to/abrownfox001/why-almost-every-polymarket-trading-bot-on-github-is-malicious-28o7 | 2026-05-27 |
| S42 | https://osv.dev/vulnerability/MAL-2026-4212 | 2026-05-20/21 |
| S43 | https://phemex.com/news/article/polycule-tg-bot-hacked-230000-in-user-funds-compromised-52040 | 2026-01-07 |
| S44 | https://skillsllm.com/skill/polymarket-trading-bot | texsellix repo listed as a "skill" |

### Reproduce §2 numbers

1. Profile to wallet:
   - `curl -s https://polymarket.com/@automatedaitradingbot | grep -o 'proxyWallet[^,]*'`, or
   - `gamma-api …/public-profile?address=`.
2. Activity: page `data-api …/activity` with `limit=500`, `offset` 0–3000, then set `end=<min timestamp>` and repeat. Offset alone stops at about 5,000.
3. `closed-positions` pages at `limit=50`.
4. Classify by `eventSlug` keywords: temperature/weather, `cr*`/cricket, `hok-`/esports, and so on.

The scratch scripts are not committed.

---

## Verification (2026-10-07)

Adversarial re-check of 10 claims. Each was re-pulled from the cited endpoint and checked against at least one independent source. All access dates are 2026-10-07.

- **Method:** read-only curl and Firecrawl. The `/activity` history was re-paged backwards by `end=` and de-duplicated on the full row (tx, type, asset, side, size, usdcSize, timestamp, conditionId, outcomeIndex). All 7,775 rows of `/closed-positions` were re-pulled.
- **Result:** 0 refuted, 3 partly, 7 confirmed.

| # | Claim | Verdict | Note | Better source |
|---|---|---|---|---|
| 1 | @automatedAItradingbot → `0xd8f8…0f11`, bio, created 2025-01-31 | **confirmed** | gamma `createdAt` 2025-01-31T04:43Z. Bio matches exactly. The profile page HTML embeds the same wallet 45×. The first activity event is 2025-01-31 05:39 UTC. | https://polymarket.com/@automatedaitradingbot (embedded data) |
| 2 | ALL PnL $88,259.38, rank 2126, vol $3,486,030; MONTH −$3.82; value $10.92 | **confirmed** | Exact: pnl 88259.378891, rank "2126", vol 3486030.47, MONTH −3.82195, `/value` 10.915. The profile page `headlinePnl` is $88,934.83, a different metric. | `/v1/leaderboard` + `/value` (as cited) |
| 3 | From 36,058 activity events: weather $69,125 of $92,678 (75%); longshots +$65,272 on $41,791, 1.54¢, 7.3% YES; median buy $1.60 | **partly** | Every figure reproduces to the dollar, but the PnL figures come from **`/closed-positions` (7,775 rows), not `/activity`**: weather 6,170 pos → $69,124.83 (74.6%), longshots 4,359 → $65,272.12 on $41,790.72, cw entry 1.542¢, 319 winners = 7.32%, median BUY $1.60. "Resolving YES" should read "won": 19 of the 319 are NO-side. "36,058" is an artifact of a dedupe key without `conditionId`, which collapses distinct REDEEM rows. The true distinct count is **41,473 events** (derived). | https://data-api.polymarket.com/closed-positions?user=0xd8f8c13644ea84d62e1ec88c5d1215e436eb0f11&limit=50&offset=N |
| 4 | 92% of London weather buys (4,367/4,723) on observation day; Apr-26 5,086 → May-26 461 fills | **confirmed** | Re-derived: 4,367 same-UTC-day of **4,719** London temperature buys (92.5%), 346 the day before. Apr 5,086–5,088 (by dedupe key), May 461. A denominator difference of 4 is immaterial. | `/activity` (as cited) |
| 5 | `MrFadiAi/Polymarket-bot` and `MrFadiAi` return 404 | **confirmed** | 404 from local curl, from Firecrawl (US-IAD, maxAge 0) and on `/issues/2`. `api.github.com/user/73393700` was **not re-checked** (unauthenticated rate limit 403). The fork page still shows "forked from MrFadiAi/Polymarket-bot", so the upstream may be suspended or hidden rather than deleted. The cause is unknown. | https://github.com/khalefbleydjadja-hub/Polymarket-bot ("forked from" link) |
| 6 | Bot's own audit file: copy PnL always $0 (risk limits never triggered), `canTrade()` Direct-only, `calculatePositionSize()` never called, "Zero real arb opportunities in testing", fixed 2026-09-09..13 | **partly** | The four quoted findings are verbatim (#2, #4, #5, #7). However: (a) the file was written by contributor **Abhiram Kamsali** (`abhiram-k-2223`), not MrFadiAi. It tracks upstream issue #2, which it says is "still OPEN upstream", and the fixes sit on "our branch `feat/data-sources-backtest`", marked "uncommitted" on 2026-09-09. (b) "Risk limits never triggered" is an inference: the file says `recordTrade(0, …)` was replaced so that "streaks/PnL are meaningful". (c) The DipArb $0-profit class was left as an open follow-up. | Commit patches `dfd24e41`, `b4746895` (https://github.com/khalefbleydjadja-hub/Polymarket-bot/commit/dfd24e41c4492c083ebc2823c96ef8bc5b8ca447.patch) |
| 7 | Weather markets: `weather_fees`, rate 0.05, exponent 1, takerOnly, rebate 0.25, orderMinSize 5 | **confirmed** | 16 open temperature events / 166 markets, all `weather_fees` with exactly this schedule. 616 of 617 weather-tag markets have `orderMinSize` 5. Caveat: the `weather` tag also covers 10 fee-free events (earthquake, meteor) and a few `general_fees` / `tech_fees` (rate 0.04) markets. Temperature-market tick sizes are split 78 at 0.01 and 88 at 0.001. | https://docs.polymarket.com/trading/fees (Weather 0.05 / maker 0 / 25% rebate); https://help.polymarket.com/en/articles/13364478-trading-fees |
| 8 | @98euf98a `0xdf4c…4f22`, joined 2026-05-07, ALL $64,401 on $9.0M, MONTH +$10,602 on $1.05M | **confirmed** | ALL 64,401.28 / 9,001,885.77 (rank 2811). MONTH 10,602.09 / 1,051,627.18. gamma `createdAt` 2026-05-07T07:16Z. Value is now $3,860.55 and `/traded` 54,805. The link to PolymTradeBot is **vendor-claimed only**: the listing names "@98euf98a". | https://www.polycatalog.io/project/polymtradebot (linkage); `/v1/leaderboard` (numbers) |
| 9 | Hijacked dev-protocol org: 20+ fake bot repos, typosquats `ts-bign` / `big-nunber`, exfiltrate .env and wallet keys, SSH backdoor | **confirmed** | StepSecurity (2026-03-15): "over 20" repos; payload deps `levex-refa` (stealer) and `lint-builder` (chown `.ssh`, `ufw allow 22/tcp`). Independent: all four packages are npm "security holding" packages with OSV/GHSA malware entries dated 2026-03-20 and 2026-03-25. | https://osv.dev/vulnerability/MAL-2026-2190 (ts-bign), MAL-2026-1934 (big-nunber), MAL-2026-2188 (levex-refa), MAL-2026-1936 (lint-builder) |
| 10 | PolyCatalog PolyBot Pro: $29/$99 per month, "AI-powered predictions", all links to a Telegram referral; polybot.trading charges 1% on successful trades, no AI | **confirmed** | Listing: Free 10 trades/day, Pro $29/mo, Enterprise $99/mo; subtitle "AI-powered predictions"; all 7 links point to `t.me/TradePolyBot?start=ref_5sqbcpu`. polybot.trading FAQ and JSON-LD: "1% flat fee on successful trades". There are 0 hits for AI, LLM or "machine learning" in the page HTML. Same bot (@TradePolyBot). | https://polybot.trading/ (FAQ #4) |

**Additional discrepancies found while re-checking (not in the claim list):**

| Location | Report says | Re-derived (2026-10-07) | Impact |
|---|---|---|---|
| §2.2 event table | REDEEM 1,729 / $157,024; total 36,058 events | **REDEEM 7,124 / $167,765** (6,448 zero-value losing redeems + same-size redeems in batched txs were collapsed); total **41,473**; TRADE 32,205 (24,429 BUY / 7,776 SELL) | Counts only. The PnL conclusions rest on `/closed-positions` and are unaffected. |
| §3.1 "Fork reviewed" | Fork commits are "authored 'Fadi'" | The `docs/AUDIT_FIXES.md` commits (2026-09-09) are authored by Abhiram Kamsali. Fork history also contains merges of upstream PRs ("Merge pull request #5 from MrFadiAi/…"). | Attribution of the audit file (see #6) |
| §1 / TL;DR "account deleted" | Deleted | Only the 404 is verified. The fork's "forked from MrFadiAi/Polymarket-bot" link still renders, which fits suspension or hiding as well as deletion. | Wording: say "unavailable (404)" |
