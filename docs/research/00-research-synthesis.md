# 00 — Research synthesis (tracks R01–R08 → design inputs)

Built 2026-10-08 from `docs/research/01..08-*.md`; design pack `docs/design/00..05` is the frozen baseline. **Legend:** `[Rnn]` = research report nn (suffix = its own section/source id; V# = its Verification row; sNN = source id in R02's own list); `[Pnn]` = design-pack doc; **(D)** = derived by this synthesis; ✗ = refuted (false); ⚠ = partly/unverified (caveat). Each report's Verification section overrides its body. R07/R08 have no Verification section (code reads, single source). Only fresh lookups: `docs.polymarket.com` changelog and V2 migration page, to settle two contradictions (§8).

## 1. Executive summary

1. **No evidence that an LLM forecaster makes money after fees on real markets.** Prediction Arena: Kalshi −16.0% to −30.8% in 57 days; Polymarket −1.1% avg over ~28 days, mostly inactivity [R04 V7, R02 V1]. Olas Polystrat −7% lifetime [R02]. The only documented positive is an LLM *blended with the market* (AIA Brier 0.106 vs 0.1106, p=0.057, LLM weight ~0.33) [R02]. P1 is an experiment.
2. **$50–150 is a measurement budget.** Realistic 3-month outcome −$100…+$75 on $100; 35–50% chance paper shows no edge and we never go live ($0 cost) [R04 §9]. $5k/month needs ≈$25–100k plus proven edge; compounding $150 takes ~30 months at an undocumented 20%/month [R04 §0].
3. **Live PnL cannot prove an edge in 3 months.** 348 resolved forecasts detect a 0.02 Brier gain, 1,392 a 0.01 gain, 2,200–15,500 a realistic frontier-vs-market gain [R02 V10]. Count distinct events, not forecasts (AIA's 1,610 questions = 322 markets × 5 dates).
4. **Min-lot trap (D).** At $50, ¼-Kelly stake < 5-share minimum unless edge ≥ 0.4·a·(1−a) (≥0.10 at mid prices; ≥0.033 at $150). With LLM weight 0.33 that needs ~30 pts of LLM-vs-market disagreement, the zone where a `max_edge` reject is likely to fire. Under pack rule R6 (reject, never widen) almost nothing trades: $50 live is a plumbing canary unless the owner sets a min-lot policy (§7).
5. **Maker-first is the one evidence-backed habit** (winners post limits, losers cross; Akey) [R04 §1]. Fee = C·rate·p(1−p), read per market from Gamma `feeSchedule`; the category table is wrong on live markets; makers pay 0 [R05 F3, R03 C2]. Resting orders need the 10 s heartbeat.
6. **Strategy:** weather = best-evidenced slow strategy but decaying, and the verified earner traded observation-day nowcasts, not T−36h forecasts (unresolved, §6). Bonds only as maker orders. Reward farming measure-only. Arbitrage, crypto, sports, news: never [R03 §5].
7. **LLM capacity ≈95–135 long-context calls/day** (nominal ~800) [R06 §1.2]. Gemini 3.5 Flash-Lite (500 RPD) is the workhorse; 3.8/3.5 Flash get 20 RPD; Groq rejects 15k prompts; GitHub Models/Cerebras free are gone; NVIDIA free is eval-only. ARGUS router needs 11 fixes, first a USD meter and token capture [R07 §3.5].
8. **Forecaster:** K=6 across 2–3 families (10 same-model samples ≈1.4 independent), blind pass, median, clip [0.03,0.97], log-odds blend with LLM prior weight 0.25–0.33, **no LLM crowd-adjust pass**, fixed transform until ~500 resolved [R02]. K=6 ⇒ ~12 markets/day, not 15.
9. **Pack defects:** client order id hashes `attempt_no` (a retry after timeout double-places) [R07 C1]; ARGUS has no calibrator, shadow ledger, USD meter, token log or call ceiling (~40% of "port from ARGUS" is net-new); ARGUS CI ran no tests until 2026-10-05 [R07].
10. **Platform volatility:** ≥9 fee/delay/oracle changes in 2026 [R03 §6]. Protocol V2 (ExchangeV3, signing domain "3", `CONDITIONAL-V2`, new `positionId`; CTF holdings not converted) is rolling out, net-new markets ~Nov 2 [R03 C4 ⚠ dates single-source]. Keep signing/redeem swappable; add a monthly changelog-diff job.

## 2. Earnings reality

**Profitability distribution.** Wallets losing: 68.8% (Akey et al., CEPR DP21615, 588M trades), 84.1% (Sergeenkov, 2.5M addresses), ~70% (DeFi Oasis). Top 1% of winners take 76.5% of profits; ≥$5k four months in a row = 0.015% of addresses; 3.14% "skilled", 44% persist, ~12% of top earners skilled [R04 §1, R03 S17]. Trailing-month ≥$10k ≈ rank 1,396 (top ~0.27% of ~527k active), ≥$5k ≈ rank 2,372 (~0.45%) [R04 §3a V]. Bots: 0.54% of wallets, 63.1% of trades, 28.3% of $; bot class −$2.82M after fees in a 9-day window (Bitquery, vendor heuristic ⚠). All bots earned ~$1.2M in <$10k markets vs $35.1M in >$10M markets (Nov-22→Feb-26) ⇒ long-tail pool ≈$31k/month for everyone (D) [R04 §2]. Leaderboard PnL can differ in sign from `user-pnl-api` for market makers: use `user-pnl-api`/`/v2/user-pnl` `economic_pnl` [R04 §3c].

| Bot / class (survivors only) | Verified | Now |
|---|---|---|
| Crypto Up/Down: 0x8dxd, gabagool22, distinct-baguette, Account88888 | $112k–$880k/month Dec-25→Mar-26 [R04 §4a] | ≈$0 from Apr-26. "$313→$414k" is [C] |
| Weather: ColdMath, VibeTrader, BeefSlayer, HondaCivic, HenryTheAtmoPhD | $8–51k/month Dec-25→Apr-26 | Flat/negative May–Jul; BeefSlayer, HondaCivic partly back Aug–Sep; ColdMath trades sports |
| `automatedAItradingbot` 0xd8f8…0f11 (owner link) | +$88.3k, 18/18 months positive, ~$5.2k/month; <10¢ entries +156% ROI at 7.3% hit rate; median buy $1.60; 92% of London buys on observation day [R01 §2] | Last trade 2026-07-14; $10.92 = open-position value |
| Arbitrage | $39.59M Apr-24→Apr-25, pre-fee [R04 V8] | Today 2 of 279 neg-risk events clear fees, both dust [R03 §3B] |
| LLM bots | Prediction Arena (above); PolyBench 2 of 7 profitable in 1 week; Olas 81.6% of qualifying agents negative; ryanfrigo n=55 edge −1.9 pts; Raven +15.9% on 42 trades (selection alone −4.7%) [R02] | None verified profitable |

**Edge half-life:** crypto ≈4–5 months; weather Dec-25→Apr-26; `automatedAItradingbot` ≈14–15 months [R04 §0]. Planning assumption 3–6 months; auto-pause a category when rolling 30-day edge/Brier <0 [R04 §10]. Cause unproven: [R04, R03] blame Fee Structure V2 (2026-03-30), yet that bot (+$7.8k) and ColdMath (+$44k) earned in April after weather fees began; the drop followed the CLOB V2 cutover 2026-04-28 [R01 §2.2].

| Net monthly return | Capital for $5k/mo | for $10k/mo | Months to grow $100 into $5k-capital |
|---|---|---|---|
| 1% / 2% | $500k / $250k | $1M / $500k | 856 / 395 |
| 5% / 10% | $100k / $50k | $200k / $100k | 142 / 65 |
| 20% / 50% | $25k / $10k | $50k / $20k | 30 / 11 |

Evidence exists at ≥10%/month only in niches that decayed within 4–5 months; durable ≈1–2% (MM-style) needs six to seven figures and low latency; there is no data point for an "LLM long-tail bot at 5–20%" [R04 §8]. Capacity (D, [R04 §7]): 6,813 long-tail markets, median $361/day volume, $4.5k liquidity, 1.8¢ spread; ≤10% of daily volume ≈ $36/day per market; $5k/month at 5% net edge needs ~90 such markets every day.

**3 months on $50–150** [R04 §9, judgment]: gain on $100 at 1/2/5/10/20/50%/month = +$3/+$6/+$16/+$33/+$73/+$238. Paper shows no edge, never live 35–50%; live −20…−100% 20–30%; −20…+10% 15–25%; +10…+50% 5–15%; >+50% ≤5%; on track for $5k/month ≈0%. Olas's best 30-day window (+10%) = $5–15/month at our size [R02 §5c].

## 3. Design-pack deltas

| Topic | Pack says | Research says | Tag | Source |
|---|---|---|---|---|
| Framework | py-sdk + thin engine [P01] | py-sdk covers REST, WS, orders, session keys, redeem; Nautilus 1.230 lacks sig type 3; 2.0.0rc6 pre-release | CONFIRMS | [R05 §2] |
| Paper broker | Fill at recorded ask+depth [P05] | No testnet. Add 1–3 s latency re-snapshot, conservative maker queue rule, shadow-mid PnL; never book at signal time (4f) | ADD | [R05 F1], [R01 §6#3] |
| Execution | Limit only; FOK to take; GTD to rest | Maker-first post-only; taker only if net edge > fee+spread+buffer; marketable GTC/FAK with price cap, not FOK; GTD ≥3 min; heartbeat | CHANGE | [R04 §10], [R05 §9] |
| Fees | Fetch rate at runtime; "bites hardest near 50¢" | Per-market `feeSchedule`; sports 0.03 still on new markets; `/fee-rate` is a flag; skip if exponent≠1 or takerOnly≠true (4c). Per-share statement is right (max 1.25¢ at 0.05); as % of notional = rate×(1−p), so use the per-share fee in probability-point edge | CONFIRMS+ADD | [R05 F3], [R03 C2, §1] |
| P1 universe | Prefer fee-free geopolitics | Geopolitics median event volume $164k, 13% <$10k: liquid. Select by volume/text-heaviness, 3–45 days; exclude <24–48h, liquid sports, short crypto | CHANGE | [R03 §3N], [R02 §5a#10] |
| K samples | K=3 | K≥5–6 across 2–3 families | CHANGE | [R02 s21, s31] |
| Crowd-adjust pass | Optional pass 2 shows price | Drop; blend in code (442/527 ⚠ → 159 of 419 clearly below on recount) | CHANGE | [R02 V6, s36] |
| Calibration | Isotonic or Platt, weekly | Platt; fixed transform until ~500; isotonic ≥~300 (D); clip [0.03,0.97] | CHANGE | [R02 V8] |
| Blend | Linear, w starts 0 | Log-odds, prior LLM weight 0.25–0.33; shadow-log w∈{0, 0.33, fitted} (D) | CHANGE | [R02 §5a#9] |
| Abstention | Spread, TTL | Add `max_edge` reject, IQR gate ≈0.15, edge ≥ max(min_edge, own ECE), `edge_source` tag | ADD | [R02 §5a#10,#12] |
| Sizing | ¼ Kelly at best ask | VWAP book-walking log-utility Kelly net of fee, ¼, caps; min-lot conflict (§1 #4) | CHANGE | [R02 §5a#11], (D) |
| Min size | 5 shares; marketable <$1 rejected | 5 shares in 1,000/1,000 live markets; unit conflict; **$1 floor unverified** | ⚠ ADD | [R05 §4, §10] |
| G2 gate | ≥60 resolved | 60 = smoke test; decide at ≥350, expect inconclusive for realistic gains; recompute from real \|q−m\| | CHANGE | [R02 §6, V10] |
| Metrics | Brier vs mid, PnL | Add edge-vs-book by side/category/horizon; YES-bias check | ADD | [R02 §5a#13] |
| Evidence | Allowlist, no URL following | ≥2 distinct sources; ≤2 rounds (LLM proposes query strings, code runs them); drop items dated after forecast | ADD | [R02 §5a#5], [R06 §4.2] |
| Call budget | 75/day, K=3+1+1 | ~8 calls/market at K=6 ⇒ ~12 markets/day at ~100 calls | CHANGE | [R02 #15], [R06 §1.2] |
| Router reuse | "Reuse that router" | Adapt, fix R1–R11 (4j); hard `distinct_families`; separate keys/projects (ARGUS `eval-models` ≈69 calls/day bypasses counters; Gemini meters per project per model) | CHANGE | [R07 §3.5, C4, C5] |
| USD cap | ARGUS $2 cap design-only | Paid tier unreachable by test; tokens never recorded: build meter first | CHANGE | [R07 C2] |
| **Client order id** | hash(market, side, forecast_id, **attempt_no**) | Drop `attempt_no` (retry after lost reply places 2nd order). V2 orders dropped `feeRateBps`, `nonce`, `taker` (changelog 2026-04-28, read by synthesis); a client-id field is **unverified**: else deterministic salt or journal-before-send + open-order lookup | CHANGE | [R07 C1, §5] |
| Net-new code | "Port" forecaster schema, calibrator, shadow ledger | Not in ARGUS (`TradePlan` carries side/entry/stop; no `calibration.py`/`shadow.py`); scrub truncates 200/600 chars and lacks `id`/`fetched_at`; `llm_calls` lacks `input_hash`/`evidence_ids`, `raw_output` is a repr, no hash chain; no daily ceiling | ADD | [R07 C3, C7, C8, C11, C12] |
| Ops | `pm-exec` service; ARGUS "proven" | ARGUS is timer-based, F-020 died silently: dead-man switch; CI false-green until 2026-10-05, `replay --prompt-version` unbuilt: re-run ported tests; scanner base uses diskcache ("missing"): research path only; test path `tests/data/test_scrub.py`; compare cumulative multipliers in SHRINK<50% (F-054); ARGUS stop/watchdog invariant does not transfer (reconcile + redeem jobs instead) | ADD | [R07 C6, C9, C10, C14–C16] |
| Strategies | P1, P2 lead; P4 "poor"; P3 observe; P5 offline | Add weather W; P4 measure-only; bonds maker-only; add "NO below 65¢" baseline; ARGUS's own cross-venue idea (S7) as candidate | ADD | [R03 §6], [R07 C13] |
| Custody | Session key preferred | Confirmed; owner cannot see session-key orders; redeem not documented for session keys; Builder key + email (4e) | CONFIRMS+ADD | [R05 F6] |
| Risk caps | Per market/event/category | Add neg-risk group worst case, weather per city/day, PnL-drift reconciliation (\|liquid+open−(baseline+PnL)\| > max($5,2%)), stale-signal skip, worst-price caps, dust skip | ADD | [R01 §6], [R03 §3H] |
| Security, drift, ntx | Official SDK only; R-list [P05] | Confirmed: >20 malicious repos (hijacked `dev-protocol` org), npm MAL-2026-4212; hash-lock, no install scripts. Add monthly changelog-diff job, swappable Protocol V2 layer, ntx R36/R13/R24/R28/R29/R47/R68/69 + adapt R44 (4k) | CONFIRMS+ADD | [R01 §5], [R03 §6, C4], [R08 §8] |

## 4. DESIGN INPUTS (exact identifiers)

### 4a. Official SDK [R05 §2, V2]
- PyPI **`polymarket-client`** 0.12.0 (uploaded 2026-10-01; 36 releases since 0.1.0b1 2026-05-21); pin with hash. Import **`polymarket`**: `from polymarket import AsyncPublicClient, AsyncSecureClient`. Python `>=3.11` (classifiers 3.11–3.13; use 3.12). MIT, Beta; repo `Polymarket/py-sdk`. Deps `httpx[http2]`, `websockets<16`, pydantic 2, `eth-account` 0.13+, `eth-abi`, `msgpack`.
- Covers: public reads (Gamma, books, prices, history, Data API v2; typed, `Decimal`); market WS `client.subscribe([MarketSpec(...)])` (async only) and `UserSpec()` (L2 creds); `place_limit_order` GTC/GTD, FAK/FOK, post-only, batch `post_orders`, cancel, heartbeats; Deposit Wallet derive/deploy/approvals (`get_trading_approvals_state`) via Relayer or Builder key; `authorize_session_key`/`fetch_session_keys`/`revoke_session_key` (scope defaults **ALL**: pass `CLOB`); split/merge/redeem (protocol chosen from market `version`); `on_rate_limit_update`.
- **Breakage risk:** 0.x minors may break. 0.10.0 broke positions, trades, price-history, leaderboard, holders (`list_price_history` replaced `get_price_history`); 0.7.1 removed `valid_until` from session keys; 0.8.0 `revoke_session_key` returns `None`. Read the changelog per bump. Fallback `py-clob-client-v2` 1.2.0 (official).
- **Avoid:** `polymarket` 0.1.2 (unofficial; module `Polymarket/` collides on case-insensitive macOS), `polymarket-sdk` 0.2.0 (module `polymarket_autopilot`), `polymarket-apis` 2.3.0, `py-clob-client` 0.34.6 (V1, dead), `polymarket-paper-trader` (agent/MCP tool, not on the key isolated server), any "polymarket bot" package.
- V2 migration page (read 2026-10-08): V2 orders use **ExchangeV3**, signing domain `"3"`, `CONDITIONAL-V2` for balance/allowance; V2 balances in PositionManager; split/merge/redeem via Router.

### 4b. Venue endpoints and fields [R05 §4, §6–8], [R03 C1]
- **Hosts:** `https://gamma-api.polymarket.com`, `https://clob.polymarket.com`, `https://data-api.polymarket.com` (v2 under `/v2`; v1 frozen), `https://bridge.polymarket.com`, geoblock `https://polymarket.com/api/geoblock`. Python's default User-Agent got HTTP 403 from the CLOB [R03]: send an explicit UA.
- **Gamma fields:** `id`, `conditionId`, `questionID`, `slug`, `clobTokenIds`/`outcomes`/`outcomePrices` (JSON *strings*), `version` (`v1` token ids, `v2` `positionId`); `bestBid`, `bestAsk` (can lag), `spread`, `lastTradePrice`, `liquidityNum`, `volume24hr`, `volume1wk`, `oneDayPriceChange`; `feesEnabled`, `feeSchedule{rate,exponent,takerOnly,rebateRate}`, `feeType`; `orderMinSize`, `orderPriceMinTickSize`, `acceptingOrders`, `enableOrderBook`, `restricted` (exclude), `rfqEnabled`; `negRisk`, `negRiskMarketID`, `negRiskOther` (event: `enableNegRisk`, `negRiskAugmented`); `rewardsMinSize`, `rewardsMaxSpread`, `holdingRewardsEnabled`; `endDate`, `startDate`, `resolutionSource` (often `""`: rules in `description`), `umaResolutionStatuses` (plural JSON-string array), `umaBond`, `umaReward`, `customLiveness`. `category` is null: use event `tags[]` (`politics` id 2, `geopolitics` id 100265; slugs `weather`, `esports`, `mention-markets`). Ignore `takerBaseFee`/`makerBaseFee`. Endpoints: `/markets/keyset` (max `limit=100`, `after_cursor`), `/events/keyset?tag_id=`, `/markets/slug/{slug}`, `/tags`, `/public-search?q=`; `GET /markets` defaults `closed=false`.
- **CLOB:** `/book?token_id=` (**bids ascending, asks descending: best = LAST**; has `min_order_size`, `tick_size`, `neg_risk`), `POST /books` (≤500), `/price`, `/midpoint`, `/spread`, `/last-trade-price`, `/tick-size?token_id=`, `/clob-markets/{condition_id}` (keys `mos`, `mts`, `ao`, `fd{r,e,to}`, `v`; meanings inferred), `/time`, `/ok`, `/rewards/markets/current`.
- **Price history:** `GET https://data-api.polymarket.com/v2/prices-history?token_id=` (not `asset_id`) with exactly one of `interval` (`max|all|1m|1w|1d|6h|1h`), `start`+`end`, `as_of`; optional `bucket_seconds`, `limit`≤10,000, `cursor`. Explicit windows cap **15 days**. Retention: 3 h/12 h buckets back to 2022-11-18; 1-min ≥7 d; 5-min ≥60 d; 30-min ≥90 d. Legacy `/prices-history?market=&interval=&fidelity=` still answers.
- **WebSockets:** market `wss://ws-subscriptions-clob.polymarket.com/ws/market` (`{"assets_ids":[…],"type":"market"}`; `PING` every 10 s; `custom_feature_enabled` adds `best_bid_ask`, `new_market`, `market_resolved`; also `tick_size_change`); user `…/ws/user` (L2 creds); sports `wss://sports-api.polymarket.com/ws`.
- **Data API v2:** `/positions`, `/trades`, `/activity`, `/value`, `/user-pnl`, `/user-stats`, `/leaderboard` (`time_period=day|week|month|all`, `sort_by=PNL|VOLUME`), `/resolutions` (one of `question_id`/`condition`/`event_id`; `status`, `was_disputed`, `expected_settlement_time`); envelope `{data, pagination}`. **Bare `volume` = shares; `*_usdc` = USD.**
- **Rate limits** (per IP/10 s; throttled, not rejected): Gamma `/markets` 300, `/events` 500; Data v2 `/v2/trades` 300, `/v2/positions` 200, `/v2/prices-history` 200; CLOB `/book` 1,500, `/prices-history` 1,000, `POST /order` 5,000 burst; Relayer `/submit` 25/min. Per-signer: orders 40/s (burst 60), cancels 80/s, warning mode since 2026-07-24 (`Poly-RateLimit-Warning`, `Retry-After`). **425** engine restart (+2 min post-only), **503** cancel-only; restarts announced on `t.me/polytradingapis`.

### 4c. Fees [R05 §3, V3–4], [R03 §1, C2]
- `fee = C × feeRate × p × (1 − p)`; at match, not in the signed order; rounded 5 dp, min 0.00001; **makers never charged**. Per share at 0.05: 0.24¢ at 5¢, 1.25¢ at 50¢; ÷ notional = rate×(1−p) (4.75% / 2.5% / 0.25% at 5¢/50¢/95¢). **✗ R04's "1.75% at p=0.5" is wrong** (crypto 3.5%).
- Doc table taker rate: crypto 0.07; sports 0.05 (was 0.03 until 2026-07-10); finance, politics, mentions, tech 0.04; economics, culture, weather, other 0.05; geopolitics 0. Maker rebate (share of taker fees, daily pUSD, $1 minimum): crypto 20%, sports 15%, others 25%.
- **Live deviations:** `sports_fees_v2` 0.03; **`sports_fees_nfl_cfb_oct26` 0.03 on markets created 2026-08-28→2026-10-07**; `sports_fees_v3` 0.05; 99–107 of 1,000 markets `feesEnabled=false` (geopolitics); `weather_fees` 0.05 rebate 0.25; all sampled `exponent=1`, `takerOnly=true`.
- **Read at runtime:** Gamma `feesEnabled` + `feeSchedule` (authoritative, changelog 2026-03-31); cross-check CLOB `/clob-markets/{condition_id}` `fd`. Never `/fee-rate` (`base_fee` 1000 for all rates, 0 if free), never `takerBaseFee`, never the category. General form `C·rate·(p(1−p))^e` is derived: assert `e==1` else skip market. Reconcile with fees on fills. Taker-rebate tiers (Bronze $2,000 30-day weighted volume) and liquidity rewards are out of reach at our size.

### 4d. Order constraints [R05 §4, V5]
- **Tick** per market from `/book`/`tick_size_change`: 0.1, 0.01, 0.005, 0.0025 (World Cup), 0.001, 0.0001. Tick 0.01 → price 2 dp, size 2 dp; tick 0.001 → 3/2 dp; amounts 6-decimal ints.
- **Min size 5 shares** (`min_order_size`; 1,000/1,000 and 993/993 sampled). ⚠ Gamma `orderMinSize` is called USDC in one page, shares in another; error `Size ({size}) lower than the minimum: {min}`. ⚠ **$1 marketable minimum** appears only in NautilusTrader docs: would block taker orders on tail buckets (5 shares × <20¢). Test 5 shares at p ≤ 0.19 in micro-live.
- **Types:** GTC; GTD (expiration ≥3 min ahead, expires 1 min early ⇒ lifetime ≥~2 min); FOK; FAK; market orders = FAK/FOK with `expiration="0"`, BUY sized in USD. **Post-only:** `postOnly:true` GTC/GTD only; crossing → `invalid post-only order: order crosses book`. Not supported: reduce-only, stop, OCO, iceberg.
- **Batch:** `POST /orders` max **15**; `DELETE /orders` max 1,000; book/price endpoints max 500.
- **Heartbeat:** opt-in; once armed, no valid heartbeat within **10 s** cancels all open orders (check every 5 s ⇒ 10–15 s; send every 5 s).
- **Delays:** crypto taker delay **150 ms since 2026-09-04**; sports `secondsDelay`; delayed orders can't be cancelled. FAK/FOK return `tradeIDs` (since 2026-07-24). V2 orders have no `feeRateBps`/`nonce`/`taker`; `builderCode` native.

### 4e. Accounts and custody [R05 §5, F6–F7]
- **Signature types:** `EOA=0`, `POLY_PROXY=1`, `GNOSIS_SAFE=2`, **`DEPOSIT_WALLET=3`** (all wallets deployed on/after 2026-05-04). Plain EOA only if allowlisted.
- **Paths:** (A) polymarket.com account → Deposit Wallet + Relayer API key (Settings → API Keys → Relayer API Keys); needs the signer key (email export `reveal.magic.link/polymarket` ⚠ unverified post-May 2026). (B, recommended D) builder profile `polymarket.com/settings?tab=builder` (Unverified tier: starts immediately, 100 relayer txns/day) → Builder API key → SDK derives and deploys a Deposit Wallet for a fresh local signer.
- **Session keys (beta):** Deposit Wallet only; **cannot withdraw**; scopes `CLOB`/`COMBOSRFQ`/`ALL`; fixed **180 days** (revoke to end early); **needs Builder API key + email builder@polymarket.com** (do at Phase 0). Sees only its own orders: **owner cannot fetch session-key orders**: reconcile with session creds or public `/v2/positions?user=`.
- **Redemption caveat:** not documented for session keys; documented path is an owner-signed Deposit Wallet batch via Relayer. `AutoRedeemer` `0xa1200000d0002264C9a1698e001292D00E1b00af` exists; auto-redeem for every wallet unverified: test with a ~$1 position; plan B = float-only owner key on isolated server.
- **KYC:** none on polymarket.com international (VP Eng, 2026-05-27); applies to Perps beta, Polymarket US, co-location.

### 4f. Paper trading [R05 §1, F1], [R01 §3.2, §6#3]
- **No testnet:** `clob-staging.polymarket.com` has no DNS record; py-sdk has only `PRODUCTION` (chain 137); Polymarket's own SDK tests use real funds. Paper = own simulator on live books, then micro-live with real pUSD (≥10 pUSD suggested).
- **Taker:** snapshot `/book` at signal; re-snapshot after 1–3 s; walk levels; fee per level from `feeSchedule`; enforce tick, 5-share min, post-only crossing rejection, GTD ≥3 min; partial fills.
- **Maker (conservative):** fill only when a public trade prints *through* our price, or at it after the queue ahead is consumed; mark each fill vs mid +1 h for adverse selection [R03 §5#4]. Journal two PnLs: simulated fill and mid "shadow". Model UMA resolution ($750 bond, 2 h challenge, disputes 4–6 days, 50/50 possible) via `/v2/resolutions`.
- **References:** `agent-next/polymarket-paper-trader` (MIT; book-walk, feeSchedule fees, no queue model so makers optimistic; read only); `evan-kolberg/prediction-market-backtesting` (queue heuristic, 75 ms latency; needs L2 archives); `warproxxx/poly-maker --paper` (MIT); NautilusTrader `SandboxExecutionClient`. **Anti-pattern:** MrFadiAi dry-run booked PnL at detection. Free history: `Jon-Becker/prediction-market-analysis` (MIT, 36 GiB), `warproxxx/poly_data` (GPL-3.0); no free L2 book history: record our own.

### 4g. LLM router [R06 §1, §4], [R07 §3.1]
| Provider / model IDs (2026-10-07) | Free limit | Use / caveat |
|---|---|---|
| Gemini (separate AI Studio project): `gemini-3.5-flash-lite` | 15 RPM, 500 RPD; 1,048,576 ctx; **resets midnight Pacific** | Workhorse. Free content used to improve products; grounding not free on 3.x; linking billing converts the project |
| `gemini-3.8-flash`, `gemini-3.5-flash` | **10 RPM, 20 RPD** (ARGUS F-088) | 1 high-thinking sample/market; ECE 0.042 ⇒ recalibrate |
| Mistral free: `mistral-large-latest`†, `open-mistral-nemo` | 6 RPM, 200 RPD (ARGUS F-096) | Second family. ⚠ ARGUS: small/medium always 429; R06 chain names `mistral-small-latest`: measure. Trains on data |
| OpenRouter: `nvidia/nemotron-3-ultra-550b-a55b:free` (1M ctx), `nvidia/nemotron-3-super-120b-a12b:free`, `google/gemma-4-31b-it:free` | 20 RPM; **50 RPD if <$10 ever bought, 1,000 RPD from ≥$9**; UTC day | Roster rotates (ARGUS: 6 of 16 returned valid JSON). Account eligibility and limits vary; verify the per-key and per-account counters. |
| Cloudflare `@cf/openai/gpt-oss-120b` | 10,000 neurons/day, 00:00 UTC ⇒ ≈17 calls/day (D) | Overflow family |
| Groq `openai/gpt-oss-120b`, `openai/gpt-oss-20b`, `qwen/qwen3.8-27b` | 30 RPM, 1K RPD, **8K TPM**, 200K TPD | **15k prompts rejected (413).** Parse/JSON-repair ≤6.5k tokens, `max_tokens` 800 |
| NVIDIA NIM trial `moonshotai/kimi-k3`, `z-ai/glm-5.3`, `deepseek-ai/deepseek-v4.1-flash` | 40 RPM | **ToS: "internal testing and evaluation… not in production"**: paper research only |
| DeepSeek paid `deepseek-flash` (V4.1-Flash, 1M ctx) | $0.30 in / $0.006 hit / $1.20 out per 1M (peak 01–04, 06–10 UTC Mon–Fri; off-peak half) | ≈$0.006/call; $1.5–3/month at 10–20% fallback |
| **Dead/avoid** | GitHub Models (retired 2026-07-30); Cerebras ($5 credit, card, 30 days); Together ($5 min); Fireworks; SambaNova (docs conflict); Cohere trial (eval-only) | — |

- **Role chains [R06 §4.1; † = measure on our key]:** `parse_rules` (≤6k tokens) Groq `openai/gpt-oss-120b` → `gemini-3.5-flash-lite` → Mistral small; `forecast_sample_A` `gemini-3.8-flash` → `gemini-3.7-flash`† → `gemini-3.5-flash-lite`; `_B` `gemini-3.5-flash-lite`; `_C` (must be non-google) `mistral-large-latest`† → `nvidia/nemotron-3-ultra-550b-a55b:free` → `@cf/openai/gpt-oss-120b` → `deepseek-flash` (paid, cap); `research_ensemble` (paper only, logged never traded) NVIDIA trio; `json_repair` Groq `openai/gpt-oss-20b` → Flash-Lite. **Reset clocks:** Gemini midnight Pacific; OpenRouter and Cloudflare 00:00 UTC; ARGUS counters assume UTC for all (R5). Seen failing: Groq TPM, 10 of 16 OpenRouter free models in ARGUS's test, Cerebras/GitHub Models free tiers.
- **Capacity/day:** Flash-Lite 40–60, 3.8 Flash 15, Mistral 20–30, OpenRouter 10–30, Cloudflare 10, Groq 20 short ⇒ ≈95–135 vs nominal ≈800. K=6 allocation (D): 3 google (1× 3.8 Flash + 2× Flash-Lite) + 3 non-google (Mistral, OpenRouter nemotron, Cloudflare gpt-oss) × ~12 markets = 72, + ~12 parse + ~12 query-gen + repairs ≈100–110.
- **Rules in code:** skip a provider at 90% of RPD/TPD; breaker after 3 consecutive 429/5xx; no reachable non-google family ⇒ `NO_FORECAST`; per-family calibration; spend counter before each call; on cap stop forecasting, keep managing, alert once. Model IDs in YAML; daily `doctor --models`; re-measure all limits in week 1. Cost: $0 in paper, ≤$5/month at go-live.
- **Skill [R06 §2]:** ForecastBench market questions: copy-market 71.4; best LLM 62.8 (93 questions ⚠); `kimi-k3` 62.6; `gemini-3.5-flash` 57.4; `gemini-3.1-flash-lite` 53.1; gpt-oss/Qwen3.8/3.5 Flash-Lite unranked. Prophet Arena ECE: Kimi K3 0.012, GLM-5.3 0.015, Gemini 3.8 Flash 0.042; margins over Kalshi ≤0.003.

### 4h. Evidence stack [R06 §3, §4.2]
Order: (0) Gamma rules/`resolutionSource` + CLOB mid (LLM never sees price in pass 1); (1) discovery: Google News RSS `news.google.com/rss/search?q=…+when:7d` (**"personal, non-commercial use"**, keep replaceable) and GDELT DOC 2.0 `api.gdeltproject.org/api/v2/doc/doc` (≤250 records, **1 req/5 s**); (2) search: Exa ($10 credit/month ≈2,500, no card) → Tavily (1,000 credits/month) → Brave ($5/month, **card required**, storage-rights clause), Serper 2,500 one-time; (3) read: Jina Reader (20 RPM keyless, 500 keyed, 10M tokens per new key), Gemini URL-context (free); trim ≤12k tokens in `<evidence id= source= fetched_at=>`; (4) structured: FRED (120/min), BLS v2 (500/day), NOAA CDO (5/s, 10k/day), NWS `api.weather.gov` (UA, no key), Wikipedia (200/min with contact UA). Skip AskNews (non-commercial), Reddit (approval), Gemini grounding (paid only; 5,000 free/month then $14/1k; no cache/"analyze"). Polymarket comments = injection vector, never evidence. Daily: ≤15 markets, ≤45 searches, ≤75 page reads; cache by (query, day).

### 4i. Forecaster best practices [R02 §5a, V], [R06 §2.3]
- **Blind pass**; XML prompt (`<question>`, `<resolution_criteria>` verbatim, `<today>`, `<close_time>`, `<evidence id= date= source=>`); steps: what resolves YES, time left, status quo, base rate, YES/NO paths, evidence for/against, "already resolved/ambiguous?"; strict JSON; code computes dates/fees/odds.
- **Samples:** K=6 = 2–3 × 2–3 families; gain sharp 1→5, modest to 15; same-model ρ≈0.7, different models 0.40; **higher reasoning effort** beat standard 8/8; teams 2–10 ≈ equal. **Aggregate** by median (≈ mean of logits); no personas, no LLM-merging.
- **Retrieval:** ≤2 rounds, ≥2 distinct sources (winners 1.72–1.75 vs 1.00), drop items dated after forecast time.
- **Calibration:** Platt on logit(p) from our forward record; Metaculus gain (Brier −0.016) measured at ≥500 forecasts/bot with a chronological 70/30 split; until then identity or AIA α=√3 on an averaged ensemble; single shots overconfident (27% error above 90%), averaged ensembles underconfident: take direction from data; per-family maps; clip raw p [0.03,0.97].
- **Blend:** `logit q = w·logit p_cal + (1−w)·logit m`; prior w=0.25–0.33 liquid (AIA 0.33 [0.12,0.47]; Halawi 0.2), more for illiquid (AIA 0.87 on FB-Market); pre-register one live candidate.
- **Abstain:** verdict≠FORECAST; IQR>0.15 (proposal); edge < max(min_edge, own ECE) (Olas 0.03, pack 0.05); **edge > max_edge**; <24–48h to resolution; short crypto/sports. Tag `edge_source ∈ {llm_disagreement, calibration_shift, blend_artifact}`.
- **Leakage:** forward-only; frozen evidence hashes + model ids; eval markets must resolve after every model's training cutoff [R08 R37].
- **Sample size:** `n ≈ ((1.645+0.842)·mean|b−p| / α*)²` = ((2.487)·0.15/0.02)² ≈ **348** (Var αᵢ=4q(1−q)(b−p)²; one-sided 5%, 80% power); α*=0.01 → 1,392; 0.005 → 5,567; frontier-vs-market α*≈0.003–0.008 → 2,200–15,500; scales with (|b−p|/α*)²: recompute from real |q−m| [R02 V10]. Return-based (mean >0, one-sided 95%) [R04 §9]: edge 3/5/10% per $ at p=0.3 → 7,133/2,595/665; p=0.5 → 3,004/1,080/268; p=0.8 → 681/227/45. Rule of thumb (D): n ≈ 2.71·((1−p)/p)/e² (p=0.5, e=5% → 1,082; conservative at 0.8). $100 resolves only a few hundred bets in 3 months.

### 4j. ARGUS port and router gaps [R07 §3.5, §5]
| ARGUS module | Action | Note |
|---|---|---|
| `llm/router.py`, `config/models.yaml` | adapt | Fix R1–R11 below; add `usd_per_mtok_in/out`, `reset_tz`; `sample_k(agent,k,distinct_families=True)` |
| `llm/schemas.py` | adapt | Keep `Strict` (`extra="forbid"`) + one discriminated shape; new `Forecast`: `verdict`, `p_yes ∈ [0.01,0.99]`, `evidence_ids`, `resolution_reading`, `ambiguity_notes` |
| `llm/prompts.py` | as-is | `version:` header; missing placeholder raises; F-093 brace test (prompts are new) |
| `data/scrub.py` | adapt | Add `id=`, `fetched_at=`; hash full rules text; neutralise URLs; `catalyst.py filter_sourced()` = grounding pattern |
| `journal/ledger.py`, `db.py`+migrations, `runs/lock.py`, `exchange/breakers.py`, `tests/conftest.py` | as-is | Insert-only SHA-256 chain; WAL+`synchronous=FULL`; lock for process lifetime |
| `journal/states.py`, `replay.py`, `config.py`, `doctor.py`, `observability/*`, `deploy/*` | adapt | CLOB order states; build prompt-version replay; `MODE=live`+`LIVE_TRADING=I_UNDERSTAND`; doctor + geoblock, V2/pUSD/sig-type, fee-rate; secret pattern + 0x-64-hex, `POLY_*` headers |
| `execution/idempotency.py`, `runs/pipeline.py vet()`, `llm/golden.py`, `gate/killswitch.py`, `protections.py`, `place.py`/`resting.py` | pattern | Never a counter/clock in ids; forecaster + one "how could it resolve the other way" check, fail closed, factor only shrinks; behaviour suites with agree cases; KILL cancels only; journal-before-send, GTD sweep |
| `data/fetchers/polymarket.py`, `data/http.py`/`cache.py` | adapt (research only) | Add pagination, `clobTokenIds`, `negRisk`, rules, fees, depth; never on the execution path (cached, "missing") |
| `gate/rules.py`, `sizing.py`, `fees.py`, `exchange/okx.py`, `runs/watchdog.py`, `llm/stub.py`, `features/*`, Telegram/UI | skip | Perp-specific; stub caused F-117/F-145 |

- **Router gaps:** R1 no USD meter · R2 tokens never recorded · R3 `raw_output` is a repr (add `input_hash`, `repair_used`, `evidence_ids`) · R4 counters per provider but limits per model · R5 resets at 00:00 UTC though Gemini resets midnight Pacific · R6 429 cooldowns process-local (persist in SQLite) · R7 `eval-models` bypasses counters · R8 no global daily ceiling · R9 family split degrades silently (`require_distinct_families`) · R10 timeouts trip the breaker · R11 model ids rot monthly. Keep RESERVE 0.1, COMFORTABLE 0.4, 429 cooldown 90 s (6 h if daily quota spent), one repair, `max_attempts` 4, `max_tokens` always sent. Encode F-001, F-024, F-019, F-119, F-139 (end-to-end forecast→gate→payload test), F-145, F-143/144, F-071/073 (missing venue data raises, never 0).

### 4k. ntx rules [R08], [P05]
- **Port as listed in [P05]** with these adaptations: drop R14's 2026-09-24 live-deploy pre-authorisation; R56 = `ARMED` file + unit pins mode; R64 asserts wallet, pUSD balance, approvals, session-key expiry, per-market fee/tick/min-size/`neg_risk`, geoblock; R74 rewritten (live danger is a phantom cache causing a re-buy); R49 = "resolved and redeemed"; R4 suffixes `*_prob`, `*_usd`, `*_shares`, `*_equity_pct`; R22 = CLOB book, not Gamma `outcomePrices`; R43 = mutmut on fee/edge/sizing/gates/calibration; R48/R58 = one gate function for all venue/chain writes; R76 = read SDK defaults against our cadence. Adapt R2 (documented override), R15 (exposure-vs-cap alert), R25/R26 (never use UA changes to evade the geoblock).
- **Recommended additions [R08 §8]:** **R36** (risk metrics from mark-to-market curve; pack cites it but omits it), **R13** (update status + failure log before ending a session), **R24** (a new strategy is a signal, not a system), R28 (named SQL constraints), R29 (never delete journal/reports), R47 (params in one registry), R68/R69, adapt **R44** (paper cannot validate signing, partial fills, redemption: rehearse with minimum-size real orders). R38/R39 never existed. Doc habits: `docs/04-status.md`, `05-failure-log.md`, `docs/12-adr/`, ntfy `26-phone-alerts.md`, `27-open-rehearsals.md`.

## 5. Features and practices: v1 (ranked) vs defer

**v1:** (1) supply-chain/key hygiene [R01 §5]; (2) one pre-execution choke point with limit tests on real PnL accounting [R01 §6#2]; (3) honest paper broker, dual PnL; (4) fee-aware net-edge gate + maker-first; (5) forward calibration ledger (frozen forecast, model ids, prompt/evidence hashes, executable ask/bid/depth; controls: market mid, mid+noise, "NO below 65¢") [R02, R03 §6#5]; (6) router fixes R1–R11 + separate keys; (7) reconciliation, PnL-drift and staleness halts; (8) heartbeat dead-man for resting orders; (9) correlation caps (neg-risk worst case, weather per city/day); (10) uncertainty breaker (IQR; weather spread >3σ); (11) raw market/order event journal; (12) stale-signal, worst-price, dust skips; (13) strategy interface + observers (weather logger, neg-risk scanner, reward recorder); (14) monthly changelog-diff; (15) injection tests + evidence grounding.
**Defer:** NautilusTrader; LLM implication hedges [R01 #14]; skilled-wallet signal; live reward farming; cross-venue (>$2k); supervisor re-search, isotonic/Beta-Bernoulli calibrator (N≥300); fine-tuning; Metaculus AIB; UI; lessons loop. **Never:** Martingale, win-streak boosts, custodial execution layers, Telegram copy-bots, retail YES+NO arbitrage, personas, LLM-chosen size [R01 #15, R02 §5b].

## 6. Strategy backlog

| # | Strategy | Fit | Evidence | LLM | Effort | Cheapest first experiment | Kill signal |
|---|---|---|---|---|---|---|---|
| 1 | **W Weather ensembles, maker-only** | 4 | gopfan2 +$317k, ColdMath +$134k, automatedAItradingbot +$88k; all flat since May; top-50 weather $412–415k/30 d [R03] | Assist (rules→spec) | M | 4-week forecast-only log: P(bucket) at T−36h/T−12h vs mid, Brier vs mid, would-post-only-bid-fill | Brier not better than mid; maker-fill paper PnL ≤0; rolling 30-d edge <0 (D) |
| 2 | **E/P2 Rules-lawyer + bonds, maker-only** | 4 | ≥90¢ pooled +0.83¢/$ pre-fee; taker −0.46%, maker +0.64–0.82%, fee-flagged +0.33% (CI incl. 0) [R03 V9] | Core | S–M | Offline Gamma study T−24h/6h/1h (IS <2026-01-01, OOS 2026); forward log of `RESOLVED_IN_SUBSTANCE` with quoted clauses | Hit-rate <98% [P03]; OOS maker return ≤0 after fees |
| 3 | **P1+N LLM long-tail** (thin, non-English, text-heavy, 3–45 d) | 3 | AIA blend p=0.057; no real-money proof [R02] | Core | M–L | Phase F forward log vs 2 controls | G2 criteria at N≥350 [P03] |
| 4 | **D-ii Reward farming** | 3→4 if measured | Unvalidated crude estimate (below) | Assist | M | 2-week read-only minute-level book recorder on ~50 reward markets; simulate score share, +1 h marks; then $20 on 2 markets | Rewards < adverse-selection marks (D) |
| 5 | **L Mention markets** | 3 | bama124 +$293k, 22% of share volume (inactive since Feb-25); 21 events/336 markets, median ~$4.5k [R03] | Core | M | Transcript corpus for 3 speakers; 4 weeks forward base rates | No Brier gain vs market (D) |
| 6 | **G Skilled-wallet flow feature** | 2–3 | 3.14% skilled, 44% persist | No | M | Sign-randomisation test on ~500 wallets; net flow vs 24 h move | No predictive power |
| 7 | **B Neg-risk scanner (observe)** | 2 | $29.0M pre-fee 2024–25; 2 of 279 clear fees today | No | S | Hourly depth-aware log | No ≥$5, ≥2% net gap resolving ≤30 d in 4 weeks |
| 8 | **F UMA disputes** | 2 | Dispute rate 0.7–1.2% [U]; proposers whitelisted | Core | S | Log disputes: predicted vs actual ruling | No better than chance (D); cap 5% bankroll |
| 9 | "NO below 65¢" prior | 2 | 73.3% resolve NO [U] | No | S | Offline control P1 must beat | control |
| 10 | L1 LLM implication hedges | n/s | PolyClaw/Alphapoly [R01] | Core | M | Deferred | — |
| 11 | C cross-venue, D-i full MM, J sports/esports in-play, I crypto 5/15-min, K news-speed, A intra-market arb | 1 | Fees, TWAP, 150 ms delay, insiders | — | L | **Never** (C, D-i revisit >$1–2k) | — |
| 12 | M holding rewards (3.25% APR, 17 events) | n/a | ≈$3/yr on $100 | No | S | Tie-break in P1 sizing | — |

**Weather** [R03 §3H, R01 §2–4, R04 §4b]: *Universe:* 367 active events (146 daily highs, 128 lows), ~51 cities, ~11 buckets/event, all neg-risk; median event volume $1.5k (p90 $41k); median market $335/day, $2.9k liquidity; tick 0.01 (78) or 0.001 (88) in 166 sampled markets. *Resolution:* NYC = NOAA LaGuardia hourly max (`weather.gov/wrh/timeseries?site=klga`); Hong Kong = HKO Daily Extract "Absolute Daily Max" (°C to 0.1); Denver = NOAA KBKF with Weather Underground fallback; units/rounding/"no data → lowest bracket" differ per market: parse to a typed spec (the LLM role). *Data:* Open-Meteo ensemble API (ECMWF IFS 51 members, GFS 31, ICON 40, MOGREPS 18; non-commercial terms ⚠), NOAA GEFS/NOMADS (public domain), HKO, METAR, `api.weather.gov`; correct station bias; WinWeatherBot method: 138 NWP runs, accuracy weights, caps per city/region/direction, halt above 3σ spread [R01 S33]. *Fee at low prices:* `weather_fees` 0.05, takerOnly, rebate 0.25: per share 0.05¢ at 1¢, 0.24¢ at 5¢; ≈4.95%/4.75% of notional; reference bot's median entry YES 1.1¢, NO 3¢, sold at 9.1¢ into repricing; 5 shares at 1–5¢ cost $0.05–0.25 (taker blocked if a $1 floor exists): rest post-only GTD bids. *Horizon tension (unresolved):* the verified earner bought on the observation day (92% of London buys, 07:00–15:00 UTC) [R01]; [R03] plans T−48h→T−12h to avoid the nowcast race, so the evidence does not transfer: log both horizons. Cap per city/day.
**Liquidity rewards** [R03 §1, V3; R04 §4c; R05 §3]: $162.0–162.5k/day configured over 18,301–18,477 markets (≈$4.9M/month); median $3/day, p90 $17–18, 9 markets >$1k/day, 208 ≥$100/day; `min_size` 20 shares in 79%, max spread 4.5¢ in 79% (top-volume Gamma samples show `rewardsMinSize` 50–200). Score `((v−s)/v)²·b`; single-sided ÷3 inside mid 0.10–0.90, nothing outside; sampled each minute; $1 minimum payout. Crude 47-book sample: $0.92/$1.33/$0.93/$1.02 per day per two-sided min-size quote in markets paying $1–5/$5–20/$20–100/$100+ (≈1–5% of capital/day, **unvalidated**, ignores adverse selection).

## 7. Owner decisions still open

1. **Min-lot policy:** allow a 5-share order when ¼-Kelly stake is below it (override of ntx R6); per-market cap 5% or 10%; or accept that $50 live mostly rejects.
2. **Go-live evidence bar:** G2 at N≥350 (underpowered), or a weaker "not worse than market, paper PnL >0" bar for a loss-capped canary.
3. **K vs markets/day:** K=6 at ~12/day within ~100 calls, or K=4 at 15.
4. **LLM spend at go-live:** strict $0 or ≤$5/month; one-time $10 OpenRouter top-up on the new account; inference counted in PnL?
5. **NVIDIA trial models** for paper research only: yes/no.
6. **Terms-risk sources:** Google News RSS, Brave card-on-file, Open-Meteo non-commercial terms (or NOAA GEFS only).
7. **Metaculus AI Benchmark** entry: yes/no.
8. **Weather:** forecast-only logger in Phase 2 at both horizons? Kill rule?
9. **Reward farming:** recorder only, no 1-minute quoting loop?
10. **Languages/regions** you can check by hand for the N sub-universe; exclude insider-adjacent single-decider markets?
11. **Custody (D3):** email builder@polymarket.com now, or go straight to plan B (programmatic Deposit Wallet, float-only owner key)?
13. **Crash behaviour:** accept all resting orders cancelled 10–15 s after process death; KILL stays cancel-only (D8)?
14. **Protocol V2 timing:** go live before/after ~Nov 2; v1 markets only until the v2 path is rehearsed?
15. **Auto-pause thresholds:** category rolling-30-day edge floor; confirm [P04 §2] defaults (¼ Kelly, 10%/market, 30% drawdown KILL).

## 8. Unverified, refuted and conflicting claims (must not drive decisions)

| Claim | Status | Source |
|---|---|---|
| Fee "$1.75 per $100 at p=0.5" and R04 §5/§9 "1.75%" | ✗ refuted (docs' $1.75 = 100 shares at $0.50); real 3.5% crypto | [R04 V10] |
| "$313→$414k", "$63→$131k", "$1k→$24k London bot", "$6k/month blueprint", "78%/80–90% accuracy", "73% of arb by sub-100ms bots" | [C]/[U] marketing | [R04 §4], [R01], [R03] |
| 0x8dxd "$2.05M on $200.6M" | ⚠ leaderboard `position_pnl`; `economic_pnl` $2.17M; pnl-api $2.38M; April +$0.2k [R04] vs +$25k [R03] | [R04 V5], [R03 C3] |
| Every PnL÷"volume" margin and "$3.49M volume" (R04 §3b/§4, R01 §2.1, R03 §2) | ⚠ volume is shares; margins understated ~2×; `automatedAItradingbot` $88k ÷ $365k cash ≈ 24% (not 2.5%) | [R03 C1], [R01 §2.2] |
| `automatedAItradingbot` median trade $1.79; "holds $10.92"; "last activity 2026-09-28" | ⚠ median $1.60–2.24; $10.92 is open-position value; 09-28 was a $1.09 REDEEM; last trade 2026-07-14 | [R04 V6], [R01] |
| Prediction Arena Polymarket −1.1% ≈ break-even | ⚠ 28 days, low activity; per-model range −2.68%…0.00% [R04] vs −1.85%…−0.09% [R02] unresolved | [R02 V1], [R04 V7] |
| "442/527 FB entries below copying price"; 71.3 baseline | ⚠ reference is a fresher price; recount 159 below / 60 above of 419; baseline copies t−1 price (freeze copier ≈70.1) | [R02 V5–6] |
| AIA beats market; Raven shows selection works; ryanfrigo −1.9 pts; Olas "60% positive/+69%" | ⚠ p=0.057, superforecasters beat AIA on FB-Market; selection alone −4.7%; Brier on n=43; subsets/staking subsidy | [R02 V2–4, V9] |
| Teams of 4–10 best; "reasoning beats non-reasoning" | ⚠ teams 2–10 equal; it is higher reasoning *effort* | [R02 V7] |
| Groq 413 above 8K TPM; Gemini 3.8 Flash 20 RPD; DeepSeek V4-Pro prices | ⚠ user-observed (docs say 429); forum + ARGUS 429 body; possibly routed to Flash since 2026-09-14 | [R06 V1, S3, V10] |
| Mistral free `mistral-small/large-latest`; OpenRouter per-key vs per-account RPD | unverified; ARGUS saw small/medium 429 | [R06 §4.1], [R07 §3.1] |
| `orderMinSize` unit; **$1 marketable minimum**; V2 client-id field; session-key/AutoRedeemer redeem; email-key export after May 2026 | unverified | [R05 §4, §5, §10], [R07 §5] |
| Protocol V2 dates (canary to Oct 30; net-new Nov 2) | ⚠ single source (The Block 2026-10-06 via [R03 C4]); docs confirm mechanics, not dates | [R03 C4] |

**Contradictions between reports.** *Resolved:* crypto taker delay is **150 ms since 2026-09-04** (changelog: 250→50 ms on 08-17, 50→150 ms on 09-04; [R05]'s "50 ms" was stale); fee percentage (above); K=3 vs K=6 (K=6 fits at ~12 markets/day); share-vs-USD volume. *Unresolved:* weather horizon (nowcast [R01] vs T−36h [R03]); decay cause; Prediction Arena range; 0x8dxd April PnL by source; reward-farming verdict ([R03] "measure", [R05] "out of reach"); Mistral availability; OpenRouter counter scope; `orderMinSize` unit and the $1 floor.
