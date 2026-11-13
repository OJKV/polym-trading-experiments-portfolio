# 05 — Polymarket venue technical facts (verified 2026-10-07)

Scope: the venue facts the implementation depends on: test environment, SDK, fees, order rules, accounts, geoblock and data endpoints.
Method: I read docs.polymarket.com page by page (raw `.md`, via the `llms.txt` index), the official GitHub repos and PyPI JSON. I also made live calls to public GET endpoints. I logged in nowhere, placed no orders and installed or ran no third-party code.
**All access dates are 2026-10-07 unless stated otherwise.**

Labels:
- **[doc]** comes from docs.polymarket.com.
- **[live]** I observed it in a live API response today.
- **[gh]** / **[pypi]** come from GitHub or PyPI metadata.
- **derived** is my own inference or calculation.
- **unverified** is a third-party claim or something not documented.

---

## 0. Findings that affect the design

| # | Fact | Evidence | Design consequence (derived) |
|---|---|---|---|
| F1 | **There is no public testnet, sandbox or paper-trading mode for CLOB V2.** OpenAPI specs still list `https://clob-staging.polymarket.com`, but the host does not resolve in DNS. The official py-sdk ships only a `PRODUCTION` environment (chain 137). | [doc] [live DNS] [gh] | "Paper" must be our own simulator running on **live** order books. After that comes a micro-live smoke test with real pUSD (the docs' quickstart recommends ≥10 pUSD). |
| F2 | The official SDK is **`polymarket-client` 0.12.0** (2026-10-01): MIT, Python ≥3.11, Beta. Its import name is `polymarket`. 0.x minor versions may break things. | [pypi] [gh] | Pin `polymarket-client==0.12.0` with a hash, and read the changelog before every bump. |
| F3 | Fees are **per market**: `feesEnabled` plus `feeSchedule{rate, exponent, takerOnly, rebateRate}` on Gamma. The doc's category table does **not** match every live market: 19 of 1,000 open markets still carry sports rate 0.03, while the doc says 0.05. A market tagged both `politics` and `geopolitics` was fee-free. | [doc] [live] | Read the fee from `feeSchedule` at decision time. Never infer it from the category, the tag or `/fee-rate`. |
| F4 | The minimum order is **5 shares** in all 1,000 sampled open markets. Ticks are 0.01 or 0.001 (0.0025 for World Cup markets only). | [live] [doc] | With a $50 bankroll and a 5% per-trade cap ($2.50), a 5-share minimum makes **only prices ≤ $0.50 tradable**. With a 10% cap ($5), all prices are tradable. (derived) |
| F6 | **Session keys need a Builder API key, plus an email to builder@polymarket.com** during rollout. They are beta, last a fixed 180 days, cannot withdraw, and the docs never say they can redeem. | [doc] | Custody plan B (a hot float wallet) must be ready from day 1. Test redemption separately. |
| F7 | **polymarket.com has no KYC** for the international product (Polymarket VP Eng, 2026-05-28). KYC applies to the Perps beta, to Polymarket US and to co-location. | press quote, unverified beyond that | No KYC blocker for the owner today. Monitor for changes. |
| F8 | **CLOB rate limits are not a constraint** at LLM cadence. Standard tier: 40 orders/s per signer. Gamma `/markets` allows 300 requests per 10 s per IP. | [doc] | A simple token bucket in the scanner is enough. |

---

## 1. Test environment / paper trading

### 1.1 Official options

| Candidate | Status 2026-10-07 | Evidence |
|---|---|---|
| `clob-staging.polymarket.com` | Listed under `servers:` in every CLOB OpenAPI page. **DNS: no address records** were returned; a later check reported NOERROR/NODATA. Not a confirmed public endpoint. | [docs api-reference/market-data/get-fee-rate](https://docs.polymarket.com/api-reference/market-data/get-fee-rate.md); `dig`/`curl` today |
| `data-api-rs.stage.pmd.use1.polymarket.sh` | Listed as "Staging" in the Data API v2 spec. Internal hostname. | [docs get-a-tokens-price-history](https://docs.polymarket.com/api-reference/markets/get-a-tokens-price-history.md) |
| `clob-v2.polymarket.com` | Was the pre-cutover V2 test host. The docs say that "as of April 28, V2 runs on clob.polymarket.com". Does not resolve today. | [changelog Apr 17 2026](https://docs.polymarket.com/changelog/predictions.md) |
| Polygon Amoy (chain 80002) | `py-clob-client-v2` still carries contract config for chain 80002, and its README says "`chain_id = 137  # or 80002 for Amoy testnet`" with host `<polymarket-clob-host>`. **No public Amoy CLOB host is documented.** The unified `py-sdk` has no Amoy environment at all. Third-party pages (Cobo) describe Amoy recipes; **unverified** that they work with V2. | [gh py-clob-client-v2 config.py](https://github.com/Polymarket/py-clob-client-v2/blob/main/py_clob_client_v2/config.py), [gh py-sdk environments.py](https://github.com/Polymarket/py-sdk/blob/main/src/polymarket/environments.py) |
| py-sdk integration tests | `.env.example` uses `POLYMARKET_RUN_METERED_TESTS=0` with "disposable test credentials". Polymarket's own SDK tests run against **production with real funds**. | [gh .env.example](https://github.com/Polymarket/py-sdk/blob/main/.env.example) |
| In-app demo / practice mode | None found. Only third-party Chrome extensions exist (PredictMirror, DemoMarket); these are **unverified** closed tools and irrelevant for a bot. | web search |

### 1.2 Open-source paper-trading simulators and backtesters (read on GitHub only, nothing run)

| Project | Licence | ★ / last push [gh] | Fill model | V2-aware | Verdict |
|---|---|---|---|---|---|
| [agent-next/polymarket-paper-trader](https://github.com/agent-next/polymarket-paper-trader) (PyPI `polymarket-paper-trader` 0.4.5, 2026-10-01) | MIT | 455 / 2026-10-06 | **Taker:** walks live ask/bid levels and reports slippage in bps. **Fees:** official curve from each market's fee schedule, rounded to 5 dp, makers exempt. **Limits:** GTC/GTD, tick-validated, partial fills; filled "if price crosses" (**no queue model**, so maker fills are optimistic). **Resolution:** manual `resolve` call. CI asserts fills fall inside the Data-API-v2 price band. | Yes (Data API v2, feeSchedule) | **Best reference** for our own taker simulator. Its maker fills are too optimistic. MCP/agent-oriented, so do not install it on the key-holding isolated server. |
| [evan-kolberg/prediction-market-backtesting](https://github.com/evan-kolberg/prediction-market-backtesting) | **Mixed**: LGPL-3.0-or-later for parts; LICENSE says "not MIT-only" | 1,208 / 2026-05-16 | Runs on NautilusTrader. Replays L2 order-book deltas (PMXT hourly archives, or Telonex snapshots plus real trade ticks). **Queue position heuristic:** snapshot of same-side depth, decremented by trades at that price. Latency model: 75 ms base. Fees: `feeSchedule.rate`. Maker rebates credited. ([execution-modeling docs](https://evan-kolberg.github.io/prediction-market-backtesting/execution-modeling/)) | Partly (fee schedule yes; last push predates some V2 changes) | **Most realistic** offline backtester. Heavy. Useful only if a book-level strategy (P3/PM-1) is promoted later. |
| NautilusTrader `SandboxExecutionClient` | LGPL-3.0 | 29,670 / 2026-10-07 | Simulated matching engine running against **live** data. Nothing reaches the venue. ([nautilus-sandbox docs](https://docs.rs/nautilus-sandbox)) | The 1.x Python Polymarket adapter supports signature types 0–2 only. The 2.0.0rc6 Rust adapter (2026-10-05, pre-release) supports Deposit Wallets. | Viable only if we adopt Nautilus. Rejected for now in `01-framework-comparison.md`. |
| [ent0n29/polybot](https://github.com/ent0n29/polybot) | MIT | 1,019 / 2026-02-20 | Java microservices with `hft.mode: PAPER`. The fill model is undocumented. | **No.** Last push was before the 2026-04-28 V2 cutover. | Stale. Read for ideas only. |
| [3kh0/stance](https://github.com/3kh0/stance) | GPL-3.0 | 3 / 2026-09-20 | Terminal UI with a paper mode. | ? | Toy. |
| [pmxt-dev/pmxt](https://github.com/pmxt-dev/pmxt) | MIT | 2,165 / 2026-09-30 | "CCXT for prediction markets", a unified Polymarket/Kalshi API. Not a simulator. | ? | Possible future multi-venue layer. |

**Recommended paper-broker fill model (derived).** It combines what agent-next and evan-kolberg do with conservative choices:

- **Taker fills.** At signal time, snapshot `/book`. Re-snapshot after a 1–3 s simulated latency. Walk the levels and charge `fee = C × rate × p(1−p)` per level from `feeSchedule`.
- **Maker (resting) fills.** Fill only when a public trade prints **through** our price, or prints at our price after the queue ahead of us has been consumed.
- **Journal two P&Ls:** the simulated fill and the mid-price "shadow" fill.

### 1.3 Kalshi demo, for comparison

Kalshi runs a real demo exchange with mock funds:

- Web: `https://demo.kalshi.co/`
- REST: `https://external-api.demo.kalshi.co/trade-api/v2` (alternative `https://demo-api.kalshi.co/trade-api/v2`)
- WebSocket: `wss://external-api-ws.demo.kalshi.co/trade-api/ws/v2`


---

## 2. Official Python SDK

| Item | Value | Source |
|---|---|---|
| PyPI name | **`polymarket-client`**. Install with `pip install polymarket-client` or `uv add polymarket-client`. | [pypi](https://pypi.org/project/polymarket-client/), [docs python](https://docs.polymarket.com/getting-started/python.md) |
| Latest | **0.12.0**, uploaded 2026-10-01. Before it: 0.11.0 (09-23), 0.10.0 (09-10), 0.9.0 (09-04). 36 releases since 0.1.0b1 on 2026-05-21. | [pypi JSON](https://pypi.org/pypi/polymarket-client/json) |
| Python | `requires_python >=3.11`; classifiers 3.11, 3.12, 3.13. Status: "4 - Beta". | pypi |
| Licence / repo | MIT. [Polymarket/py-sdk](https://github.com/Polymarket/py-sdk): 136★, 37 forks, 39 open issues, last push 2026-10-06, created 2026-05-04. | [gh API] |
| Import name | `polymarket` (`from polymarket import AsyncPublicClient, AsyncSecureClient`) | docs |
| Core deps | httpx[http2], websockets <16, pydantic 2, eth-account 0.13+, eth-abi, msgpack. Extras: `pandas`, `polars`, `arrow`, `quant`. | pypi `requires_dist` |
| Stability statement | "Although minor releases on the 0.x line may include breaking changes, we aim to avoid them… Patch releases remain backward compatible except for APIs marked experimental… All Perps APIs are currently experimental." | [README](https://github.com/Polymarket/py-sdk/blob/main/README.md) |
| Real breakage history | **0.10.0** broke positions, trades, price-history, leaderboard and holder APIs (cursors; `list_price_history` replaced `get_price_history`). **0.7.1** removed `valid_until` from session keys. **0.8.0** changed `revoke_session_key` to return `None`. | [SDK changelog](https://docs.polymarket.com/changelog/sdks.md) |

**Coverage** ([docs python](https://docs.polymarket.com/getting-started/python.md), [SDK changelog](https://docs.polymarket.com/changelog/sdks.md)):

| Area | Covered | Auth |
|---|---|---|
| Gamma markets/events/tags/search, order books, prices, price history, Data API v2 (positions, trades, activity, leaderboard, holders, PnL) | ✅ `PublicClient` / `AsyncPublicClient`, typed models, `Decimal` money values, paginators with `to_pandas/polars/arrow` | **None** for public reads |
| Market WebSocket (`book`, `price_change`, `last_trade_price`, `tick_size_change`, `best_bid_ask`, `new_market`, `market_resolved`), sports stream | ✅ `client.subscribe([MarketSpec(...)])`, **async only** | None |
| User WebSocket (orders and trades) | ✅ `UserSpec()` | CLOB L2 creds |
| Limit orders (GTC/GTD), market orders (FAK/FOK), post-only, batch `post_orders`, cancel, heartbeats | ✅ `AsyncSecureClient.place_limit_order(...)` etc. | private key + wallet |
| Deposit Wallet: derive, deploy and trading approvals; gasless ops through the Relayer | ✅ (`get_trading_approvals_state` added in 0.7.0) | Relayer **or** Builder API key |
| Session keys: `authorize_session_key`, `fetch_session_keys`, `revoke_session_key` | ✅ since 0.7.0 (scopes default to **ALL**, so pass `CLOB` explicitly) | owner key + Builder API key |
| Split / merge / redeem | ✅ protocol chosen from market `version` (0.12.0); `REDEEMABLE` / `REDEEMABLE_LOST` / `MERGEABLE` position filters | owner (Deposit Wallet batch through the relayer) |
| Combos RFQ, Collateral Return, PolyBolt price streams, Perps | ✅ (Perps experimental) | various |

**Unofficial or legacy packages. Do not install these next to the official SDK.**

| PyPI name | Version / date | Publisher | What it actually installs | Risk |
|---|---|---|---|---|
| `polymarket` | 0.1.2 / 2024-08-06 | Matt Maximo ("export historical odds") | top-level module **`Polymarket/`** | **Unofficial.** Source repo `MattMaximo/polymarket-py` returns GitHub 404. **On the macOS default case-insensitive filesystem, `Polymarket/` collides with the official `polymarket/`** (derived). |
| `polymarket-sdk` | 0.2.0 / 2026-07-22 | "counterfactual5" | top-level **`polymarket_autopilot`** | **Unofficial.** Source repo returns 404. Treat as suspicious. |
| `polymarket-apis` | 2.3.0 / 2026-06-08 | Razvan Gheorghe | third-party wrapper | Unofficial. |
| `py-clob-client` | 0.34.6 / 2026-02-19 | Polymarket | V1 client | **Dead since the V2 cutover** on 2026-04-28. |
| `py-clob-client-v2` | 1.2.0 / 2026-09-25 | Polymarket | V2 CLOB client (167★, pushed 2026-10-06) | Official but "We recommend Polymarket/py-sdk for new projects". This is the fallback if py-sdk breaks. |

Wheel top-level names were listed from the zip index only, without installing anything ([pypi files](https://pypi.org/project/polymarket/#files)).

---

## 3. Fees

**Formula** [doc]: `fee = C × feeRate × p × (1 − p)`, where C is shares and p is price. Fees are applied **at match time** and are not part of the signed order (V2 removed `feeRateBps` from the order struct). "**Makers are never charged fees.**" Fees are rounded to 5 dp; the minimum charged is 0.00001 USDC. There are no Polymarket deposit or withdraw fees. ([Fees](https://docs.polymarket.com/trading/fees.md))

| Category (doc table) | Taker rate | Maker rate | Maker rebate (share of taker fees) |
|---|---|---|---|
| Crypto | 0.07 | 0 | 20% |
| Sports | 0.05 (raised from 0.03 on 2026-07-10) | 0 | 15% |
| Finance, Politics, Mentions, Tech | 0.04 | 0 | 25% |
| Economics, Culture, Weather, Other/General | 0.05 | 0 | 25% |
| **Geopolitics / world events** | **0 (fee-free)** | 0 | — |

**Live distribution across the first 1,000 open markets** (`/markets/keyset?closed=false`, 10 pages) [live]:

| `feesEnabled` | `feeSchedule.rate` | `feeType` | n |
|---|---|---|---|
| True | 0.04 | politics_fees | 613 |
| **False** | — | — | **107** |
| True | 0.05 | culture_fees | 80 |
| True | 0.05 | economics_fees | 62 |
| True | 0.04 | finance_prices_fees | 46 |
| True | 0.07 | crypto_fees_v2 | 42 |
| True | **0.03** | sports_fees_v2 | 19 (older sports markets; the doc says 0.05) |
| True | 0.04 / 0.05 / 0.05 | tech / general / weather | 17 / 8 / 6 |

All sampled markets had `exponent = 1`, `takerOnly = true` and `version = "v1"` (CTF).

**Fetching the fee rate at runtime:**

| Source | Field | Use? |
|---|---|---|
| Gamma `GET /markets/{id}` or `/markets/keyset` | `feesEnabled`, `feeSchedule{rate, exponent, takerOnly, rebateRate}` | **Yes, authoritative.** Changelog 2026-03-31: "Fees should now be calculated using the `feeSchedule` object within a market." |
| CLOB `GET /clob-markets/{condition_id}` | `fd{r, e, to}` (rate, exponent, taker-only); absent on fee-free markets | Yes, as a cross-check in the same call as tick and min size |
| CLOB `GET /fee-rate?token_id=` | `{"base_fee":1000}` or `{"base_fee":0}` (basis points, legacy) | **No.** It does not give the curve rate. |
| Gamma `takerBaseFee` / `makerBaseFee` | 1000 / 1000 even when the rate is 0.04 | **No.** Legacy field. |

The exponent's semantics are only "exponent applied to the price component" [doc]. The general form `C·rate·(p(1−p))^e` is **derived**: implement it, and assert `e == 1` or skip the market.

**Programs at a $50–150 bankroll (derived):**

- **Taker Rebate tiers** (live since 2026-05-28) need ≥$2,000 of 30-day weighted volume for Bronze (3%). Geopolitics earns 0 wV. **Irrelevant at our size.**
- **Liquidity rewards** have a $1 minimum payout and per-market `rewardsMinSize` of 50–200 shares, with `rewardsMaxSpread` of 4.5¢ in the samples. **Mostly out of reach.**

---

## 4. Order constraints and rate limits

| Constraint | Value | Source |
|---|---|---|
| Tick sizes | 0.1, 0.01, 0.005, 0.0025 (World Cup only), 0.001, 0.0001. Live sample: 802 × 0.001 and 198 × 0.01. Listen for `tick_size_change`. | [Market details](https://docs.polymarket.com/market-data/market-details.md) [live] |
| Price / size decimals | Tick 0.01: price 2 dp, size 2 dp, amount 4 dp. Tick 0.001: 3 / 2 / 5. Amounts are encoded as 6-decimal integers. | [Place orders](https://docs.polymarket.com/trading/place-orders.md) |
| Minimum size | `min_order_size` on `/book` "is the minimum number of **shares**" = **5** in all 1,000 sampled markets. **The docs conflict:** the Gamma field `orderMinSize` is described as "Minimum order size in USDC". The error text is "`Size ({size}) lower than the minimum: {min}`". | [doc] [live] |
| Min notional for marketable orders | "Marketable orders rejected below $1 notional" (NautilusTrader 2.0 docs, quoted in `01`). **Unverified** in Polymarket docs. | — |
| Order types | GTC; GTD (expiration must be ≥3 min in the future, and the order expires 1 min early, so effective lifetime ≥ ~2 min); FOK; FAK. Market orders are FAK or FOK with `expiration = "0"`. A market BUY is sized in USD. | [doc] |
| Post-only | `postOnly: true` with **GTC/GTD only**. Rejected with `invalid post-only order: order crosses book`. | [doc] |
| Batch | `POST /orders` takes **max 15** orders (processed in parallel). `DELETE /orders` takes max 1,000 IDs. `POST /books`, `/prices`, `/midpoints`, `/spreads` and `/last-trades-prices` take max 500 items. | [api-ref](https://docs.polymarket.com/api-reference/trade/post-multiple-orders.md), changelog 2026-06-15 |
| Not supported | reduce-only, stop, OCO, iceberg (per NautilusTrader docs, `01`) | — |
| Heartbeat (dead-man switch) | Opt-in. "Once the first heartbeat is accepted… If a valid heartbeat is not received within **10 seconds**, all open orders… are canceled." | [Manage orders](https://docs.polymarket.com/trading/manage-orders.md) |
| Delays | Some crypto/finance up/down markets have a taker delay: **50 ms** since 2026-08-17 (the order-lifecycle page still says 250 ms). Sports markets have a game delay (`secondsDelay`). A delayed order cannot be canceled. | changelog, [order lifecycle](https://docs.polymarket.com/concepts/order-lifecycle.md) |
| Engine restarts | HTTP **425** during a restart, then **2 minutes of post-only mode**. **503** means cancel-only or disabled. Restarts are announced about 2 days ahead on Telegram `t.me/polytradingapis` and Discord. | [Matching engine](https://docs.polymarket.com/trading/matching-engine.md) |
| Order response | Since 2026-07-24, FAK/FOK matches return `tradeIDs`, not `transactionHashes`. Poll trades for the hash. | changelog |

**Rate limits.** Cloudflare limits are per IP over a sliding window, and excess requests are **throttled (queued), not rejected** ([Rate limits](https://docs.polymarket.com/api-reference/rate-limits.md)):

| API | Limit (requests per 10 s) |
|---|---|
| Gamma | 4,000 general; `/events` 500; `/markets` 300; listing 900; `/public-search` 350; `/tags` 200 |
| Data v2 | 800 general; `/v2/trades` 300; `/v2/positions` 200; `/v2/prices-history` 200 |
| CLOB market data | 9,000 general; `/book` 1,500; `/books` 500; `/price` and `/midpoint` 1,500; `/prices-history` 1,000; tick size 200 |
| CLOB trading (burst / sustained) | `POST /order` 5,000/10 s and 120,000/10 min; `POST /orders` 2,000 and 21,000; `DELETE /cancel-all` 250 and 6,000 |
| Bridge | 50; Relayer `/submit` 25 per minute |

**Per-signer token buckets** ([trading rate limits](https://docs.polymarket.com/api-reference/trading-rate-limits.md)) ran in warning mode from 2026-07-24; Polymarket will announce live enforcement.

- **Standard tier:** orders 40/s with burst 60; cancels 80/s with burst 120.
- **Higher tiers** start at a 30-day volume of $30k.
- **Headers:** `Poly-RateLimit-Remaining`, `-Reset`, `-Tier`, `-Warning`, and `Retry-After` on 429.

The SDK exposes these through `on_rate_limit_update`.

---

## 6. Geoblock

- **Endpoint:** `GET https://polymarket.com/api/geoblock`. It is on the website host, not the API host. Response: `{blocked, ip, country, region}` ([Geoblock](https://docs.polymarket.com/api-reference/geoblock.md)).

| Group (doc) | Jurisdictions |
|---|---|
| **Block completely** (frontend and API; cannot close positions) | IR, SY, CU, KP, UA-43 (Crimea), UA-14 (Donetsk), UA-09 (Luhansk) |
| **Close-only, frontend and API** | AU, BY, BE, BI, BR, CA-BC, CA-ON, CA-AB, CA-QC, CF, CD, ET, FR, DE, IQ, IT, LB, LY, MM, NZ, NI, KP, PL, RU, SG, SO, SK, SS, SD, TW, TH, GB, US, UM, VE, YE, ZW |


**Other facts:**

- Primary servers are in `eu-west-2` (London); the "closest non-georestricted region" is `eu-west-1` [doc].
- Markets carry `restricted: true` ("geo-restricted for some jurisdictions"). Example: the Iran-blockade market. The affected jurisdictions are not listed, so log the flag and exclude these markets by default (derived).
- Using a VPN to evade the geoblock breaks Polymarket's restriction regime. Check eligibility from the intended execution environment; do not bypass restrictions.

---

## 7. Data endpoints for research

**Gamma** (`https://gamma-api.polymarket.com`, no auth). Endpoints:

- `/markets`, `/markets/keyset` (max `limit=100`, `after_cursor`)
- `/markets/slug/{slug}`, `/events`, `/events/keyset?tag_id=`
- `/tags`, `/tags/slug/{slug}` (geopolitics = **100265**)
- `/public-search?q=`, `/series`, `/sports`

`GET /markets` defaults to `closed=false` since 2026-04-09.

**Market fields observed live:**

| Purpose | Fields (exact names) |
|---|---|
| IDs | `id`, `conditionId`, `questionID`, `slug`, `clobTokenIds` (**JSON string**), `outcomes` (JSON string), `version` (`v1` = CTF token IDs; `v2` = Protocol V2 `positionId`) |
| Price / liquidity | `outcomePrices` (JSON string), `bestBid`, `bestAsk`, `spread`, `lastTradePrice`, `liquidityNum`, `volume24hr`, `volume1wk`, `oneDayPriceChange` |
| Fees / constraints | `feesEnabled`, `feeSchedule`, `feeType`, `orderMinSize`, `orderPriceMinTickSize`, `acceptingOrders`, `enableOrderBook`, `restricted`, `rfqEnabled` |
| Neg-risk | `negRisk`, `negRiskMarketID`, `negRiskOther`; on the event: `enableNegRisk`, `negRiskAugmented` |
| Rewards | `rewardsMinSize`, `rewardsMaxSpread`, `holdingRewardsEnabled` |
| Resolution / timing | `endDate`, `startDate`, `resolutionSource` (often `""`; the rules live in `description`), **`umaResolutionStatuses`** (plural, JSON-string array, e.g. `["proposed"]`), `umaBond`, `umaReward`, `customLiveness` |
| Category | **`category` was `null`** on both markets and events in the samples. Use the event's `tags[]` (e.g. `politics` id 2, `geopolitics` id 100265) and `feeType`. |

**CLOB public** (`https://clob.polymarket.com`, no auth):

- **Order books and prices:** `/book?token_id=`, `POST /books`, `/price`, `/midpoint`, `/spread`, `/last-trade-price`, `/tick-size?token_id=`
- **Market parameters:** `/clob-markets/{condition_id}`
- **Legacy:** `/fee-rate` (bps), `/prices-history?market=&interval=&fidelity=` (still answers, but superseded)
- **Server:** `/time`, `/ok`

**`/book` ordering [live]:** bids are listed **ascending** and asks **descending**, so the best bid and best ask are the **last** elements. Do not take `[0]`.

**Data API v2** (`https://data-api.polymarket.com/v2`, released 2026-09-04; v1 is frozen):

- **Routes:** `/positions`, `/positions/combos`, `/trades`, `/activity`, `/value`, `/holders`, `/oi`, `/live-volume`, `/user-pnl`, `/user-stats`, `/user-volume`, `/leaderboard`, `/biggest-winners`, `/builders/leaderboard`, `/builders/volume`, `/resolutions`, `/prices-history`, `/approvals`, `/status`, `/openapi.json` ([overview](https://docs.polymarket.com/api-reference/data-api/overview.md)).
- **Responses:** an envelope `{data, pagination{limit, offset, has_more, next_cursor}}`.
- **`/v2/leaderboard`:** `time_period=day|week|month|all`, `category=overall|<gamma category>|combos`, `sort_by=PNL|VOLUME`.
- **`/v2/resolutions`:** needs exactly one of `question_id`, `condition` or `event_id`. Returns `status`, `was_disputed`, `proposed_price`, `expected_settlement_time` and `settlement_time_basis`. Useful for resolution-risk features.

**`/v2/prices-history`** [doc] ([token price history](https://docs.polymarket.com/api-reference/markets/get-a-tokens-price-history.md)):

- **Parameters:** `token_id`, plus exactly one of `interval` (`max|all|1m|1w|1d|6h|1h`), `start` + `end`, or `as_of`. Optional: `bucket_seconds`, `limit` (≤10,000), `cursor`.
- **Explicit start/end windows cap at 15 days.**
- **Retention:** 3 h and 12 h buckets are permanent back to 2022-11-18. 1-min data is kept ≥7 days, 5-min ≥60 days, 30-min ≥90 days.

**WebSockets** [doc]:

| Channel | URL | Notes |
|---|---|---|
| Market (public) | `wss://ws-subscriptions-clob.polymarket.com/ws/market` | Subscribe with `{"assets_ids":[…],"type":"market"}`. Send `PING` every 10 s. `custom_feature_enabled` adds `best_bid_ask`, `new_market` and `market_resolved`. |
| User (auth) | `wss://ws-subscriptions-clob.polymarket.com/ws/user` | CLOB L2 creds; `PING` every 10 s |
| Sports | `wss://sports-api.polymarket.com/ws` | The server sends `ping` every 5 s; answer `pong` within 10 s |
| PolyBolt reference prices | `wss://ws-live-v2.polymarket.com/ws` | Needs CLOB creds; replaces RTDS `wss://ws-live-data.polymarket.com` |

**Free historical data** (bulk; read-only, licences as stated) [gh]:

| Source | Content | Licence / size / activity |
|---|---|---|
| [Jon-Becker/prediction-market-analysis](https://github.com/Jon-Becker/prediction-market-analysis) | "largest publicly available dataset of Polymarket and Kalshi market and trade data", Parquet; `data.tar.zst` **36 GiB compressed** on R2 | MIT, 3,860★, pushed 2026-10-04; README cites the 2026 favorite-longshot and calibration papers |
| [warproxxx/poly_data](https://github.com/warproxxx/poly_data) | Retriever for markets, order events and trades | GPL-3.0, 2,332★, pushed 2026-09-08 |
| [SII-WANGZJ/Polymarket_data](https://github.com/SII-WANGZJ/Polymarket_data) (HuggingFace) | On-chain trades plus Gamma metadata. The description says 1.1B records; a search snippet claims 1.9B / 163 GB (unverified). | MIT, 858★, last push 2026-01-01 (**pre-V2**) |
| pmxt archive (`archive.pmxt.dev`) | Hourly order-book and trade Parquet dumps; source of the L2 replays used by evan-kolberg's backtester | **Could not load on 2026-10-07** (connection reset); unverified |
| Goldsky / CryptoHouse (ClickHouse SQL), Dune, Allium | On-chain trades, balances, positions | Listed on [Data Resources](https://docs.polymarket.com/resources/blockchain-data.md). Note: "Third-party schemas can lag contract upgrades." |

---

## 8. Live sample calls (trimmed; 2026-10-07 ~04:20 UTC)

```text
GET https://clob.polymarket.com/time           -> 1791361275
GET https://clob.polymarket.com/ok             -> "OK"
GET https://clob-staging.polymarket.com/ok     -> curl: (6) Could not resolve host
```

```jsonc
// GET https://gamma-api.polymarket.com/markets?limit=5&closed=false&order=volume24hr&ascending=false   (one row, trimmed)
{"question":"US announces end of Iranian blockade by October 15, 2026?",
 "conditionId":"0x1674c7a6…f028","clobTokenIds":"[\"34545997…2022\", \"31187265…2781\"]",
 "outcomePrices":"[\"0.085\", \"0.915\"]","bestBid":0.08,"bestAsk":0.09,"spread":0.01,
 "feesEnabled":false,"feeSchedule":null,"takerBaseFee":null,"orderMinSize":5,"orderPriceMinTickSize":0.01,
 "negRisk":false,"version":"v1","restricted":true,"umaResolutionStatuses":"[]","resolutionSource":"",
 "endDate":"2026-10-16T03:59:00Z"}
// a sports row from the same call:
{"feesEnabled":true,"feeSchedule":{"exponent":1,"rate":0.05,"takerOnly":true,"rebateRate":0.15},
 "feeType":"sports_fees_v3","takerBaseFee":1000,"makerBaseFee":1000}
```

```jsonc
// GET https://clob.polymarket.com/fee-rate?token_id=34545997…2022   (geopolitics)  -> {"base_fee":0}
// GET https://clob.polymarket.com/fee-rate?token_id=30357198…2406   (sports)       -> {"base_fee":1000}
// GET https://clob.polymarket.com/tick-size?token_id=34545997…2022                 -> {"minimum_tick_size":0.01}
// GET https://clob.polymarket.com/clob-markets/0x07d9a47d…5ebd   (sports, trimmed)
{"gst":"2026-10-07T08:30:00Z","t":[{"t":"30357198…2406","o":"GAM Esports"},{"t":"45610906…5867","o":"Shopify Rebellion"}],
 "mos":5,"mts":0.01,"mbf":1000,"tbf":1000,"ao":true,"ibce":true,"fd":{"r":0.05,"e":1,"to":true},"v":"v1"}
```

```jsonc
// GET https://clob.polymarket.com/book?token_id=34545997…2022   (tail of each side; best = LAST element)
{"market":"0x1674c7a6…f028","timestamp":"1791361406003",
 "bids":[…,{"price":"0.06","size":"47339.69"},{"price":"0.07","size":"15138.81"},{"price":"0.08","size":"32656.52"}],
 "asks":[…,{"price":"0.11","size":"12471.37"},{"price":"0.1","size":"8590.54"},{"price":"0.09","size":"4778.17"}],
 "min_order_size":"5","tick_size":"0.01","neg_risk":false,"last_trade_price":"0.090"}
```

```jsonc
// GET https://data-api.polymarket.com/v2/prices-history?token_id=34545997…2022&interval=1w&bucket_seconds=86400
{"data":[{"timestamp":1790726400,"price":0.165,"resolution_seconds":86400}, …,
         {"timestamp":1791361440,"price":0.085,"resolution_seconds":0}],     // terminal point
 "pagination":{"limit":10000,"offset":0,"has_more":false,"next_cursor":null}}
// (param name is token_id; asset_id -> {"error":"required query param 'token_id' not provided"})
```

```jsonc
// GET https://data-api.polymarket.com/v2/resolutions?condition=0xbfacb085…1bbe
{"data":[{"status":"proposed","was_disputed":false,"proposed_price":"1000000000000000000",
  "expected_settlement_time":"2026-10-07T08:31:06Z","settlement_time_basis":"managed_proposal_expiration"}]}
// GET https://data-api.polymarket.com/v2/leaderboard?time_period=month&limit=3  -> rank 1 pnl ≈ $3.65M on $28.2M volume (monthly)
// GET https://bridge.polymarket.com/supported-assets -> 235 assets; e.g. Tron USDT minCheckoutUsd 2, Polygon USDC 2, Ethereum USDC 3
```

---

## 9. Implementation checklist derived from these facts

1. **Paper phase = in-process simulator on live data.** Use the taker book-walk, the conservative maker rule (section 1.2) and the per-market `feeSchedule`. Then **micro-live** with $10–20 to verify signing, Deposit Wallet approvals, fills, fee accounting and redemption or auto-redeem.
2. **Pin the SDK.** Use `polymarket-client==0.12.0` (hash-pinned) on Python 3.12. Never install `polymarket` or `polymarket-sdk`. Keep `py-clob-client-v2==1.2.0` as the documented fallback.
3. **Fee engine.**
   - Read `feesEnabled` and `feeSchedule` per market at decision time; `feesEnabled == false` means fee 0.
   - Skip the market if `exponent != 1` or `takerOnly != true`.
   - Ignore `/fee-rate`, `takerBaseFee` and the category table.
4. **Sizing floor.** A ticket of ≥5 shares costs ≥ 5·p, so add a gate: `5·p ≤ per_trade_cap`. At $50 with a 5% cap, only p ≤ 0.50 is tradable. Decide the cap with the owner.
5. **Execution.** Use a marketable limit (GTC at the best ask, or FAK with a price cap) rather than FOK. Respect the tick from `/book` or the `tick_size_change` event. Do not enable heartbeats unless orders rest, because a crash or the 10 s rule would otherwise cancel everything (which may in fact be desired; owner decision). Handle 425, 503 and 429 with `Retry-After`.
6. **Reconciliation.**
   - Use the venue as truth: `/v2/positions?user=<deposit_wallet>` (public) plus the user WebSocket under whichever credentials placed the orders.
   - A session key cannot see the owner's orders, and the owner cannot see the session key's.
7. **Custody.** Plan A: email builder@polymarket.com early (session-key enablement may take time). Plan B: a dedicated Deposit Wallet created through path B with a float-only owner key on the isolated server.
8. **Start-up assertions.** Geoblock `blocked == false` from the isolated server IP. `/ok` returns 200. Clock skew against `/time` is under 2 s. Exclude markets with `restricted == true` and markets where `acceptingOrders == false`.
9. **Market selection.** Use `tags[]` (geopolitics 100265 for fee-free candidates) together with `feesEnabled`. Treat `version == "v2"` markets as a separate code path (`positionId`, ExchangeV3, domain `"3"`). No open v2 markets were seen in the 1,000-market sample.

## 10. Open or unverified items

| Item | Why it matters | How to close it |
|---|---|---|
| Session-key enablement for an individual builder | Custody plan A | Owner emails builder@polymarket.com (Phase 3) |
| Can a session key, or the AutoRedeemer, redeem without the owner? | Whether the redeem job needs the owner key | $1 micro-live test; ask in the builder Telegram |
| `orderMinSize` unit (shares vs USDC) and the $1 marketable minimum | Sizing gate | Micro-live: try 5 shares at p ≤ 0.19, where notional < $1 |
| Signer-key export for post-May-2026 email accounts | Onboarding path A | Prefer path B or a wallet login |
| pmxt archive contents and licence | Possible future L2 backtests | Retry `archive.pmxt.dev` |

## Sources (all accessed 2026-10-07)

- **Polymarket docs index:** https://docs.polymarket.com/llms.txt
- **Polymarket docs pages:** fees, market-details, place-orders, manage-orders, wallets-auth, session-keys, positions/manage, concepts/pusd, concepts/order-lifecycle, bridge/deposit, bridge/withdraw, bridge/supported-assets, programs (builders overview and tiers, taker-rebates, maker-rebates, liquidity-rewards), api-reference (rate-limits, trading-rate-limits, geoblock, data-api/overview, get-a-tokens-price-history, get-clob-market-info, get-fee-rate, post-multiple-orders, send-heartbeat, boards/get-the-trader-leaderboard), changelog/predictions, changelog/sdks, resources (contracts, blockchain-data, error-codes), trading/matching-engine, market-data/realtime-data. All at `https://docs.polymarket.com/<path>.md`.
- **PyPI:** https://pypi.org/pypi/polymarket-client/json, …/polymarket/json, …/polymarket-sdk/json, …/py-clob-client/json, …/py-clob-client-v2/json, …/polymarket-apis/json, …/nautilus_trader/json, …/polymarket-paper-trader/json
- **GitHub:**
  - Polymarket: https://github.com/Polymarket/py-sdk (README, `src/polymarket/environments.py`, `.env.example`) and https://github.com/Polymarket/py-clob-client-v2 (`py_clob_client_v2/config.py`, `constants.py`, README)
  - Simulators and data: agent-next/polymarket-paper-trader, evan-kolberg/prediction-market-backtesting (+ https://evan-kolberg.github.io/prediction-market-backtesting/execution-modeling/), ent0n29/polybot, 3kh0/stance, pmxt-dev/pmxt, Jon-Becker/prediction-market-analysis, warproxxx/poly_data, SII-WANGZJ/Polymarket_data, nautechsystems/nautilus_trader
- **Live endpoints:** gamma-api.polymarket.com (`/markets`, `/markets/keyset`, `/events/699735`, `/tags/slug/geopolitics`); clob.polymarket.com (`/ok`, `/time`, `/book`, `/fee-rate`, `/tick-size`, `/clob-markets/{cid}`, `/prices-history`); data-api.polymarket.com (`/v2/prices-history`, `/v2/leaderboard`, `/v1/leaderboard`, `/v2/trades`, `/v2/resolutions`); bridge.polymarket.com/supported-assets; polymarket.com/api/geoblock
- **Kalshi:** https://docs.kalshi.com/getting_started/demo_env
- **KYC:**
  - https://www.cryptotimes.io/2026/05/28/polymarket-denies-mandatory-kyc-plans-for-main-platform/
  - https://www.crowdfundinsider.com/2026/05/282200-prediction-markets-polymarket-pushes-for-trader-identity-verification-due-to-regulatory-and-sanctions-pressures/
  - https://help.polymarket.com/en/articles/13364258-how-do-i-export-my-key
- **NautilusTrader sandbox:** https://docs.rs/nautilus-sandbox

## Verification (2026-10-07)

Adversarial re-check of 10 headline claims. Each was re-opened at its cited source and checked against at least one independent source: live public APIs, `dig`, py-sdk source on GitHub, the docs OpenAPI spec, the changelog and news coverage. Everything was read-only. Live sample: `gamma-api /markets?closed=false&active=true&order=volume24hr`, 10 pages × 100 = 1,000 rows (971 unique ids, because the ranking shifts during paging), pulled ~08:30 UTC. Labels: **derived** means computed by us; **unverified** means not independently reproducible.

| # | Claim | Verdict | Note | Better source |
|---|---|---|---|---|
| 1 | No public CLOB V2 testnet: staging host is listed but does not resolve; py-sdk has only PRODUCTION (chain 137) | confirmed | `clob-openapi.yaml` still lists `clob-staging.polymarket.com` under `servers:`. DNS checks returned no A, AAAA or CNAME record and `curl` failed. **Wording fix:** the DNS status is NOERROR/NODATA, not NXDOMAIN, because the Cloudflare zone answers NOERROR+SOA even for a random control name. `environments.py` defines exactly one `_create_environment` (PRODUCTION, `chain_id=137`). The archived legacy `py-clob-client` still carries Amoy (80002) contract configs, but those are V1 and unusable for V2. Third-party "Amoy" recipes (e.g. Cobo) show no V2 CLOB host. | https://docs.polymarket.com/api-spec/clob-openapi.yaml ; https://github.com/Polymarket/py-clob-client/blob/main/py_clob_client/config.py |
| 2 | `polymarket-client` 0.12.0, uploaded 2026-10-01, Python >=3.11, MIT, Beta; 0.x minors may break | confirmed | PyPI JSON: version 0.12.0, files uploaded 2026-10-01T13:56Z, `requires_python >=3.11`, classifiers `Development Status :: 4 - Beta` and `MIT`. The GitHub repo license is MIT and the repo is not archived (pushed 2026-10-07). README line 77: minor 0.x releases "may include breaking changes". | https://github.com/Polymarket/py-sdk (README) |
| 3 | Taker fee = C·rate·p·(1−p); makers free; category rates (Sports 0.05 …); 5 dp rounding | partly | The doc text matches exactly (formula, "Makers are never charged", table, "rounded to 5 decimal places", min 0.00001). **Live rates deviate from the table:** 76/1,000 sampled markets are sports at **0.03**. 57 are `sports_fees_v2`, created 2025-09 to 2026-07. 19 are `sports_fees_nfl_cfb_oct26`, created 2026-08-28 to 09-28, *after* the 2026-07-10 0.03→0.05 change, so the 0.03 rate is not only on legacy markets (derived). Other live rates: crypto 0.07 (163), sports_v3 0.05 (160), politics 0.04 (186), weather/culture/economics/general 0.05, tech/finance 0.04, and 99 markets with no `feeSchedule` (fee-free). No Mentions market appeared in the sample. | https://docs.polymarket.com/changelog/predictions.md (Jul 10 2026 entry) ; live `gamma-api.polymarket.com/markets` `feeSchedule`/`feeType` |
| 4 | Read fee from Gamma `feeSchedule` (changelog 2026-03-31); CLOB `/fee-rate` returns legacy `base_fee` bps (1000 / 0) | confirmed | Changelog "Mar 31, 2026 — Fees should now be calculated using the `feeSchedule` object" (`feesEnabled` is a Gamma field the entry does not name). Live: `/fee-rate` returned `{"base_fee":1000}` for **all 11** fee types with rates from 0.03 to 0.07, and `{"base_fee":0}` for a fee-free (Iran blockade) market. So `base_fee` is effectively an on/off flag, not the curve rate (derived). The OpenAPI still describes it as "Base fee in basis points" with an example value of 30. | live `clob.polymarket.com/fee-rate?token_id=` vs Gamma `feeSchedule` |
| 5 | Min size 5, tick 0.001/0.01 in sample; batch max 15; GTD ≥3 min; heartbeat 10 s cancels all | confirmed | Reproduced: 1,000/1,000 `orderMinSize`=5; tick 0.001 (543) or 0.01 (457); `/book` returns `min_order_size:"5"`. **Batch max 15 is not stated in place-orders.md.** It is in `clob-openapi.yaml` ("Maximum 15 orders per request", `maxItems: 15`). GTD: "at least **3 minutes** in the future" plus a 60 s security threshold. Heartbeat: opt-in (applies once the first heartbeat is accepted), timeout 10 s, the cancel check runs every 5 s (so effectively 10–15 s), send every 5 s. **Caveats:** the docs conflict on units (market-details says `orderMinSize` is "in USDC", place-orders says shares). The changelog of 2026-07-02 moved World Cup markets to a **0.0025** tick, so do not hard-code {0.001, 0.01}. | https://docs.polymarket.com/api-spec/clob-openapi.yaml ; https://docs.polymarket.com/market-data/market-details.md |
| 6 | Session keys: beta, Deposit-Wallet-only, no withdraw, 180 days fixed, scopes CLOB/COMBOSRFQ/ALL, need Builder key + email; owner cannot fetch session-key orders | confirmed | Every sub-point is in the doc: "Session Keys are in beta"; "work only with Deposit Wallets"; "cannot withdraw"; 180 days, and "Configurable shorter … expirations are not currently supported" (revoke to end early); Builder API key required; builder@polymarket.com during rollout; "A Deposit Wallet Owner cannot fetch orders submitted by its authorized Session Keys". Independent: py-sdk `SessionKeyKnownScope` = `ALL`, `CLOB`, `COMBOSRFQ`, and `ALL` cannot be combined with another scope. The SDK notes that combo session-key actions are not yet supported. | https://github.com/Polymarket/py-sdk/blob/main/src/polymarket/session_keys.py |
| 8 | Standard per-signer 40/s (burst 60) orders, 80/s (burst 120) cancels; Gamma `/markets` 300 req/10s; Cloudflare throttles not rejects | confirmed | All numbers match the docs tables. Cloudflare limits: "throttled (delayed/queued) rather than immediately rejected". **Caveat:** the per-signer limiter started in *warning mode* on 2026-07-24 ("Polymarket will announce when live enforcement begins"). No enforcement announcement appears in the changelog through 2026-10-02, so check the `Poly-RateLimit-Warning` header. Docs only; not load-tested (by design). | https://docs.polymarket.com/api-reference/rate-limits.md |
| 9 | VP Eng stated on 2026-05-28 that no KYC is being added to existing polymarket.com; KYC applies to a beta product | partly | The substance is confirmed by Cointelegraph (via TradingView) and CoinMarketCap Academy: Josh Stevens on X, "No KYC is being added to any part of existing polymarket.com with this launch". **The date is off by one:** the X post id 2059657154093535449 decodes to **2026-05-27 15:25 UTC** (derived from the snowflake id); 05-28 is the article date. The beta product is Perps by press inference; Stevens did not name it. A counter-signal (unverified): The Information, via secondary coverage, reported that Polymarket asks *some* customers to identify themselves and is tightening VPN/IP blocks. KYC risk is low but not zero. | https://www.tradingview.com/news/cointelegraph:4eaabcbbf094b:0-polymarket-exec-says-kyc-limited-to-beta-product-not-existing-platform/ ; https://x.com/devjoshstevens/status/2059657154093535449 |
| 10 | Live bridge minimums $2 (Polygon/Tron/Solana/Arbitrum/Base/BSC), $3 ETH, $3.6 BTC, below the docs table ($7 ETH, $9 BTC/Tron); withdrawals instant and free | confirmed | Live `/supported-assets`: every chain at `minCheckoutUsd` 2 (incl. Tron, Optimism, Monad, HyperEVM, Lightning) except Ethereum 3 and Bitcoin 3.6. The docs table says ETH $7, BTC $9, Tron $9, and the deposit page says to check `/supported-assets` for current minimums. Withdraw doc: "Withdrawals are **instant** and **free** — Polymarket does not charge withdrawal fees". The withdrawal *quote* endpoint still returns a fee breakdown, so network/bridge costs can apply; "free" means no Polymarket fee. | https://docs.polymarket.com/trading/bridge/withdraw.md ; https://docs.polymarket.com/trading/bridge/supported-assets.md |

**Net effect on the design:** no claim was refuted. Two corrections to carry forward:

1. **Fees:** the 0.03 sports rate is still being assigned to *new* NFL/CFB markets. This strengthens F3: always read `feeSchedule.rate` per market.
2. **Tick size:** must be read per market. A 0.0025 tick exists.

**Independent re-run (2026-10-07, ~14:00 local; the section was not duplicated).** Every verdict above reproduced against a *different* sample and sources:

| Check | Result |
|---|---|
| DNS `clob-staging` | The DNS check returned NOERROR with an SOA but no address; the record was absent at the time of the check, rather than NXDOMAIN. Retest using a resolver of your choice. |
| py-sdk / PyPI | `environments.py` `__all__ = ["Environment","PRODUCTION"]`, `chain_id=137`. PyPI shows 0.12.0, wheel uploaded 2026-10-01T13:56:12Z, `>=3.11`, MIT, `4 - Beta`. |
| Gamma sample #2 | `order=id desc` (newest), 10×100 rows gave 993 unique. `orderMinSize`=5 for all 993. Tick 0.01 (989) or 0.001 (4). Rates: sports_v3 0.05 (836), crypto_v2 0.07 (124), politics 0.04 (19), **`sports_fees_nfl_cfb_oct26` 0.03 (7, created 2026-10-07)**, no schedule (7). `feeSchedule` = `{exponent:1, rate, takerOnly:true, rebateRate}` (derived from the live API). |
| `/fee-rate` | `base_fee` is 1000 for each of rates 0.03, 0.04, 0.05 and 0.07, and 0 for a fee-free geopolitics market. This re-confirms that `base_fee` is an on/off flag. |
| Heartbeat source | The 10 s timeout and 5 s check are in `trading/manage-orders.md` (lines ~1485–1493), not in place-orders.md. |
| KYC date | ForkLog independently says the statement was made "on May 27", which matches the snowflake decode (2026-05-27 15:25 UTC). Source: https://forklog.com/en/news/polymarket-executive-denies-mandatory-kyc-implementation |
| Bridge live | All chains are at `minCheckoutUsd` 2 except Ethereum 3 and Bitcoin 3.6. The docs table still lists ETH $7, BTC $9 and Tron $9. |
