# Data sources


**How to read the tables**

| Column | Meaning |
|---|---|
| Auth | `none`, `free key` (free signup, no card unless stated), `account` (OAuth or approval step), `paid` |
| Tested | HTTP status the researcher saw on 2026-10-08. **✓** = the independent retest agreed. **⚠️** = the retest disagreed, or the keyless call did not prove the endpoint (explained in the row or the section note) |
| Priority | **v1** = used by P1, P6 or P3 in Phase 1-2, or a required safety gate. **later** = useful, not needed yet. **reject** = listed only in §9. Where this file overrides the researcher's priority, the row says so |
| Terms | Licence or ToS point that matters for a profit-seeking personal bot. "Not legal advice" applies to every Terms cell. **Paper only until owner OK (D-038)** marks non-commercial or personal-use terms: use the source in paper research only; a live decision that depends on it needs the owner's sign-off on its terms |

**Scope rules.** Only endpoints that returned data, or a clean auth/validation error from a live host, are in §§2-8. Sources that failed both tests and are not rejected are omitted (see the end of §11). Sources that are rejected, retired, paid, unofficial or ToS-restricted are in §9. Per-area sources for the 29 betting areas (303 rows) are not copied here: §1b gives each area's strategy, primary v1 sources and keys, and links research/09 §3 for the tables.

**Priority changes versus the research notes.** Upgraded to v1 because BUILD_PLAN 1.4 needs them in Phase 1: Data API v2 `/v2/trades` and the market WebSocket. Downgraded to later because their strategy is backlog or excluded from P1 (STRATEGIES §P1): xtracker (P7), Binance klines (crypto price markets), liquidity rewards (P4), Pyth (paid). Groq upgraded from later to v1 for short parse/repair tasks (research/06 §4.1).

---

## 1. What feeds a bet decision

Each table is the "big picture" for one strategy: the decision input, where it comes from, and whether it is in v1. Section numbers point to the detailed tables. **No LLM ever sees the market price in the P1 pass-1 forecast** (ARCHITECTURE §8); price enters only the blend and the edge check.

### P1 - market-anchored LLM forecast (Phase 2, build now)

| Decision input | Source(s) | § | v1? |
|---|---|---|---|
| Candidate universe: binary, 3-45 days, liquidity, volume, tags, `restricted`, neg-risk | Gamma `/markets/keyset`, `/events/keyset`, `/tags` | 2 | yes |
| Rules text, end date, `resolutionSource`, UMA status (given to the LLM) | Gamma market `description` and fields | 2 | yes |
| Market probability (blend and edge only, never in pass 1) | CLOB `POST /midpoints`, `/spread`; Data API `/v2/prices-history` | 2 | yes |
| Executable depth, tick, min size, fee schedule, neg-risk flag | CLOB `/book`, `/books`, `/clob-markets/{condition_id}` | 2 | yes |
| Resolution-state guard (proposed, disputed) | Data API `/v2/resolutions`, `/v2/tokens` | 2 | yes |
| News discovery (headlines, publishers, dates) | Google News RSS (titles only; paper only until owner OK, D-038), Wikipedia Current events, GDELT DOC (best effort) | 4 | yes |
| Web search for evidence | Exa, then Tavily (Brave optional) | 4 | yes |
| Article text for the prompt | Jina Reader (keyless); Firecrawl or Gemini URL context as fallback | 4 | yes |
| Entity background, "did it already happen" hints, attention flag | Wikipedia REST summary and revisions, Wikimedia Pageviews (flag only) | 5 | yes |
| Official facts by `resolutionSource` domain | Fed, NY Fed, FRED, BLS, Treasury, Cleveland Fed, Federal Register, Congress.gov, DOJ, CourtListener, SEC EDGAR, Eurostat, PortWatch, LMArena | 6 | adapters built lazily, in frequency order |
| Cross-venue sanity bound | Kalshi public API, hand-built series mapping | 8 | **later** (P10 or a cross-market signal; Developer Agreement unread); stored in `signal_snapshots` when built |
| The forecaster: samples A/B/C, rule parsing, JSON repair | Gemini free (samples A, B); Vercel AI Gateway free credit (sample C and short roles, D-036); Mistral free, OpenRouter `:free`, Groq as backups; paid fallback DeepSeek, OpenAI, OpenRouter under the D-015 cap | 3 | yes |

### P6 - weather ensembles, maker-only (paper in Phase 2)

| Decision input | Source(s) | § | v1? |
|---|---|---|---|
| Market list and rules text (station, unit, rounding, fallback) | Gamma `tag_slug=weather` events; rules parsed once per city template (LLM assist, human check) | 2, 7 | yes |
| Fee rate (0.05, maker rebate 25%), tick, neg-risk | CLOB `/clob-markets/{condition_id}` | 2 | yes |
| Maker price and depth per bucket | CLOB `/book`, `/books`, `/midpoints` | 2 | yes |
| Live running max/min at the resolution station | AviationWeather METAR (48 of 49 ICAOs), NWS API (US), HKO since-midnight feed (Hong Kong) | 7 | yes |
| Station history for bias and rounding calibration | Iowa Mesonet ASOS archive | 7 | yes |
| Ensemble members | NOAA GEFS (AWS or NOMADS), ECMWF ENS open data (byte ranges) | 7 | yes |
| Fast paper-phase scoring | Open-Meteo ensemble API (non-commercial free tier) | 7 | paper only until owner OK (D-038); licence open (D-029) |
| Settlement check the morning after | HKO Daily Extract; weather.gov hourly "Temp" column (METAR) | 7 | yes |
| More ensemble families, London benchmark | DWD ICON-EPS, ECCC GEPS, Met Office DataHub | 7 | later |

### P3 - neg-risk sum-of-asks observer (observe-only, Phase 1)

| Decision input | Source(s) | § | v1? |
|---|---|---|---|
| Neg-risk event and leg list | Gamma `/events/keyset` (`negRisk`) | 2 | yes |
| Best YES ask of every leg, with depth | CLOB `POST /books` (up to 500 tokens per call) | 2 | yes |
| Fee per leg | CLOB `/clob-markets/{condition_id}` | 2 | yes |
| Episode duration and our order round trip | Market WebSocket or polling; CLOB `/time` | 2 | yes |

No key, no LLM and no evidence source is involved.

### Later strategies

| Strategy | Decision inputs and sources | v1? |
|---|---|---|
| **P2** rules-lawyer and near-resolution bonds | Gamma rules; Data API `/v2/resolutions`; `/v2/prices-history` for the offline study of prices at or above 90 cents; the §4 evidence stack plus official result pages (§6) for "resolved in substance" | later |
| **P4** liquidity rewards | CLOB `/rewards/markets/current`; Gamma reward fields; `/book` or WebSocket minute recorder; Data API `/v2/trades` for fill study | later |
| **P7** mention markets | xtracker JSON (resolution source of "# tweets" markets), YouTube and Twitch APIs for count markets (§5). **Gap: no source for speech transcripts has been researched** | later |
| **P8** skilled-wallet flow | Data API `/v2/leaderboard`, `/v2/holders`, `/v2/positions`, `/v2/user-pnl`, `/v2/trades` (§2) | later |

### Ops gates for every strategy

| Gate | Source | § | v1? |
|---|---|---|---|
| Geoblock refusal at start-up | `polymarket.com/api/geoblock` | 2 | yes |
| Clock skew and liveness | CLOB `/time`, `/ok` | 2 | yes |
| Data API freshness and schema drift | `/v2/status`, `/v2/openapi.json` | 2 | yes |

---

## 1b. Betting areas

The owner plans to bet wherever free data exists, so every Polymarket market is assigned to one of **29 areas** (research/09 §1: a full Gamma crawl on 2026-10-08, 22,441 active events and 268,110 active markets, about $40M of 24h volume; only about 1,370 events trade more than $1k a day). All areas are tagged and scored from Phase 1 (D-037); an area trades only through a strategy whose gate has passed. **Strategy** is taken from [STRATEGIES §Area coverage](STRATEGIES.md): P1 = LLM forecast on evidence, P6 = weather, P10 = sports models (backlog), P11 = esports (backlog), P12 = crypto price ladders (backlog), P7 = mention and post-count markets (backlog), **excluded** = insider-contaminated or single-decider markets. **Fee observed** is the taker rate k in `fee = shares x k x p x (1-p)` (worst case k/4 at p = 0.5; makers pay nothing and earn a rebate where stated); the live rate is always read per market from CLOB `/clob-markets/{condition_id}`, never hard-coded. Volumes are a 2026-10-08 snapshot.

**Per-area source tables** (endpoints, terms, tested status; 303 rows) stay in [research/09 §3](research/09-areas-and-sources.md); the keys are merged into §10. "Primary v1 sources" is the researchers' short list per area, for orientation only: Phase 2 runs on the §4 evidence stack, and an area's own adapters are built when its strategy starts (P1 areas lazily, in frequency order). **[NC]** = non-commercial or personal-use terms: **paper only until owner OK (D-038)**.

| Area | 24h volume | Fee observed | Strategy | Primary v1 sources | Keys needed |
|---|---|---|---|---|---|
| 1 NFL | $0.85M | 0.05; 0.03 at the NFL/CFB Oct-2026 rate | P10 | nflverse games, injuries and depth charts | none (`ODDS_API_KEY` to confirm odds; `GITHUB_TOKEN` optional) |
| 2 College football | $0.35M | 0.03 (Oct-2026 rate, most events); 0.05 | P10 | CFBD API | `CFBD_API_KEY` |
| 3 Basketball (NBA, WNBA, NCAA, Euroleague) | $0.30M | 0.05; 0.03 (v2) | P10 | hoopR NBA/MBB data; wehoop WNBA; NBA injury PDFs [NC]; CBBD | `CBBD_API_KEY`; `GITHUB_TOKEN` optional (file fetches) |
| 4 MLB, KBO, NPB | $0.51M | 0.05; 0.03 (v2) | P10 | MLB StatsAPI [NC] (KBO, NPB: no source) | none |
| 5 NHL and hockey | $0.56M | 0.05; 0.03 (v2) | P10 | NHL web API [NC]; NHL stats REST [NC] | none |
| 6 Soccer: top European leagues and cups | $1.77M | 0.05; 0.03 (v2); maker rebate 15% | P10 | football-data.co.uk (fixtures and season CSVs); openfootball; football-data.org | `FOOTBALL_DATA_ORG_TOKEN` |
| 7 Soccer: international | $0.04M | 0.05 | P10 | martj42 international_results; openfootball worldcup.json | none |
| 8 Soccer: rest of world | $1.08M | 0.05; 0.03 (v2) | P10 | American Soccer Analysis API; football-data.co.uk `USA.csv` [NC] | none |
| 9 Tennis | $2.14M | 0.05; 0.03 (v2) | P10 | none (gap); later Tennis Abstract Elo [NC], Sackmann mirror [NC] | none (`ODDS_API_KEY` for Grand Slam odds) |
| 10 Golf | $0.01M | 0.05; 0.03 (v2) | P10 | none; later OWGR API, Slash Golf, ESPN | none (`SLASHGOLF_RAPIDAPI_KEY` later) |
| 11 Motorsport (F1, NASCAR, IndyCar, MotoGP) | $0.06M | 0.05; 0.03 (v2) | P10 (F1; other series thin) | Jolpica-F1 [NC] | none |
| 12 Combat sports | $0.03M | 0.05; 0.03 (v2) | P10 (MMA; boxing and WWE thin) | none (The Odds API, §8); later Greco1899 CSV, ESPN, TheSportsDB | `ODDS_API_KEY` |
| 13 Esports | $6.30M | 0.05; 0.03 (v2, 335 of 736 events) | P11 | OpenDota; Valve standings; PandaScore; Liquipedia MediaWiki; Leaguepedia Cargo | `PANDASCORE_TOKEN`, `LEAGUEPEDIA_BOT_PASSWORD` |
| 14 Cricket | $0.04M | 0.05; 0.03 (v2) | P10 | Cricsheet; CricketData.org | `CRICAPI_KEY` |
| 15 Other sports (chess, table tennis, pickleball, darts, rugby, cycling, poker) | $0.02M | 0.05; 0.03 (v2) | P1 (assumed: not named in D-037; thin) | Lichess API; FIDE rating lists; chess.com Published-Data API | none |
| 16 US elections | $4.66M | 0.04; maker rebate 25% | P1 | VoteHub polls; Silver Bulletin generic ballot; Wikipedia tables; Kalshi and PredictIt [NC] (§8) | none |
| 17 US politics and government | $0.19M | 0.04; 0.05 culture-tagged; 13 events fee-free | P1 | Silver Bulletin approval; White House RSS; Federal Register; House Clerk XML; Voteview; SCOTUS docket JSON; DOJ RSS | `DATA_GOV_API_KEY` |
| 18 World elections and leaders | $2.01M | 0.04 | P1 | DAWUM; UK Parliament APIs; TSE Dados Abertos; Wikipedia | none |
| 19 Geopolitics and conflict | $4.29M | **0** on 368 of 522 events; 0.04 on 144 | P1 | none new (news stack, §4); gap: ISW map, DeepState, IMF PortWatch untested | none |
| 20 Economy, Fed, central banks | $1.88M | 0.05 economics; 0.04 politics | P1 | SoSoValue (licence unpublished; paper only until owner OK, D-038); Atlanta Fed MPT; NY Fed API; Treasury bills; Fed RSS; BoE, BoC, BoJ APIs | `FRED_API_KEY`; `SOSOVALUE_API_KEY` optional |
| 21 Crypto (ladders, launches, listings) | $4.61M | **0.07** (highest tier); maker rebate 20% | P12 (ladders and hit-price); 5M/15M Up/Down rejected (STRATEGIES) | Binance; Coinbase; Kraken; Deribit; exchangeInfo and products diff; Blockstream Esplora; DefiLlama | none (`ETHERSCAN_API_KEY` at P12 start) |
| 22 Stocks, commodities, forex, IPOs | $0.70M | 0.04 finance; 0.05 economics | P1 (finance events) | Finnhub [NC]; Twelve Data [NC]; Alpha Vantage [NC]; Cboe VIX CSVs; EIA; Frankfurter | `FINNHUB_API_KEY`, `TWELVEDATA_API_KEY`, `ALPHAVANTAGE_API_KEY`, `EIA_API_KEY` (all optional) |
| 23 Tech and AI | $0.95M | 0.04; 0.05 culture-tagged | P1 | LMArena HF dataset; HF Hub; OpenRouter models; Artificial Analysis; Epoch AI; GitHub; lab newsroom RSS | `ARTIFICIAL_ANALYSIS_API_KEY`, `GITHUB_TOKEN`, `HF_TOKEN` (all optional) |
| 24 Culture and entertainment | $0.38M | 0.05; 0.04 politics | P1 (culture long tail) | Netflix TSVs; Apple chart RSS; Wikipedia lists; trade-press RSS; TVmaze; Smarkets (§8); Steam keyless endpoints | `NYT_API_KEY` optional |
| 25 Awards and prizes | $1.21M | 0.05 | P1; **Nobel Peace before the announcement and Google Year in Search are excluded** (insider cases) | Nobel Prize API; Smarkets (§8); Wikipedia | none |
| 26 Weather and natural disasters | $1.77M | 0.05; maker rebate 25%; 10 events fee-free | P6 | Open-Meteo [NC] and NWS (§7); USGS quakes; NHC; GDACS; GISTEMP; NCEI; Climate Pulse ERA5; NSIDC; CPC ENSO | none (P6 v1 is keyless, §7) |
| 27 Science and space | $0.16M | 0.04; 0.05 culture-tagged | P1 | none new (Launch Library 2 and NASA DONKI, §6.4) | none |
| 28 Health and pandemics | $0.10M | 0.05 general; 0.04 | P1 | data.cdc.gov (Socrata); openFDA; FDA press RSS | `SOCRATA_APP_TOKEN`, `OPENFDA_API_KEY` (both optional) |
| 29 Mentions and social-post counts | $3.07M | 0.05 culture-tagged; 0.04 mentions | P7 | XTracker (§5); CNN Truth Social archive; White House RSS; govinfo CPD RSS; Alpha Vantage transcripts [NC]; Hansard | `DATA_GOV_API_KEY`, `ALPHAVANTAGE_API_KEY` |

Notes. (1) Area 15 is not named in D-037: P1 is assumed because its markets are illiquid micro-markets (about $0.02M/24h); a chess rating model would belong to P10. (2) Crypto 5M/15M Up/Down (about 3,140 of 3,940 crypto events) is rejected in STRATEGIES, so area 21 means ladders and hit-price markets only. (3) Geopolitics has the second-largest lifetime volume and **no free data source tested**; tennis has no current free results feed (research/09 §2). P1 leans on the §4 news stack for both. (4) Keys: none in Phase 1; Phase 2 needs only the LLM and evidence keys in §10; every key named above is optional until its strategy starts.

---

## 2. Polymarket venue data

**All 20 rows were live-tested on 2026-10-08. No key, signup or state-changing call is needed for anything in this section. Phase 1 needs no key.**

¹ **Polymarket ToS** (eff. 2026-08-11; `polymarket.com/tos` is an embedded Google Doc). The Data clause bars non-retail professional capital-markets entities (prop firms, market makers, fintechs, hedge funds) and market-data distributors from using Polymarket data, raw, derived or aggregated, without a written agreement, and bars reselling it to them. **The owner must run PYTHIA as an individual retail user and must not resell or redistribute the data.** Do not incorporate the operation.

**Gotchas verified live (all must become contract tests, BUILD_PLAN 1.4):**
1. Data API v2 filters by `condition`, not `market` (`market=` returns 400).
2. Bare `volume` and `size` are **shares**; fields ending `_usdc` are USD (one wallet: volume 2,690,025 shares vs `volume_usdc` 144,935).
3. CLOB `/price?side=BUY` returns the best **bid**, `SELL` the best **ask**.
4. In `/book`, bids ascend and asks descend: best bid and best ask are the **last** array elements. The book-level `last_trade_price` was 0.993 for both the Yes and No token, so it is wrong; do not use it.
5. Gamma `clobTokenIds`, `outcomes` and `outcomePrices` are JSON-encoded **strings** (decode twice).
6. Weather market descriptions name the NOAA or HKO station (used by P6, §7).

Not verified: the geoblock result from a remote IP (owner or isolated server step); WebSocket events beyond book snapshots (the test market was quiet). The legacy bulk lists CLOB `/markets`, `/simplified-markets` and `/sampling-markets` return 1,000 rows (0.5-2.5 MB) and ignore small limits: **do not use them for discovery**.

| Source | Use in decision | Endpoint (example) | Auth | Free limits | Terms | Tested | Priority |
|---|---|---|---|---|---|---|---|
| Gamma: markets and events (keyset) | Candidate selection, rules text for the LLM, fees, liquidity, `negRisk`, `umaResolutionStatuses`, reward fields, and the token ids every CLOB and Data call needs | `GET https://gamma-api.polymarket.com/markets/keyset?closed=false&limit=100&liquidity_num_min=20000&volume_num_min=50000&end_date_min=2026-10-09T00:00:00Z&end_date_max=2026-10-20T00:00:00Z&order=volume24hr&ascending=false`; also `/events/keyset`, `/markets/slug/{slug}`, `/events/slug/{slug}` | none | Per IP, throttled (queued), not rejected: general 4,000/10s, `/markets` 300/10s, `/events` 500/10s. We need ~100 calls/day | ¹ | 200 ✓ (0.6-0.9s) | v1 |
| Gamma: tags, series, sports, teams | Category routing (tag ids such as geopolitics 100265, `weather`); series groups. Sports and teams unused | `GET https://gamma-api.polymarket.com/tags/slug/geopolitics`; `/series?limit=2` | none | `/tags` 200/10s; general 4,000/10s | ¹ | 200 ✓ (sports 150 KB, series 52 KB: cache daily) | v1 |
| Gamma: public-search | Ad-hoc keyword match of a topic to existing markets | `GET https://gamma-api.polymarket.com/public-search?q=nobel%20peace&limit_per_type=2` | none | 350/10s | ¹ | 200 ✓ (slow: 3.7s, 315 KB for 2 events) | later |
| Gamma: comments, public-profile | Crowd chatter. Not a probability input; leaks crowd opinion into a price-blind pass and is the strongest prompt-injection vector | `GET https://gamma-api.polymarket.com/comments?parent_entity_type=Event&parent_entity_id=60182&limit=2` | none | 200/10s | ¹ Third-party user text with wallet addresses: do not feed to prompts or store | 200 ✓ (parent id 1 returns `[]`) | later |
| CLOB: order book | Executable price, depth and slippage for sizing and the paper fill model; liquidity filter on the real book, not Gamma's number | `GET https://clob.polymarket.com/book?token_id=<token>`; `POST /books` (up to 500 tokens) | none | `/book` 1,500/10s; `/books` 500/10s; CLOB market data 9,000/10s | ¹ | 200 ✓ (~0.5s) | v1 |
| CLOB: prices (price, midpoint, spread, last-trade-price, batch POSTs) | Market probability for the blend and edge-after-cost; one `POST /midpoints` snapshots up to 500 tokens | `GET https://clob.polymarket.com/midpoint?token_id=<token>`; `POST /midpoints`, `/prices` (body `{token_id, side}`), `/spreads`, `/last-trades-prices` | none | `/price`, `/midpoint` 1,500/10s; batch 500/10s | ¹ | 200 ✓ | v1 |
| CLOB: market parameters | Tick size, min order size (5), neg-risk flag, fee schedule, accepting-orders flag, reward params. Avoids hard-coding values that changed several times in 2026 | `GET https://clob.polymarket.com/clob-markets/{condition_id}`; `/tick-size`, `/neg-risk`, `/fee-rate` (all take a token id or condition id) | none | `/tick-size` 200/10s | ¹ | 200 ✓ | v1 |
| CLOB: prices-history (legacy) | Fallback for the price path. Query param is `market` but its value is the **token** id | `GET https://clob.polymarket.com/prices-history?market=<token>&interval=1w&fidelity=60` | none | 1,000/10s | ¹ | 200 ✓ | later |
| CLOB: liquidity rewards | Which markets pay maker rewards, daily rate, min size, max spread (P4 screen) | `GET https://clob.polymarket.com/rewards/markets/current` | none | Within CLOB 9,000/10s | ¹ | 200 ✓ (1.7s, 175 KB, 500 rows per page) | later (researcher said v1; P4 is measure-first) |
| Geoblock check | Check eligibility at startup and daily; refuse new entries when blocked or eligibility is unknown | `GET https://polymarket.com/api/geoblock` | none | None published | Do not evade venue restrictions | Historical check; verify in the intended environment | v1 |
| CLOB: server time, health | Compare the local system clock with the venue server time; liveness gate before an order batch | `GET https://clob.polymarket.com/time` (unix seconds); `/ok` | none | `/ok` 100/10s | ¹ | 200 ✓ | v1 |
| CLOB: market WebSocket | Streaming book and price changes for open positions, P3 episode timing, P4 recorder. Subscribe key is spelled `assets_ids`; send text `PING` about every 10s | `wss://ws-subscriptions-clob.polymarket.com/ws/market` | none | No connection cap found | ¹ | 101 ✓ (2 book snapshots and 1 PONG in 10s on a quiet market) | v1 (Phase 1 per BUILD_PLAN 1.4; researcher said later) |
| Sports WebSocket | Live scores for sports markets, which we do not trade | `wss://sports-api.polymarket.com/ws` | none | Server pings every 5s; reply within 10s | ¹ | 101 ✓ | later |
| Data API v2: prices-history | Bucketed price history per token for drift, news-reaction speed, backtests | `GET https://data-api.polymarket.com/v2/prices-history?token_id=<token>&interval=1w&bucket_seconds=86400` | none | 200/10s; v2 general 800/10s; explicit windows cap at 15 days, `limit` up to 10,000 | ¹ | 200 ✓ (0.5s) | v1 |
| Data API v2: trades | Recent fills (size in shares, price, side, wallet): realised flow and stale-book detection. Rows carry pseudonymous wallet, name and bio: keep aggregates only | `GET https://data-api.polymarket.com/v2/trades?condition=<condition_id>&limit=3` | none | 300/10s; v2 general 800/10s | ¹ | 200 ✓ (`market=` returns 400) | v1 (Phase 1 per BUILD_PLAN 1.4; researcher said later) |
| Data API v2: market stats (oi, holders, positions, live-volume) | Open interest as a crowding filter; holder concentration for manipulation and resolution-risk checks | `GET https://data-api.polymarket.com/v2/oi?condition=<condition_id>`; `/v2/holders`, `/v2/positions`, `/v2/live-volume?event_id=` | none | `/v2/positions` 200/10s; general 800/10s | ¹ Pseudonymous wallets | 200 ✓ (OI example: $3.01M) | later |
| Data API v2: resolutions, token lookup | UMA state (posed, proposed, disputed, proposed price) for the resolution-risk guard; token to condition, outcome and opposite token | `GET https://data-api.polymarket.com/v2/resolutions?condition=<condition_id>`; `/v2/tokens?condition=` | none | General 800/10s | ¹ | 200 ✓ (needs exactly one of `question_id`, `condition`, `event_id`) | v1 |
| Data API v2: wallet analytics | Smart-money research (P8) and our own PnL and value reconciliation: `leaderboard`, `biggest-winners`, `user-pnl`, `user-stats`, `user-volume`, `value`, `activity`, `approvals` | `GET https://data-api.polymarket.com/v2/user-volume?user=<our proxy wallet>`; `/v2/leaderboard?time_period=week&limit=3` | none | General 800/10s | ¹ Third-party wallets: aggregate only, never publish per-wallet profiles | 200 ✓ (`value`, `activity` need `user`) | later (reconciliation endpoints become useful at micro-live) |
| Data API v2: status, OpenAPI | Freshness gate (`lag_seconds`) before trusting Data API numbers; `openapi.json` for schema-drift checks in CI (the SDK broke Data API shapes at 0.10.0) | `GET https://data-api.polymarket.com/v2/status`; `/v2/openapi.json` (156 KB, 20 paths) | none | General 800/10s; health 100/10s | ¹ | 200 ✓ (lag 2s) | v1 |
| Bridge API: supported-assets | Funding step only: which chains and tokens can deposit, minimum checkout USD (Polygon USDC: 2) | `GET https://bridge.polymarket.com/supported-assets` | none | 50/10s | No Polymarket deposit fee; network costs apply | 200 ✓ (236 assets, 41 KB) | later (used once by the owner) |


---

## 3. LLM providers (forecaster roles)

**Rule (D-024):** a new Google AI Studio project plus Mistral and OpenRouter keys for this bot only; never reuse ARGUS keys; the Gemini free project is never linked to billing (that converts it to paid). **Roster by role (D-036, provisional):** samples A and B on the Gemini free tier direct; **sample C and the short roles (`parse_rules`, `query_writer`, `json_repair`) on the Vercel AI Gateway free credit**; Mistral free, OpenRouter `:free` and Groq are backups; the paid chain (D-015) comes last. Free limits drift silently: week-1 measurement on our keys is BUILD_PLAN task 2.3, and model ids live in `config/models.yaml`, never in code.

**⚠️ on LLM rows.** The retest flagged these only because a keyless GET to a POST endpoint returns 401 or 404, which shows the host answers but does not prove the model id, the path or the quota. Verify each with one keyed POST in Phase 2 task 2.3. The Vercel AI Gateway and Jev rows were tested on 2026-10-08 with the owner's own keys (one run each, no independent retest) and are the only keyed proofs so far. Gemini's path form is `models/<id>:generateContent` (colon); the researcher's URL used a slash, which explains its 404.

| Source | Use in decision | Endpoint (example) | Auth | Free limits | Terms | Tested | Priority |
|---|---|---|---|---|---|---|---|
| Google Gemini API (AI Studio, free project) | `forecast_A` (3.8 Flash), `forecast_B` (3.5 Flash-Lite), rule parsing, free URL-context tool | `POST https://generativelanguage.googleapis.com/v1beta/models/gemini-3.5-flash-lite:generateContent` | free key `GEMINI_API_KEY_FREE` | 3.5 Flash-Lite: 15 RPM, 500 RPD. 3.8 Flash: ~20 RPD (claimed). Per project; resets midnight Pacific; TPM unknown (measure). A retest note said "1,000/day": research/06 and ARGUS say 500, use 500 | Free-tier content is used to improve products and may be read by humans; low risk, we serve no users. Search grounding is not available on 3.x free | GET 404 ⚠️ (wrong path form plus GET; unproven until a keyed POST) | v1 |
| Vercel AI Gateway (free-tier models, $5/month credit) | **Forecast sample C** (D-036): `spacexai/grok-4.1-fast-non-reasoning` (xAI, 1M context, $0.20 in / $0.50 out per MTok). **Short roles**: `openai/gpt-5-nano` ($0.05 / $0.40; strict structured output). Backups: `deepseek/deepseek-v3.1` (sample C), `alibaba/qwen3.7-flash` (short roles), `google/gemini-2.5-flash-lite` (Google family when the direct free tier is exhausted). Paper-only diversity ablation (K = 3 vs 6): `alibaba/qwen3.5-flash`, `zai/glm-4.7-flash`, `deepseek/deepseek-v3.1`; the sample C choice among Grok, DeepSeek and `openai/gpt-5-mini` is decided in paper, not assumed | `GET https://ai-gateway.vercel.sh/v1/models` and `/v1/credits` (historical authenticated response); chat completions on the same OpenAI-compatible `/v1` base (path per Vercel docs; confirm in task 2.3) | account key `AI_GATEWAY_API_KEY` (configuration required); Vercel Hobby plan, **card on file required** (verification only; no purchase) | $5/month credit, renewing, starts with the first request. **140 of 267** language models usable; the newest generation (gpt-6-luna, Gemini 3.x, DeepSeek V4.x, Kimi K3) and Jev are excluded. Free-tier calls bill at list price against the $5 (derived: sample C on ~15 markets/day about $1.69/month on Grok; short roles about $0.36/month on gpt-5-nano). **Never buy Vercel credits**: a purchase ends the free tier | Vercel forwards prompts to each model provider: send only public market text and news (the rule for every LLM call) | **2026-10-08 historical test:** one 16-token call to each of 267 models: 140 returned 200, 124 returned 403 ("Free tier users do not have access to this model"), 3 other errors; the probe cost $0.0176. **The roster can change without notice**: re-probe monthly (`pythia doctor --llm`) (research/12) | v1 (D-036: sample C primary, short roles) |
| Mistral API (free mode) | `forecast_C` backup after Vercel Grok (D-036), the non-Google family | `POST https://api.mistral.ai/v1/chat/completions` | free key `MISTRAL_API_KEY` | Not published; ARGUS measured 6 RPM and 200 RPD. A retest note said "100/day": measure on our key | Free mode: inputs and outputs may be used for training | 401 ⚠️ (host live; key needed) | v1 (backup since D-036) |
| OpenRouter (`:free` models) | Second and third family; fallback for `forecast_C`. The model list is public | `GET https://openrouter.ai/api/v1/models` (468 models); `POST /api/v1/chat/completions` | free key `OPENROUTER_API_KEY` | 20 RPM. 50 RPD, or 1,000 RPD after ≥$10 lifetime purchases. Roster rotates (ARGUS: 6 of 16 tool-capable free models answered). A balance ≤ 0 returns 402 even on free models | Free models may train on data; per-model opt-out removes providers. Card fee 5.5% on top-ups | 200 ✓ (model list) | v1 |
| Groq (free plan) | Short tasks only: `parse_rules`, `json_repair` (≤ 6.5k tokens); backup to Vercel `gpt-5-nano` since D-036 | `POST https://api.groq.com/openai/v1/chat/completions` | free key `GROQ_API_KEY` | 30 RPM, 1K RPD, 8K TPM, 200K TPD per model. One request above the TPM cap returns 413 | No training on customer data; zero-retention toggle | GET 404 ⚠️ (POST-only; unproven) | v1 (short tasks; researcher said later) |
| Cloudflare Workers AI | Overflow family: `@cf/openai/gpt-oss-120b` is ~17 calls/day at our call shape | `POST https://api.cloudflare.com/client/v4/accounts/{account_id}/ai/run/@cf/openai/gpt-oss-120b` | free key (account id and token) | 10,000 neurons/day, resets 00:00 UTC | Cloudflare terms | Not tested this round (docs, research/06) | later |
| NVIDIA NIM (build.nvidia.com) | Paper-phase research ensemble only (Kimi K3, GLM-5.3, DeepSeek V4.1 Flash) | `GET https://integrate.api.nvidia.com/v1/models` (public list, 80 models) | free key `NVIDIA_API_KEY` | 40 RPM | Trial is "only for internal testing and evaluation, not in production": live trading is a terms risk, so never trade on it | List 200 per research/06; not re-tested | later |
| DeepSeek `deepseek-flash` (paid fallback) | `forecast_C` fallback; `query_writer` fallback; use off-peak when possible | `POST https://api.deepseek.com/chat/completions` | paid prepay `DEEPSEEK_API_KEY` | None. $0.30 in (cache hit $0.006) and $1.20 out per MTok peak; half price off-peak (peak = 01-04 and 06-10 UTC, Mon-Fri) | Inputs stored in China; training on by default with opt-out (secondary source) | Docs only (research/10) | v1 (fallback, under the $5/month cap, D-015) |
| OpenAI `gpt-6-luna` (paid fallback) | `rules_parse` with strict schema, `json_repair`, `query_writer`; Batch for non-urgent re-forecasts | `POST https://api.openai.com/v1/chat/completions` | paid prepay `OPENAI_API_KEY` | None. $0.10 in (cache $0.01), $0.50 out per MTok; Batch -50% | $5 minimum prepay, credits expire after 1 year; no training on API data by default | Docs only (research/10) | v1 (fallback, same cap) |
| OpenRouter paid models | Last link of the paid chain (`qwen3.8-flash`, `glm-5.3-flash`, `mistral-large-4-0`) | Same endpoint and key as OpenRouter above | paid (same key) | None; no inference markup | Same | Docs only (research/10) | later |

**Chosen v1 router (research/06 §4.1, research/10, research/12; D-036; every limit enforced in code).** Order inside each chain: direct free tiers, then the Vercel free credit, then paid. Chains and model ids live in `config/models.yaml`, never in code.

| Role | Chain, free first |
|---|---|
| `parse_rules` | Vercel `openai/gpt-5-nano` → Groq `gpt-oss-120b` → Vercel `alibaba/qwen3.7-flash` → Gemini 3.5 Flash-Lite → Mistral Small → (paid) `gpt-6-luna` → `deepseek-flash` |
| `forecast_A` | Gemini 3.8 Flash → 3.7 Flash → 3.5 Flash-Lite → Vercel `google/gemini-2.5-flash-lite` |
| `forecast_B` | Gemini 3.5 Flash-Lite → Vercel `google/gemini-2.5-flash-lite` |
| `forecast_C` (must be non-Google) | **Vercel `spacexai/grok-4.1-fast-non-reasoning`** → Mistral Large → OpenRouter `nvidia/nemotron-3-ultra-550b-a55b:free` → Vercel `deepseek/deepseek-v3.1` → Cloudflare `gpt-oss-120b` (if its key exists) → (paid) `deepseek-flash` → `gpt-6-luna` → OpenRouter `qwen3.8-flash` |
| `json_repair` | Vercel `openai/gpt-5-nano` → Groq `gpt-oss-20b` → Vercel `alibaba/qwen3.7-flash` → Gemini 3.5 Flash-Lite → (paid) `gpt-6-luna` → `deepseek-flash` |
| `query_writer` | Vercel `openai/gpt-5-nano` → Groq (short prompts) → Vercel `alibaba/qwen3.7-flash` → Gemini 3.5 Flash-Lite → (paid) `gpt-6-luna` → `deepseek-flash` |
| `research_ensemble` (paper only, never traded) | NVIDIA NIM `kimi-k3`, `glm-5.3`, `deepseek-v4.1-flash`; diversity ablation (K = 3 vs 6): Vercel `alibaba/qwen3.5-flash`, `zai/glm-4.7-flash`, `deepseek/deepseek-v3.1` |

**Paid fallback chain (research/10; D-015).** It starts only after the free and Vercel links fail or hit a cap. Prices per MTok: `deepseek-flash` $0.30 in (cache hit $0.006) and $1.20 out at peak, half price off-peak; `gpt-6-luna` $0.10 in (cache $0.01) and $0.50 out, Batch -50%; OpenRouter `qwen3.8-flash` $0.15 / $0.47.

| Role | Paid tail |
|---|---|
| `forecast_C` | `deepseek-flash` (thinking off or low; off-peak when possible) → `gpt-6-luna` (low effort) → OpenRouter `qwen3.8-flash` or `mistral-large-4-0` |
| `parse_rules` | `gpt-6-luna` (strict schema) → `deepseek-flash` (`json_object` plus validation) |
| `query_writer`, `json_repair` | `gpt-6-luna` → `deepseek-flash` (thinking off) |
| Non-urgent re-forecasts | `gpt-6-luna` Batch (within 24 h) |

Paid caps: **$5/month in total, binding** (sub-caps DeepSeek $2, OpenAI $1.5, OpenRouter $1, TypeSafe $0.5; keys in RISK §2) plus a daily cap of about $0.33, checked before every call against the worst case (input estimate + `max_tokens` x price; `max_tokens` is always set); no auto-recharge; resets on the 1st (UTC). A paid model fills at most one sample slot, and every forecast keeps at least one free sample from a different family, otherwise `NO_FORECAST`. On the cap: stop paid calls, fall back to free, stop new entries if forecasting stops, keep managing positions, alert once. Cheap paid models are availability insurance, not better forecasters (Prophet Arena, 2026-10-05: DeepSeek V4.1 Flash 0.9462 and GPT-6 Luna 0.9453 against the Kalshi market's 0.9458). Derived cost at 100 calls/day: 10% paid is about $0.40 (Luna) to $0.99 (DeepSeek peak) per month; 100% paid is about $4.01 to $9.91. If D-035 is approved, Jev's `jev_usd_cap_month` of $2 sits inside the same cap.

Rules: sample C never shares a family with A or B (a Google model reached through Vercel is still the Google family); if no non-Google family is reachable the result is `NO_FORECAST`. Skip a provider at 90% of its RPD or TPD (for Vercel: 90% of the $5 monthly credit, read from `/v1/credits`); circuit-break after 3 consecutive 429 or 5xx. Paid spend is checked before every call against the D-015 cap ($5/month: DeepSeek $3, OpenAI $2, OpenRouter $1). Planned free capacity is ~95-135 long-context calls/day against the `llm_daily_call_ceiling` in RISK §2 (120).

---

## 4. Evidence: news discovery, web search, page reading

**Chosen v1 stack**

| Step | v1 choice | Why |
|---|---|---|
| Discovery (3) | **Google News RSS** (titles, publishers, dates only), **GDELT DOC 2.0** (best effort), **Wikipedia Current events** | Keyless; Google gives breadth and a Russian locale; GDELT gives direct publisher URLs and language metadata; Wikipedia gives a neutral cited digest |
| Search (2, quota-rotated) | **Exa**, then **Tavily**. Brave only if the owner accepts a card on file | Both free with no card. 45 queries/day is ~1,350/month against ~3,500/month combined (≈2.6x headroom) |
| Reader (1) | **Jina Reader, keyless** (20 RPM). Fallbacks: Gemini URL-context tool, Firecrawl | No key and no card at ~75 reads/day |

**Caveats to carry into the build**
1. Google News RSS item links are opaque `news.google.com/rss/articles/...` URLs that redirect to `consent.google.com` from an EU egress, and Jina blocks `news.google.com`. Use RSS **only** for titles, sources and dates, then fetch text through Exa or Tavily result URLs. Keep the feed behind a replaceable interface: its terms say personal, non-commercial use only, so the feed is **paper only until owner OK (D-038)**; if the owner declines, discover through GDELT, Wikipedia Current events and Exa or Tavily.
2. GDELT DOC returned **429 on about half of ~14 calls even at 8-40 s spacing**; the index lagged ~8 hours; a Cyrillic query returned `{}` three times. Build a serial queue (≥ 6 s spacing, backoff) and never block a forecast on it.
3. Research/06 estimated 10M Jina tokens last 2-3 months. At 75 reads/day x 5k tokens that is ~27 days. Keyless Reader avoids the quota.
4. Exa and Firecrawl are POST-only. The GET tests prove the host is live but give no auth-error proof (⚠️).
5. Commercial-use wording is unclear or restrictive for NewsAPI, GNews, Mediastack, Event Registry and the Guardian developer key (see §9 and the table).
6. Wikimedia requires a User-Agent with contact info; the owner must choose it (`PYTHIA_USER_AGENT`, §10). A generic `python-requests` UA got HTTP 403.
7. Every fetched string (RSS, snippets, pages, Wikipedia, social) is **untrusted**: scrub, wrap and flag per ARCHITECTURE §15 (PM2) and BUILD_PLAN 2.1.

### 4.1 Discovery

| Source | Use in decision | Endpoint (example) | Auth | Free limits | Terms | Tested | Priority |
|---|---|---|---|---|---|---|---|
| Google News RSS (search and topic feeds) | Breadth-first headlines per market (~80-100 items/query); headlines pick the search queries | `GET https://news.google.com/rss/search?q=ceasefire+when:1d&hl=en-US&gl=US&ceid=US:en` (Russian: `hl=ru&gl=RU&ceid=RU:ru`) | none | None published; ≤ 1 request per market per hour, cache | Feed copyright: personal, non-commercial use only; not an official API: **paper only until owner OK (D-038)** | 200 ✓ (EN 83 items, RU 79) | v1 (discovery only) |
| GDELT DOC 2.0 (`artlist`, `timelinevol`, `timelinetone`) | Direct publisher URLs, domain, language, source country; `timelinevol` shows coverage spikes | `GET https://api.gdeltproject.org/api/v2/doc/doc?query=Iran%20sourcelang:english&mode=artlist&maxrecords=25&format=json&timespan=1d&sort=datedesc` | none | 1 request per 5 s documented; responses cached 900 s; ~3-month window | Open data; throttle text asks heavy users to move to bulk data | 200, retest **429** ⚠️ (rate limited; works with spacing; ~half of calls throttled; ~8 h index lag; empty `{}` on Cyrillic; empty `timelinevol` for "ceasefire") | v1 (best effort) |
| GDELT Context 2.0 | Sentence-level snippets around the query phrase; fresher than DOC at the time | `GET https://api.gdeltproject.org/api/v2/context/context?query=%22ceasefire%22&mode=artlist&format=json&maxrecords=2&timespan=1d` | none | Same host and throttle as DOC | Open data | 200, retest **timeout** ⚠️ | later |
| Wikipedia Current events portal | Neutral daily digest with citations: check that an event happened; catch big stories the LLM may not know. Fetch yesterday and earlier (today's page is partial). Page names: `Portal:Current_events/2026_October_7` | `GET https://en.wikipedia.org/w/api.php?action=parse&page=Portal:Current_events/2026_October_7&prop=wikitext&format=json&formatversion=2` | none (contact User-Agent required) | 10 req/min unidentified; 200/min with a contact UA; 2,000/min authenticated. We need ~7 calls/day | CC BY-SA 4.0; lag of hours; a lead, never a resolution source | 200 ✓ (parse 21 KB; REST `feed/featured/2026/10/08` 302 KB with 5 "in the news" items) | v1 |
| Publisher RSS (BBC World, Guardian World, Al Jazeera) | Fast confirmation of breaking events | `GET https://feeds.bbci.co.uk/news/world/rss.xml`; `https://www.theguardian.com/world/rss`; `https://www.aljazeera.com/xml/rss/all.xml` | none | None; poll ≤ every 10-15 min | Per publisher; BBC: personal or non-commercial with accreditation, so the BBC feed is **paper only until owner OK (D-038)** | 200 ✓ (31, 45, 25 items) | later |
| The Guardian Open Platform | Full-text UK and world articles (`show-fields=body`) | `GET https://content.guardianapis.com/search?q=ceasefire&api-key=<key>` | free key `GUARDIAN_API_KEY` | Developer key: 1 call/s, 500/day | **Developer key is non-commercial only**; commercial key needs a separate request: **paper only until owner OK (D-038)** | 401 ✓ (live; "No API key found") | later (needs owner to treat PYTHIA as personal research) |
| New York Times Article Search | Headlines, abstracts and URLs; no full text | `GET https://api.nytimes.com/svc/search/v2/articlesearch.json?q=ceasefire&api-key=<key>` | free key `NYT_API_KEY` | **Unverified** (third parties say 10/min and 4,000/day; older docs said 500/day) | Not verified; read NYT API terms | 401 ✓ (live) | later |
| NewsData.io | Non-English and regional coverage via `/latest` | `GET https://newsdata.io/api/1/latest?q=ceasefire&apikey=<key>` | free key `NEWSDATA_API_KEY` | 200 credits/day, 12-hour delay, no card; archive is paid | Free-plan commercial use **not confirmed** | 401 ✓ (live) | later |
| Currents API | Possible supplement | `GET https://api.currentsapi.services/v1/search?keywords=ceasefire&apiKey=<key>` | free key `CURRENTS_API_KEY` | 250 requests/day | Attribution required; no customer-facing AI summaries, republication or long-term archival; prototypes and evaluation | 401 ✓ (live) | later |

### 4.2 Web search

| Source | Use in decision | Endpoint (example) | Auth | Free limits | Terms | Tested | Priority |
|---|---|---|---|---|---|---|---|
| Exa Search API | **Primary evidence search.** News category with date filters; returns publisher URLs and optional text or highlights | `POST https://api.exa.ai/search` with header `x-api-key` | free key `EXA_API_KEY` | $10 free credit per month plus a one-time $10 onboarding bonus, no card (≈2,500 searches/month at $4 per 1k). 10 QPS search, 100 QPS contents. Contents $1 per 1k pages | Standard commercial terms; data-retention terms not reviewed | GET 404 ⚠️ (`NOT_FOUND`: host live, POST-only, no auth proof) | v1 |
| Tavily Search API | **Second search** and fallback; `topic=news` with a time range returns extracted content | `POST https://api.tavily.com/search` with `Authorization: Bearer <key>` | free key `TAVILY_API_KEY` | 1,000 credits/month, no card; dev key 100 RPM. Basic search 1 credit, advanced 2 (from research/06) | Standard commercial terms; not reviewed in detail | 401 ✓ ("missing or invalid API key") | v1 |
| Brave Search API | Optional third, independent web index when Exa and Tavily run out | `GET https://api.search.brave.com/res/v1/web/search?q=ceasefire` with header `X-Subscription-Token` | free key (**card required**) `BRAVE_API_KEY` | $5 credit/month (≈1,000 requests); 50 QPS; overages billed with **no spending cap** | Terms bar storing results or training or tuning an LLM unless on a plan with storage rights | 422 ✓ (shows the required header; `/news/search` same) | later (owner approval for a card) |
| Serper.dev (Google SERP) | Emergency reserve for Google-quality results | `POST https://google.serper.dev/search` with header `X-API-KEY` | free key `SERPER_API_KEY` | 2,500 free queries, **one time**, no card | Scrapes Google (unofficial); terms not reviewed | 403 ✓ ("Unauthorized. Sign up") | later |
| Jina Search (`s.jina.ai`) | Optional extra search with contents attached | `GET https://s.jina.ai/ceasefire` with `Authorization: Bearer <key>` | free key `JINA_API_KEY` | 100 RPM; each request costs ≥ 10,000 tokens, so the one-time 10M free tokens give ≤ ~1,000 searches | Standard Jina terms | 401 ✓ (`AuthenticationRequiredError`) | later |
| SearXNG (self-hosted metasearch) | Keyless unlimited fallback on the isolated server | Self-host; add `json` to `search.formats` in `settings.yml` | none | No quota of its own; upstream engines may block the isolated server IP | AGPL software; queries Google/Bing/DDG HTML upstream (unofficial) | Public instance 200 but **HTML only** (JSON disabled) | later |

### 4.3 Page reading

| Source | Use in decision | Endpoint (example) | Auth | Free limits | Terms | Tested | Priority |
|---|---|---|---|---|---|---|---|
| Jina Reader (`r.jina.ai`) | Turns an article URL into clean Markdown (title, published time, body) for the prompt | `GET https://r.jina.ai/https://www.bbc.co.uk/news/articles/cmx2qlz7qrd4o` | none (optional `JINA_API_KEY`) | 20 RPM keyless (`x-ratelimit-limit: 20`); 500 RPM with a free key; a new key includes 10M tokens, one time | Public pages only; hosted proxy. Send `x-no-cache: true` when freshness matters. `news.google.com` links return 403 | 200 ✓ (BBC article 8.8 KB; Wikipedia 47 KB) | v1 |
| Firecrawl (`/scrape`) | Fallback for JS-heavy pages Jina cannot parse | `POST https://api.firecrawl.dev/v1/scrape` | free key `FIRECRAWL_API_KEY` | Pricing page shows 1,000 credits/month, no card, 10 requests/min (**unverified**; older docs said 500 one-time credits) | Standard commercial terms; some sites refuse hosted scrapers | GET 405 ⚠️ (`Use POST`; host live, no auth proof) | later |

---

## 5. Reference and social signals

**Rules.** Social signals are weak: use them as flags, never as probabilities. All user-generated text (Bluesky, Mastodon, HN, Polymarket comments, Wikipedia text) is untrusted and a prompt-injection surface. **Wikimedia** (Wikipedia, Wikidata, Pageviews, EventStreams) requires a descriptive User-Agent **with contact info**; limits are 10 req/min unidentified, 200/min with a compliant UA, max 3 concurrent. The owner picks the contact string (`PYTHIA_USER_AGENT`, a project mailbox, not a personal email). Pageviews gotcha: use the **canonical** article (the redirect `Federal_Reserve_System` shows only ~100 views/day, the canonical `Federal_Reserve` is the right key); read the canonical title from the summary response. The Polymarket Gamma comments row is in §2 (later; not an input).

| Source | Use in decision | Endpoint (example) | Auth | Free limits | Terms | Tested | Priority |
|---|---|---|---|---|---|---|---|
| Wikipedia REST: page summary | One call per market for entity background (plain extract, `wikibase_item` QID, last-modified); resolves the canonical title | `GET https://en.wikipedia.org/api/rest_v1/page/summary/Federal_Reserve` | none (contact UA) | Wikimedia tiers above; our load < 150/day | CC BY-SA 4.0; generic UA gets 403 | 200 ✓ | v1 |
| Wikipedia Action API (search, extracts, revisions) | `list=search` finds the article; `prop=revisions` timestamps show whether and when an article changed (death, appointment, result): a cheap "did it happen" hint | `GET https://en.wikipedia.org/w/api.php?action=query&prop=revisions&titles=Federal_Reserve&rvlimit=3&rvprop=timestamp%7Ccomment%7Cuser&format=json&formatversion=2` | none (contact UA) | 200 req/min with a contact UA; ≤ 3 concurrent | CC BY-SA 4.0; user-editable, so untrusted; a lead, not a resolution source | 200 ✓ (search, extracts, recentchanges also 200) | v1 |
| Wikimedia Pageviews: per article | Daily views of the canonical article vs 28-day median as an attention-spike flag (flag only) | `GET https://wikimedia.org/api/rest_v1/metrics/pageviews/per-article/en.wikipedia/all-access/user/Federal_Reserve/daily/20261001/20261007` | none (contact UA) | ≤ 200 req/s; need a few calls/day; latest complete day is yesterday (UTC) | Data CC0 | 200 ✓ | v1 |
| Wikimedia Pageviews: top articles | Daily top 1,000: which entities are suddenly newsworthy. Filter out `Main_Page`, `Special:*`, `Wikipedia:*` | `GET https://wikimedia.org/api/rest_v1/metrics/pageviews/top/en.wikipedia/all-access/2026/10/07` | none | Same policy; 1 call/day | CC0 | 200 ✓ (55 KB) | later |
| Wikimedia EventStreams (`recentchange` SSE) | Real-time edit stream; overkill at hourly cadence | `GET https://stream.wikimedia.org/v2/stream/recentchange` | none | Long-lived SSE, not request-metered; limit not verified | Wikimedia terms | 200 ✓ (event-stream frames received) | later |
| Wikidata SPARQL | Structured facts: office-holders with dates, births and deaths; verifies "is X currently Y" | `GET https://query.wikidata.org/sparql?format=json&query=SELECT...` | none (contact UA) | 60 s timeout; 60 s compute per 60 s per client; ≤ 5 parallel queries; 429 with Retry-After | CC0; can lag or be vandalised | 200 ✓ | later |
| Wikidata Action API and EntityData | `wbsearchentities` maps a name to a QID; `wbgetentities` returns claims (prefer it over the 161 KB `Special:EntityData`) | `GET https://www.wikidata.org/w/api.php?action=wbsearchentities&search=Federal%20Reserve&language=en&format=json&limit=2` | none (contact UA) | 200 req/min with a contact UA | CC0 | 200 ✓ | later |
| Hacker News search (Algolia) | Tech-community reaction and primary-source links for AI, tech and crypto announcement markets | `GET https://hn.algolia.com/api/v1/search_by_date?query=OpenAI&tags=story&numericFilters=created_at_i%3E1790863380` | none | 10,000 requests/hour per IP | No terms stated; user text untrusted. Search is fuzzy: use quoted phrases and a points filter | 200 ✓ | later |
| Hacker News Firebase API | Fallback raw feed and comment trees | `GET https://hacker-news.firebaseio.com/v0/topstories.json` | none | "No rate limit" (official README) | MIT README; no ToS | 200 ✓ | later |
| Bluesky `searchPosts`, unauthenticated | Weak sentiment and breaking-news hint; returns `hitsTotal` | `GET https://api.bsky.app/xrpc/app.bsky.feed.searchPosts?q=%22federal%20reserve%22&sort=latest&limit=3` | none | 3,000 requests per 5 min per IP; **page 1 only** | Public AT Protocol data. **Fragile**: `public.api.bsky.app` returns 403 for search; the `api.bsky.app` host works; the cursor page returns 403; search may require auth at any time | 200 ✓ (page 1), 403 on cursor | later |
| Bluesky `searchPosts`, authenticated | Same, stable and paginated, with a dedicated bot account | `GET https://bsky.social/xrpc/app.bsky.feed.searchPosts?q=bitcoin&limit=1` | account `BSKY_APP_PASSWORD` | Per-account limits not verified | Free account and app password (`createSession` is a POST, not tested); use a bot account, never the owner's main one | 401 ✓ (`AuthMissing`, live) | later |
| Bluesky public AppView (profiles, author feeds) | Latest posts of named official accounts without search ("did X announce Y") | `GET https://public.api.bsky.app/xrpc/app.bsky.feed.getAuthorFeed?actor=bsky.app&limit=1&filter=posts_no_replies` | none | 3,000 requests per 5 min per IP | Public data; most politicians post on X, so coverage is thin | 200 ✓ | later |
| Mastodon public API (tag timeline, trends) | Very weak attention signal for niche topics | `GET https://mastodon.social/api/v1/timelines/tag/bitcoin?limit=2` | none | 300 requests per 5 min per IP | Per-instance rules; v2 search returned empty and `timelines/public` 422 without auth | 200 ✓ | later |
| Polymarket xtracker (post counter) | Official resolution source of Polymarket "# tweets" markets (Elon Musk, Trump on Truth Social, others): raw post timestamps give a posts/hour rate model and the exact current count. 10 tracked handles | `GET https://xtracker.polymarket.com/api/users/elonmusk/trackings` (92 KB; `/posts` is 4.4 MB: avoid); `/api/users`, `/api/trackings/{id}` | none | None published; ≤ 1 call/hour per handle, cache | **Undocumented** first-party JSON behind the front end; no ToS or stability promise | 200 ✓ | later (P7 backlog; researcher said v1 if tweet-count markets are in scope) |
| Google Trends "Trending now" RSS | ~10 trending searches per country with approximate traffic; cannot answer a specific market | `GET https://trends.google.com/trending/rss?geo=US` | none | Not published; a few calls/day | No explicit API terms; `robots.txt` disallows only `/explore`; old `dailytrends` paths 404 | 200 ✓ | later |
| Google Trends API (official alpha) | Per-term interest over time for attention markets | `https://developers.google.com/search/apis/trends` (apply; no callable endpoint until accepted) | account `GOOGLE_TRENDS_API_KEY` | Not published | Application-based alpha; acceptance not guaranteed; terms unknown | Docs page 200 | later (owner can apply in parallel, free) |
| YouTube Data API v3 | View, like and subscriber counts for threshold markets | `GET https://www.googleapis.com/youtube/v3/channels?part=statistics&forHandle=MrBeast&key=<key>` | free key `YOUTUBE_API_KEY` | 10,000 units/day per project; `search.list` 100 calls/day; reads cost 1 unit | YouTube API Services ToS: stored data ≤ 30 days; read the "derived metrics" clause before modelling on it | 403 ✓ (live; "unregistered callers") | later |
| Twitch Helix API | Stream and viewer data for the 18 twitch.tv-resolved events | `GET https://api.twitch.tv/helix/streams?user_login=stableronaldo` | account `TWITCH_CLIENT_ID` and secret (app token) | ~800 points/min (not re-verified) | Twitch Developer Services Agreement | 401 ✓ (live) | later |

---

## 6. Official and structured data by market category

### 6.1 What Polymarket markets actually resolve on (audit, 2026-10-08)

Method: Gamma `/events?tag_slug=<tag>&closed=false&active=true` over 17 tag feeds (**502 active events**) plus the top 300 markets by 24h volume. Cheap to re-run weekly.

- **`resolutionSource` is empty on 215 of 502 events (43%) and on 145 of the top 300 markets (48%).** For those, the URLs and named sources are in the rules `description`: parse it, do not rely on the field.
- Top `resolutionSource` domains by distinct events: `weather.gov` 35, `hltv.org` 24, `twitch.tv` 18, `pythdata.app` 17, `federalreserve.gov` 15, `kick.com` 10, `claymath.org` 9, `dashboard.ornnai.com` 9, `binance.com` 8, `x.com` 7, `atptour.com` 7, `home.treasury.gov` 6, `bls.gov` 6, `arena.ai` 6, `nfl.com` 6, `ec.europa.eu` 5, `portwatch.imf.org` 4, `bea.gov` 4, `fec.gov` 3, `db.cec.gov.tw` 3, `ecb.europa.eu` 2.
- Domains seen only in rules text: `xtracker.polymarket.com`, `huggingface.co`, `reuters.com`, `cdc.gov`, `senate.gov`, `congress.gov`, `stats.gov.cn`, `stat.go.jp`, `rbnz.govt.nz`, `interieur.gouv.fr`, `dadosabertos.tse.jus.br`, `indec.gob.ar`, `inegi.org.mx`, `gasprices.aaa.com`, `spacex.com/launches`, `justice.gov`, `wsj.com`.

| Resolution domain | Machine source we use | Status |
|---|---|---|
| `weather.gov` (temperature) | AviationWeather METAR, NWS API (§7) | adapter, P6 |
| `federalreserve.gov`, NY Fed `openmarket.htm` | Fed press RSS, FOMC calendar page, NY Fed Markets API, FRED (§6.2) | adapter, v1 |
| `home.treasury.gov` | Daily par yield curve CSV (§6.2) | adapter, v1 |
| `bls.gov`, `bea.gov` | BLS API (keyless v1), FRED mirror for BEA series (§6.2) | adapter, v1 |
| `ec.europa.eu`, `ecb.europa.eu` | Eurostat API, ECB Data Portal (§6.2) | Eurostat v1, ECB later |
| `portwatch.imf.org` | PortWatch ArcGIS (§6.4) | adapter, v1 |
| `arena.ai`, `lmarena.ai` | LMArena dataset on Hugging Face (§6.4) | adapter, v1 |
| `binance.com`, `pythdata.app` | Binance klines (later); Pyth needs a paid key (§9) | later; skip Pyth |
| `x.com`, xtracker | xtracker JSON (§5) | later (P7) |
| `fec.gov`, `justice.gov`, `congress.gov`, courts | OpenFEC (key), DOJ API, Congress.gov (key), CourtListener (§6.3) | adapters, v1 for DOJ, Congress, CourtListener |
| Stat agencies (INDEC, IBGE, ONS, BoJ, StatCan) | National APIs (§6.2) | later, per market |
| `twitch.tv`, `youtube.com` | Key or OAuth APIs (§5) | later |
| **No usable API (HTML only):** `hltv.org`, `gol.gg`, `vlr.gg`, `dotabuff`, `atptour.com`, `nfl.com`, `nhl`, `claymath.org`, `dashboard.ornnai.com`, `gasprices.aaa.com` (EIA weekly is a proxy only), `db.cec.gov.tw`, `interieur.gouv.fr`, `stats.gov.cn`, `stat.go.jp` (e-Stat needs an appId), `spacex.com/launches` (Launch Library 2 is a proxy), `cdc.gov` measles page | LLM plus search evidence only; sports and esports are outside P1 anyway | none |

**Build rule.** None of §6 blocks Phase 1. Build one resolver per `resolutionSource` domain, in the frequency order above, only when a shortlisted market needs it. Tagged **v1** below means "first resolver set", not "all at once".

### 6.2 Economy, central banks and statistics

| Source | Use in decision | Endpoint (example) | Auth | Free limits | Terms | Tested | Priority |
|---|---|---|---|---|---|---|---|
| FRED API (St. Louis Fed) | Latest and historical CPI, unemployment, payrolls, GDP, Treasury yields (DGS2, DGS10), fed funds target (DFEDTARU, DFEDTARL), H.10 FX; release dates via `fred/releases/dates` | `GET https://api.stlouisfed.org/fred/series/observations?series_id=CPIAUCSL&file_type=json&api_key=<key>` | free key `FRED_API_KEY` (instant) | ~120 requests/min per key (community-documented; the ToU says discretion) | Commercial use allowed; show "This product uses the FRED API but is not endorsed or certified by the Federal Reserve Bank of St. Louis"; third-party series may need permission; do not replicate FRED | 400 ✓ (`Variable api_key is not set`: live) | v1 |
| ALFRED vintages (FRED `realtime_start`/`realtime_end`) | As-first-published versus revised values; avoids look-ahead in backtests | `GET https://api.stlouisfed.org/fred/series/vintagedates?series_id=CPIAUCSL&api_key=<key>` | free key (same) | Same | Same | 400 ✓ (same host) | later |
| BLS Public Data API | Official CPI, core CPI, unemployment, payrolls; resolution source of `bls.gov` CPI markets | `GET https://api.bls.gov/publicAPI/v1/timeseries/data/CUUR0000SA0` (v2 uses POST) | none (v2: free key `BLS_API_KEY`) | v1 keyless: 25 queries/day, 25 series/query, 10 years. v2: 500/day. (The old draft said "25/hour": wrong) | US government, public domain | 200 ✓ (CPI-U Aug 2026 = 334.980). `www.bls.gov` HTML and `bls.ics` return 403 to scripted UAs: use the API and take release dates from FRED | v1 |
| BEA API | GDP and PCE advance and revisions; `bea.gov` resolves GDP markets | `GET https://apps.bea.gov/api/data?UserID=<key>&method=GETDATASETLIST&ResultFormat=JSON` | free key `BEA_API_KEY` | 100 requests/min, 100 MB/min, 30 errors/min then a 1-hour lockout (not re-verified) | Public domain | 200 ✓ (empty `UserID` gives a JSON error: live) | later (FRED mirrors GDP and PCE) |
| US Census Data API | Retail sales, trade, construction (EITS), population; rarely a resolution source | `GET https://api.census.gov/data/2023/acs/acs1?get=NAME,B01001_001E&for=state:06&key=<key>` | free key `CENSUS_API_KEY` | Docs claim 500 queries/IP/day keyless, but **keyless data calls were rejected** | Public domain | 302 to `missing_key.html` ⚠️ (contradicts the docs; treat a key as required; `variables.json` is 13 MB, do not poll) | later |
| Treasury FiscalData API | Debt to the penny, auctions, fiscal series | `GET https://api.fiscaldata.treasury.gov/services/api/fiscal_service/v2/accounting/od/debt_to_penny?sort=-record_date&page%5Bsize%5D=1` | none | None published; < 1 req/s | Public domain | 200 ✓ (2026-10-06: $40.273T) | later |
| Treasury daily par yield curve CSV | **Exact resolution source** of "Treasury yield above or below X" markets (6 events): 1M to 30Y daily yields | `GET https://home.treasury.gov/resource-center/data-chart-center/interest-rates/daily-treasury-rates.csv/2026/all?type=daily_treasury_yield_curve&field_tdr_date_value=2026&page&_format=csv` | none | None published; fetch once a day after ~15:30 ET | Public domain; an official download feed, not a versioned API | 200 ✓ (10/07: 2Y 4.77, 10Y 5.28, 30Y 5.67) | v1 |
| TreasuryDirect auctions | Bill and note auction results | `GET https://www.treasurydirect.gov/TA_WS/securities/auctioned?format=json&type=Bill` | none | None published (300 KB response) | Public domain | 200 ✓ | later |
| NY Fed Markets API (EFFR, SOFR) | Effective fed funds rate and target range to price FOMC markets and confirm a decision took effect | `GET https://markets.newyorkfed.org/api/rates/unsecured/effr/last/1.json` | none | None published | Public data; cite source | 200 ✓ (EFFR 3.88%, target 3.75-4.00) | v1 |
| Federal Reserve press RSS and FOMC calendar page | Detects an FOMC statement the minute it publishes; meeting dates parsed from the HTML calendar | `GET https://www.federalreserve.gov/feeds/press_monetary.xml`; `https://www.federalreserve.gov/monetarypolicy/fomccalendars.htm` | none | None published; poll every few minutes on decision day only | US government work | 200 ✓ | v1 |
| `federalreserve.gov/json/calendar.json` | Machine-readable speeches and testimony list | `GET https://www.federalreserve.gov/json/calendar.json` | none | None published (544 KB) | **Undocumented** site-internal file | 200 ✓ | later |
| Cleveland Fed Inflation Nowcasting | Model nowcast of monthly CPI, core CPI and PCE versus bucket thresholds: the best free prior for CPI markets | `GET https://www.clevelandfed.org/-/media/files/webcharts/inflationnowcasting/nowcast_month.json` | none | None published; 7.6 MB, fetch once a day and cache | Cleveland Fed terms; attribute. Path undocumented and may move | 200 ✓ | v1 |
| Atlanta Fed GDPNow | Current-quarter GDP nowcast | `GET https://www.atlantafed.org/rss/GDPNow` | none | None published | Attribute | 200 ✓ (RSS; XLSX needs `openpyxl`) | later |
| ECB Data Portal (SDMX) | Euro FX reference rates, policy rates, HICP; resolution source of ECB-decision markets | `GET https://data-api.ecb.europa.eu/service/data/EXR/D.USD.EUR.SP00.A?lastNObservations=2&format=jsondata` | none | None published; fair use | Free to reuse with acknowledgement | 200 ✓ (USD/EUR 1.1186) | later |
| Eurostat Statistics API | Euro-area HICP flash and final; oil stocks `nrg_stk_oilm`; sole machine source for `ec.europa.eu` markets | `GET https://ec.europa.eu/eurostat/api/dissemination/statistics/1.0/data/prc_hicp_minr?format=JSON&lang=EN&geo=EA&coicop18=TOTAL&unit=RCH_A&sinceTimePeriod=2026-07` | none | None published; large queries go asynchronous | CC BY 4.0 | 200 ✓ (HICP annual rate 2.9, 3.2, 3.8). Gotcha: legacy `prc_hicp_manr` ended in 2025; use `prc_hicp_minr` with `coicop18` | v1 |
| World Bank Indicators v2 | Annual macro indicators for long-dated markets | `GET https://api.worldbank.org/v2/country/US/indicator/NY.GDP.MKTP.KD.ZG?format=json&per_page=3` | none | None published | CC BY 4.0 | 200 ✓ (first call timed out at 15s, retry 0.6s) | later |
| IMF SDMX API | WEO, IFS and PortWatch-era datasets | `GET https://api.imf.org/external/sdmx/2.1/dataflow/all/all/latest` | none | None published | Attribute IMF | 200 ✓ (old `dataservices.imf.org` no longer resolves) | later |
| IMF DataMapper (WEO forecasts) | Annual growth and inflation forecasts | `GET https://www.imf.org/external/datamapper/api/v1/NGDP_RPCH/USA` | none | None published | Attribute IMF | **403 with a custom UA, 200 with curl's default UA** ⚠️ (Akamai filter; fragile; retest from the isolated server) | later |
| OECD Data Explorer (SDMX) | Composite leading indicators, CPI, unemployment across OECD countries | `GET https://sdmx.oecd.org/public/rest/data/OECD.SDD.STES,DSD_STES@DF_CLI,4.1/USA.M.LI...AA...H?lastNObservations=2&format=jsondata` | none | **Tight**: 20 queries/min and 20 downloads/hour (older FAQ: 60/hour); anonymised traffic blocked; a Mar 2026 notice restricts some parameters (page not readable) | Attribute OECD | 200 ✓ | later (keep < 10 calls/day) |
| DBnomics (aggregator) | One keyless adapter for ECB, Eurostat, IMF, OECD, BLS mirrors; fallback | `GET https://api.db.nomics.world/v22/series/ECB/EXR/D.USD.EUR.SP00.A?observations=1&limit=1&metadata=false` | none | None published | Open platform; licences per provider; mirrors lag hours to days, so never resolution truth | 200 ✓ (280 KB without `metadata=false`) | later |
| ONS time series (UK) | UK CPI (series D7G7, dataset MM23), labour, GDP | `GET https://www.ons.gov.uk/economy/inflationandpriceindices/timeseries/d7g7/mm23/data` | none | None published | Open Government Licence v3 | 200 ✓ | later |
| Bank of Canada Valet | CAD FX and policy rate | `GET https://www.bankofcanada.ca/valet/observations/FXUSDCAD/json?recent=1` | none | None published | Attribute | 200 ✓ | later |
| Bank of Japan time-series API | BoJ policy rate and money data | `GET https://www.stat-search.boj.or.jp/api/v1/getDataCode?format=json&lang=en&db=FM01&code=STRDCLUCON&startDate=202601` | none | None published | Attribute | 200 ✓ | later |
| IBGE Agregados (Brazil) | IPCA inflation and Brazilian macro prints | `GET https://servicodados.ibge.gov.br/api/v3/agregados?assunto=IPCA` | none | None published (listing is 1.9 MB) | Brazilian open data; attribute | 200 ✓ | later |
| Argentina Series de Tiempo (INDEC CPI) | Monthly CPI (15 markets name `indec.gob.ar`) | `GET https://apis.datos.gob.ar/series/api/series?ids=148.3_INIVELNAL_DICI_M_26&limit=1&format=json` | none | None published | CC BY 4.0 per datos.gob.ar (verify) | 200, retest **301** ⚠️ (redirect: follow redirects; check the series updates the day INDEC publishes) | later |
| Statistics Canada WDS | Canadian CPI and labour tables | `GET https://www150.statcan.gc.ca/t1/wds/rest/getAllCubesListLite` (5 MB: do not poll; use `getDataFromVectorsAndLatestNPeriods`) | none | None published | Statistics Canada Open Licence | 200 ✓ | later |
| EIA Open Data API v2 | Weekly US retail gasoline and diesel, crude and gas storage; weekly proxy for the daily `gasprices.aaa.com` markets | `GET https://api.eia.gov/v2/petroleum/pri/spt/data/?frequency=daily&length=1&api_key=<key>` | free key `EIA_API_KEY` | None stated | Public domain | 403 ✓ (`API_KEY_MISSING`: live) | later |

### 6.3 Government, politics, law, conflict

| Source | Use in decision | Endpoint (example) | Auth | Free limits | Terms | Tested | Priority |
|---|---|---|---|---|---|---|---|
| Federal Register API | Executive orders, rules, notices and proclamations the day they publish | `GET https://www.federalregister.gov/api/v1/documents.json?conditions%5Bpresidential_document_type%5D=executive_order&per_page=1&order=newest` | none | None published; ~1 req/s | Public domain; publication lags signing by days | 200 ✓ (1,570 EOs) | v1 |
| Congress.gov API | Bill status, actions, cosponsors, committee stage | `GET https://api.congress.gov/v3/bill?format=json&limit=1&api_key=<key>` | free key `DATA_GOV_API_KEY` (instant) | 5,000 requests/hour per key | Library of Congress data, public domain | 403 ✓ (`API_KEY_MISSING`: live) | v1 |
| GovInfo API | Full text of bills, Congressional Record, public laws, CFR | `GET https://api.govinfo.gov/collections?offsetMark=*&pageSize=1&api_key=<key>` | free key `DATA_GOV_API_KEY` (one api.data.gov key *usually* works across Congress.gov, FEC, NASA, GovInfo: **unverified**) | 1,000 requests/hour | Public domain | 401 ✓ (live) | later |
| House Clerk roll-call XML | Official House votes minutes after they happen | `GET https://clerk.house.gov/evs/2026/roll001.xml` | none | None published | Public domain | 200 ✓ (95 KB) | later |
| GovTrack API (unofficial aggregator) | Fallback for bill, vote and member data | `GET https://www.govtrack.us/api/v2/bill?limit=1&sort=-introduced_date` | none | None published | Private site derived from official data; check attribution; can lag | 200 ✓ | later |
| CourtListener REST v4 (Free Law Project) | Court opinions and RECAP dockets for legal markets (indictments, sentencings, rulings) | `GET https://www.courtlistener.com/api/rest/v4/search/?q=trump&type=o&page_size=1` with `Authorization: Token <token>` | free key `COURTLISTENER_TOKEN` | Authenticated defaults: 5/min, 50/hour, 125/day. Anonymous search works today but must not be relied on | Non-profit; commercial agreements exist for heavy use; RECAP coverage is partial | 200 ✓ (anonymous; 32,890 hits) | v1 |
| DOJ press releases API | Indictments, pleas, convictions, sentencings | `GET https://www.justice.gov/api/v1/press_releases.json?pagesize=1&sort=date&direction=DESC` | none | None published | Public domain | 200 ✓ (273,813 releases; 1.7 s) | v1 |
| OpenFEC API | Candidate and committee filings; `fec.gov` is named on 3 House-race events but certified results come later | `GET https://api.open.fec.gov/v1/candidates/?per_page=1&api_key=<key>` | free key `DATA_GOV_API_KEY` | 1,000 calls/hour (7,200 on request) | Public domain; contributor data not for solicitation | 403 ✓ (`API_KEY_MISSING`: live) | later |
| SEC EDGAR data APIs (`data.sec.gov`) | 8-K, S-1, 13F filings, XBRL facts for merger, IPO and earnings markets | `GET https://data.sec.gov/submissions/CIK0000320193.json` | none; **User-Agent with app name and contact email required** (`SEC_USER_AGENT`) | 10 requests/second max | SEC fair-access policy | 200 ✓. `www.sec.gov/files/company_tickers.json` returned 403 "Request Rate Threshold Exceeded" with a UA lacking an email ⚠️: retest with a compliant UA | v1 |
| ReliefWeb API v2 (UN OCHA) | Humanitarian reports and disasters | `GET https://api.reliefweb.int/v2/reports?appname=<approved>&limit=1` | account: **pre-approved appname** (`RELIEFWEB_APPNAME`) | 1,000 calls/day | Respect original-content IP; v1 API returns 410 | 403 ✓ (live: "not using an approved appname") | later |
| ACLED API | Georeferenced conflict events | `GET https://acleddata.com/api/acled/read?_format=json&limit=1` (OAuth token; valid 24 h) | account (myACLED; `ACLED_EMAIL`, `ACLED_PASSWORD`) | 5,000 rows per request; free extent depends on the user category chosen at signup | Self-declared category incl. commercial; **commercial terms unclear for a trading bot** | 403 ✓ (`Access denied` without token; old host no longer resolves) | later |
| UNHCR Refugee Statistics | Annual refugee and IDP counts | `GET https://api.unhcr.org/population/v1/population/?limit=1&yearFrom=2024&yearTo=2024` | none | None published | CC BY 4.0 per UNHCR (verify) | 200 ✓ | later |
| OCHA HDX HAPI | Harmonised humanitarian indicators by country | `GET https://hapi.humdata.org/api/v2/metadata/location?output_format=json&limit=1&app_identifier=<id>` | free key `HAPI_APP_IDENTIFIER` (from app name and email) | Not verified | HDX/HAPI terms | 403 ✓ (`Invalid app identifier`: live) | later |
| OFAC Sanctions List Service | New SDN designations | `GET https://sanctionslistservice.ofac.treas.gov/api/PublicationPreview/exports/SDN.XML` | none | None published; the full file is 29 MB, do not poll it | Public domain | 200, retest **302** ⚠️ (redirect: follow redirects; the researcher downloaded the full XML) | later |
| UK Parliament open APIs | MP details, Commons divisions, bill stages | `GET https://commonsvotes-api.parliament.uk/data/divisions.json/search?queryParameters.take=1` | none | None published | Open Parliament Licence | 200 ✓ | later |
| Brazil TSE open-data CKAN | Candidate and result datasets for Brazilian elections (runoff 25 Oct) | `GET https://dadosabertos.tse.jus.br/api/3/action/package_list` | none | None published | Brazilian open data | 200 ✓ (datasets, not a live-night feed) | later |

### 6.4 Space, shipping, tech leaderboards, crypto-adjacent

| Source | Use in decision | Endpoint (example) | Auth | Free limits | Terms | Tested | Priority |
|---|---|---|---|---|---|---|---|
| Launch Library 2 (The Space Devs) | Launch schedule and outcomes for SpaceX, Starship and rocket launch-count markets (`spacex.com/launches` has no API) | `GET https://ll.thespacedevs.com/2.3.0/launches/upcoming/?limit=1&mode=list` | none | **15 requests/hour per IP**; cache, ≤ 1 call/hour | Free tier is hobby-oriented; **commercial clause not verified** | 200 ✓ (470 upcoming) | v1 |
| IMF PortWatch (ArcGIS REST) | Daily vessel transits for the Strait of Hormuz and other chokepoints; named source of 4 "Hormuz traffic" events | `GET https://services9.arcgis.com/weJ1QsnbMYJlCHdG/arcgis/rest/services/Daily_Chokepoints_Data/FeatureServer/0/query?where=portname%3D%27Strait%20of%20Hormuz%27&outFields=date,portname,n_total,capacity&resultRecordCount=2&f=json&orderByFields=date%20DESC` | none | No quota published; `resultRecordCount` cap 2,000 | Attribute IMF PortWatch; licence not verified | 200 ✓ (data lags ~4-5 days) | v1 |
| LMArena leaderboard dataset (Hugging Face) | Current rank and Elo of AI models; `arena.ai` is the named source of "which lab has the top model" markets (6 events name `arena.ai`; 9 counting `lmarena.ai`) | `GET https://huggingface.co/api/datasets/lmarena-ai/leaderboard-dataset` | none | 500 requests per 300 s | CC BY 4.0 dataset; leaderboard web pages stay the resolution truth. Style-control variants matter; file layout not yet inspected | 200 ✓ (modified 2026-10-08) | v1 |
| Binance Spot market data | Settlement feed of BTC and ETH up/down and threshold markets (1-minute kline close); **P1 excludes crypto price markets** | `GET https://data-api.binance.vision/api/v3/klines?symbol=BTCUSDT&interval=1m&limit=1` | none | 6,000 weight/min per IP | Binance API terms; US blocked; `binance.vision` is a market-data-only mirror | 200 ✓ (both hosts; **retest from the isolated server**) | later (researcher said v1) |
| CoinGecko API | Market cap, volume and rank for non-price crypto markets | `GET https://api.coingecko.com/api/v3/simple/price?ids=bitcoin&vs_currencies=usd` | none (Demo key `COINGECKO_DEMO_API_KEY`, header `x-cg-demo-api-key`) | Keyless is IP-shared and throttled. Demo: 100/min per the doc fetched today (older: 30/min, 10,000/month: verify) | Demo plan requires attribution; check ToS for commercial use | 200 ✓ | later |
| USGS Earthquake FDSN | Magnitude and time for "M7+" markets | `GET https://earthquake.usgs.gov/fdsnws/event/1/query?format=geojson&minmagnitude=6&limit=1&orderby=time` | none | 20,000 events per query | Public domain | 200 ✓ | later |
| NOAA SWPC space weather | Kp index, flare alerts | `GET https://services.swpc.noaa.gov/products/noaa-planetary-k-index.json` | none | None published | Public domain | 200 ✓ | later |
| NASA Open APIs | Asteroid approaches (NeoWs), solar flares (DONKI) | `GET https://api.nasa.gov/neo/rest/v1/feed/today?api_key=<key>` | free key `DATA_GOV_API_KEY` | `DEMO_KEY` 30/hour and 50/day; registered key 1,000/hour | Public domain | 403 ✓ (`API_KEY_MISSING`: live) | later |

---

## 7. Weather (P6)

### 7.1 Resolution-source finding (Gamma weather events, 2026-10-08)

391 unique weather events over 4 pages; **284 are daily temperature events** (150 highest, 134 lowest) across **51 cities**. Resolution sources are in each market's `description` (`event.resolutionSource` is **empty** for Hong Kong, Moscow, Tel Aviv and Istanbul, so always parse the description):

| Resolves on | Events | Rule detail |
|---|---|---|
| NOAA `weather.gov/wrh/timeseries?site=<ICAO>` | 268 (47 cities) | Highest reading in the "Temp" column. Fallback: Weather Underground "Daily Observations" if NOAA is missing by 11:59 PM ET the next day |
| Hong Kong Observatory "Absolute Daily Max/Min" (Daily Extract) | 6 | 0.1 °C; 7-day wait; first publication counts |
| Weather Underground as **primary** | 10 | Taipei RCSS is live now; Jinan ZSJN and Zhengzhou ZHCC are old May events |

67 events settle in whole °F (US cities), the rest in whole °C. Fee schedule on temperature markets: rate 0.05, taker-only fee, maker rebate 25%, `negRisk` true, `orderMinSize` 5. The NYC rule says to use the "Show Hourly Data" view.

**City to resolution station:** amsterdam EHAM, ankara LTAC, atlanta KATL, austin KAUS, beijing ZBAA, buenos-aires SAEZ, busan RKPK, cape-town FACT, chengdu ZUUU, chicago KORD, chongqing ZUCK, dallas KDAL, denver KBKF, guangzhou ZGGG, helsinki EFHK, hong-kong HKO, houston KHOU, istanbul LTFM, jeddah OEJN, jinan ZSJN (WU), karachi OPKC, kuala-lumpur WMKK, london EGLC, los-angeles KLAX, lucknow VILK, madrid LEMD, manila RPLL, mexico-city MMMX, miami KMIA, milan LIMC, moscow UUWW, munich EDDM, nyc KLGA, panama-city MPMG, paris LFPB, qingdao ZSQD, san-francisco KSFO, sao-paulo SBGR, seattle KSEA, seoul RKSI, shanghai ZSPD, shenzhen ZGSZ, singapore WSSS, taipei RCSS (WU), tel-aviv LLBG, tokyo RJTT, toronto CYYZ, warsaw EPWA, wellington NZWN, wuhan ZHHH, zhengzhou ZHCC (WU).

**Gotchas for the build**
1. The resolution airport is often not the obvious one: **London EGLC** (City, not Heathrow), **Paris LFPB** (Le Bourget, not CDG), **Moscow UUWW** (Vnukovo), **Denver KBKF** (Buckley, not KDEN), **Houston KHOU** (Hobby).
2. US markets use **hourly METAR rows only**. Do not take the daily max from NWS 5-minute observations.
3. International METARs carry whole-degree °C only; US ASOS adds a T-group (KLGA `T01830117` = 18.3 °C). Whole-°F settlement values need the same rounding weather.gov applies. **Calibrate the rounding on ~30 days of settled markets before trusting edge near bucket borders.**
4. Weather Underground has no free, legitimate API for its airport history pages (§9). Taipei is 1 of 51 cities: skip WU-primary markets, or trade them only at wide edges and verify by hand.
5. The research/03 note and STRATEGIES §P6 say ECMWF ENS has 51 members; the open-data files list **50 perturbed members** for `2t` (no control member found). Open-Meteo shows 51 series because it includes the control. Count what the files contain.

### 7.2 Licence-safe v1 stack (all keyless)

| Layer | v1 choice |
|---|---|
| Market and rules | Gamma `tag_slug=weather`; parse station, unit, rounding and fallback from each rule, with a human check per new city template |
| Live observations | **AviationWeather METAR** (running max; 48 of 49 ICAOs), **NWS API** as US cross-check, **HKO since-midnight** feed for Hong Kong |
| History for calibration | **Iowa Mesonet ASOS archive** |
| Settlement check | **HKO Daily Extract** (Hong Kong); the weather.gov hourly "Temp" column via METAR (everyone else) |
| Forecast ensembles | **NOAA GEFS** (AWS or NOMADS; public domain) and **ECMWF ENS open data** (byte ranges; CC-BY-4.0). **Open-Meteo for the paper phase only** (paper only until owner OK, D-038) |

**Open decision for the owner before any live money (D-029):** Open-Meteo's free API is non-commercial, and a profit-seeking bot is likely commercial (the researcher's reading; the terms do not mention trading). Options: (a) GEFS plus ECMWF direct, no licence risk, more ops work; (b) pay Open-Meteo (API Standard about 29 EUR/month; **confirm ensemble access is included**, since the pricing page lists API Professional as the tier with historical and ensemble data); (c) self-host Open-Meteo (AGPLv3). Not tested: GEFS bulk download speed, ECMWF AWS retention, Open-Meteo weighted call cost for ensembles.

### 7.3 Sources

| Source | Use in decision | Endpoint (example) | Auth | Free limits | Terms | Tested | Priority |
|---|---|---|---|---|---|---|---|
| Polymarket Gamma weather events | Lists active temperature markets and the exact station, unit, rounding and fallback; the P6 parser keys off the rules, not the title | `GET https://gamma-api.polymarket.com/events?tag_slug=weather&closed=false&active=true&limit=100&order=volume24hr&ascending=false` | none | `/events` 500/10s; we use ~1-2 calls/hour | Public read API; ¹ (§2) | 200 ✓ | v1 |
| AviationWeather.gov METAR API (NOAA AWC) | Live running daily max/min per resolution ICAO: a near-exact proxy for the weather.gov "Temp" column; updates P(bucket) intraday | `GET https://aviationweather.gov/api/data/metar?ids=KLGA,EGLC,RJTT,VHHH,ZSPD&format=json&hours=24` | none (a custom User-Agent is requested) | 100 requests/min; 400 entries per query; last 30 days. For bulk use the per-minute cache `https://aviationweather.gov/data/cache/metars.cache.csv.gz` (248 KB). **Exceeding limits gets the IP blocked** | US government, open | 200 ✓ (216 METARs for 5 stations in 24 h; 48 of 49 ICAOs returned data; only ZSJN had none) | v1 |
| NWS API (`api.weather.gov`) | US cross-check of station highs (5-minute ASOS, QC flag) and the NDFD/NBM point forecast `maxTemperature` as a bias reference | `GET https://api.weather.gov/stations/KLGA/observations?start=2026-10-07T00:00:00Z&end=2026-10-08T12:00:00Z&limit=500` | none; a **descriptive User-Agent is required** (empty UA got 403) | No number published ("generous"); retry after ~5 s; cache 2-5 min | Public domain | 200 ✓ (441 features; non-US ICAO such as EGLC returns 404; `/points/40.7794,-73.8803` gives OKX/37,46) | v1 (US only) |
| Iowa Environmental Mesonet ASOS archive | Backfill 1-3 years of METAR temperatures per resolution ICAO to learn station bias, rounding and forecast-error distributions | `GET https://mesonet.agron.iastate.edu/cgi-bin/request/asos.py?station=EGLC&data=tmpf&data=tmpc&year1=2026&month1=10&day1=6&year2=2026&month2=10&day2=8&tz=UTC&format=onlycomma&latlon=no&elev=no&missing=M&trace=T&direct=no&report_type=3&report_type=4` | none | None published; academic service: run serially, one station per request, cache | Public domain; attribution appreciated | 200 ✓ (EGLC: 96 rows; non-US networks are covered: the GB ASOS network lists 112 stations). Short lag: use AWC for live data | v1 |
| NOAA GEFS (NOMADS grib filter, AWS `noaa-gefs-pds`) | 31 members (control plus 30): `TMP` 2 m, `TMAX` and `TMIN` 2 m at 0.5° (NOMADS) or 0.25° (AWS `pgrb2sp25`), turned into P(bucket) after bias correction | `GET https://nomads.ncep.noaa.gov/cgi-bin/filter_gefs_atmos_0p50a.pl?dir=%2Fgefs.20261008%2F06%2Fatmos%2Fpgrb2ap5&file=gep01.t06z.pgrb2a.0p50.f024&var_TMP=on&lev_2_m_above_ground=on&subregion=&toplat=41.5&leftlon=-74.5&rightlon=-73&bottomlat=40` | none | NOMADS **120 hits/min per IP** (abusive IPs firewalled); AWS no key and no stated limit; 4 runs/day (00, 06, 12, 18Z). NOMADS keeps ~4 days; AWS keeps more | Public domain; no restriction | 200 ✓ (valid GRIB2 for one member and a small box). Needs GRIB decoding (`cfgrib`/`eccodes` or Herbie); plan ~31 members x ~8 lead times per run with a city subregion box | v1 |
| ECMWF Open Data: IFS ENS (`data.ecmwf.int` and AWS/GCP/Azure mirrors) | Best-skill free ensemble: 50 perturbed members with `2t`, `mx2t3`, `mn2t3` at 0.25°; second family beside GEFS | `GET https://data.ecmwf.int/forecasts/20261008/00z/ifs/0p25/enfo/20261008000000-24h-enfo-ef.index`, then HTTP **Range** requests; AWS mirror `https://ecmwf-forecasts.s3.eu-central-1.amazonaws.com/<date>/<run>z/ifs/0p25/enfo/...` | none | Portal capped at 500 simultaneous connections overall; rolling archive of the last 12 runs (~2-3 days; 2026-10-01 returned 404); releases ~7-8 h after run time; Python client `ecmwf-opendata` | **CC-BY-4.0 plus ECMWF Terms of Use: commercial use and redistribution allowed with attribution** | 200 ✓. The full 24 h file is **6.7 GB: never download it whole**. The `.index` (2 MB, 8,500 messages) gives byte offsets; one member's `2t` range returned 206, 660 KB. Plan ~8 steps x 33 MB per run, 2 runs/day ≈ 0.5 GB/day | v1 |
| Open-Meteo Forecast and Ensemble API | One JSON call per city returns every member of ECMWF IFS (51), AIFS (51), GFS (31), ICON (40), GEM (21), UKMO (18) for daily `temperature_2m_max/min`: fastest way to build and score the paper log | `GET https://ensemble-api.open-meteo.com/v1/ensemble?latitude=51.505&longitude=0.055&daily=temperature_2m_max&models=ecmwf_ifs025&forecast_days=2&timezone=Europe%2FLondon` | none (key `OPEN_METEO_API_KEY` only on a paid plan) | Free: < 10,000 calls/day, 5,000/hour, 600/min, 300,000/month; no uptime guarantee. At ~50 cities x 3 models x hourly we use ~3,600/day | **Free API is non-commercial only**: **paper only until owner OK (D-038)**; data CC BY 4.0. See the open decision above. The old draft said "commercial use allowed, 100 calls/min": that is wrong | 200 ✓ (all listed models; daily max built from hourly data can sit slightly below the true METAR max: calibrate) | v1 (paper phase only; licence decision pending) |
| Open-Meteo Historical Forecast, Archive, Previous-Runs, Single-Runs | Past deterministic forecasts at the same lead times and ERA5 reanalysis for error measurement and backtests (not archived ensemble members) | `GET https://historical-forecast-api.open-meteo.com/v1/forecast?latitude=40.7794&longitude=-73.8803&start_date=2026-10-01&end_date=2026-10-03&daily=temperature_2m_max&models=ecmwf_ifs025` | none | Same free tier; heavy date ranges cost more weighted calls | Same non-commercial terms: **paper only until owner OK (D-038)**; safest for pre-live research | 200 ✓ | later |
| Hong Kong Observatory Open Data API | Live running max/min at the HKO settlement station, 1-minute cadence, 0.1 °C, the market's unit | `GET https://data.weather.gov.hk/weatherAPI/hko_data/regional-weather/latest_since_midnight_maxmin.csv` | none | No number published; poll ≤ every 5-10 min | HKO open data is free for commercial or non-commercial use via data.gov.hk; separate publication terms need written authorisation for commercial reuse of publication materials: attribute "Hong Kong Observatory" and use only the Open Data API | 200 ✓ (`202610082150,HK Observatory,29.0,25.6`; `rhrread`, `fnd`, `flw` also 200). `CLMMAXT`/`CLMMINT` returned data to Aug 2026 but an **empty list for Sep and Oct**: it lags over a month and cannot settle recent days | v1 |
| HKO Climatological Daily Extract (`cis/dailyExtract`) | The exact table the Hong Kong rules name ("Absolute Daily Max"); confirms the settled value the next morning and backfills history | `GET https://www.weather.gov.hk/cis/dailyExtract/dailyExtract_202610.xml` (JSON text; empty content-type; URL found in the page's own JavaScript) | none | None published; one request per day per month file | **Undocumented** page feed, not the Open Data API: fragile; HKO publication terms apply | 200 ✓ (2026-10-07 already present on 2026-10-08: max 28.8, min 17.3). The path `/en/cis/dailyExtract/...` returns an HTML 404 | v1 (unofficial) |
| NOAA NCEI Access Data Service | Long-history daily Tmax/Tmin (GHCN-Daily) and hourly ISD for bias calibration | `GET https://www.ncei.noaa.gov/access/services/data/v1?dataset=daily-summaries&stations=USW00014732&startDate=2026-10-01&endDate=2026-10-05&format=json&dataTypes=TMAX,TMIN&units=metric` | none | None published; occasional delays | Public domain | 200 ✓ but the latest 2-3 days were missing and `global-hourly` was empty for several recent periods; station ids are GHCN or ISD codes, not ICAO | later (IEM covers it) |
| NOAA NCEI Climate Data Online v2 | Alternative GHCN-Daily and GSOM access with metadata search | `GET https://www.ncei.noaa.gov/cdo-web/api/v2/datasets` with header `token` | free key `NCEI_CDO_TOKEN` (email) | 5 requests/s and 10,000/day per token | Public domain | 400 ✓ (`Token parameter is required`: live) | later |
| DWD Open Data: ICON-EPS, ICON-EU-EPS, ICON-D2-EPS | Extra European ensemble family (40 global members; finer Europe and Germany grids) | `GET https://opendata.dwd.de/weather/nwp/icon-eps/grib/12/t_2m/` | none | None published; bz2 GRIB2 per lead time, 4 runs/day | GeoNutzV: commercial use allowed with "Source: Deutscher Wetterdienst" (the DWD copyright page returned 404; terms from search results) | 200 ✓ (70 files for one run). Global files use an **icosahedral grid** and need regridding weights: most work per member | later |
| Environment Canada MSC Datamart GEPS and GeoMet OGC API | GEPS 21-member ensemble (`TMP` 2 m, 0.5°); observations and climate for Toronto | `GET https://dd.weather.gc.ca/today/ensemble/geps/grib2/raw/00/024/`; `https://api.weather.gc.ca/collections/swob-realtime/items` | none | None published; keep requests modest | Server end-use licence: any purpose incl. commercial; attribute ECCC for public applications | 200 ✓ (one file with all members: 3.3 MB, range requests supported) | later |
| UK Met Office Weather DataHub (Site-Specific) | Official UK point forecast for London City (EGLC) as an extra input and benchmark | `GET https://data.hub.api.metoffice.gov.uk/sitespecific/v0/point/hourly?latitude=51.505&longitude=0.055` | free key `METOFFICE_API_KEY` | Free: Global Spot 360 calls/day; Site-Specific Blended Probabilistic 55 calls/day for **one** site. Paid £8-£420/month | "Powered by Met Office data" attribution; **commercial use on the free plan not confirmed** | 401 ✓ (`Missing Credentials`: gateway live) | later |

---

## 8. Cross-market signals


| Source | Use in decision | Endpoint (example) | Auth | Free limits | Terms | Tested | Priority |
|---|---|---|---|---|---|---|---|
| Kalshi public market data (`trade-api/v2`) | Cross-venue price check and sanity bound on P(YES) for matched questions (Fed decisions via series `KXFED`, CPI, 2026 midterms, 2028 elections, some macro and crypto). Fed ladders are threshold contracts ("Above 3.00%") and need conversion to Polymarket buckets. Existing research (Prophet Arena) says the Kalshi price is at least as good as top LLMs | `GET https://api.elections.kalshi.com/trade-api/v2/events?series_ticker=KXFED&status=open&with_nested_markets=true&limit=3`; also `/markets`, `/markets/{ticker}/orderbook`, `/markets/trades`, `/exchange/status` | none | Unauthenticated limits undocumented (authenticated Basic tier: 200 read tokens/s); 1-2 calls per matched market per hour is far below any plausible cap | **Kalshi Developer Agreement v1.1 (PDF) could not be read: commercial-use and redistribution terms are UNVERIFIED.** Internal model input is the lowest-risk use; do not republish. Eligibility rules apply to trading, not to reading | 200 ✓. Docs base `https://external-api.kalshi.com/trade-api/v2` is the same API; old host `trading-api.kalshi.com` returns 401 "moved". Prices are dollar strings (`0.9900`). **Never call `GET /series` unfiltered (18.7 MB)** | v1 (read-only logger; use in the blend is a design decision) |
| Kalshi candlesticks | Backtest the Kalshi-vs-Polymarket gap on matched families; does the gap persist or revert | `GET https://api.elections.kalshi.com/trade-api/v2/series/KXFED/markets/KXFED-26OCT-T2.75/candlesticks?start_ts=1790000000&end_ts=1791460000&period_interval=1440` | none | Same | Same agreement | 200 ✓ | later |
| Manifold Markets API | Weak crowd prior on matched questions (play money). Trust a match only with high title overlap, same resolution date and > ~50 bettors | `GET https://api.manifold.markets/v0/search-markets?term=trump&limit=2&filter=open`; `/v0/market-probs?ids=a&ids=b` (≤ 100 ids) | none | 500 requests/min per IP (do not use several IPs) | Bots allowed; no scraping outside the API; **no AI/ML training on the data for commercial purposes without a data licence**; read-time prompt features are not training, but do not store for fine-tuning | 200 ✓ | later |
| PredictIt market data | Cross-check for US politics (203 live markets). Prices carry a 10% profit fee and per-contract caps, so they bias to the middle | `GET https://www.predictit.org/api/marketdata/all/` (430 KB) | none | **5 requests/min per IP** (`x-rate-limit-limit-minute: 5`); feed refreshes ~1/min; 1 call/hour covers everything | Non-commercial per secondary sources; PredictIt's own terms page was not reachable. Treat as non-commercial; do not redistribute: **paper only until owner OK (D-038)** | 200 ✓ | later |
| Smarkets public API v3 | Exchange prices for UK, EU and US politics that Kalshi lacks. Quotes are basis points (8929 = 89.29%) with depth. Many nodes are category containers whose `/markets/` are empty | `GET https://api.smarkets.com/v3/events/?type_domain=politics&state=upcoming&limit=3&sort=id`; `/v3/markets/{id}/quotes/` | none | 240 requests per ~60 s unauthenticated; API users 1,200 per 60 s | Docs say unrestricted latest prices need API-user status, so free prices may be delayed (looked live); no data licence found: contact `api@smarkets.com` | 200 ✓ | later |
| The Odds API v4 | Sharp-book consensus for Polymarket **sports** markets (vig-free probabilities). Politics coverage is one key, so useless for political questions | `GET https://api.the-odds-api.com/v4/sports/?apiKey=<key>` | free key `ODDS_API_KEY` | 500 credits/month (~16/day); `/v4/sports` is free of quota | Commercial use in user-facing apps allowed; no reselling of the data as a product; no Polymarket or Kalshi prices | 401 ✓ (`MISSING_KEY`; with the literal placeholder `invalid`: `INVALID_KEY`: no real key was used) | later (only if sports enter scope) |

---

## 9. Rejected or not usable

Paid, retired, ToS-restricted, unofficial, geo-gated or broken. Each row says what to use instead.

| Source | Why it is not usable | Class | Tested | Use instead |
|---|---|---|---|---|
| Bing Search API v7 and Bing News Search | Retired 2025-08-11; no new signups. The "Grounding with Bing Search" replacement needs an Azure AI Agent project and returns no raw results | retired | 401 | Exa, Tavily |
| Bing News RSS | Unofficial; only 5 items, localised to the egress (Dutch UI strings), no terms | unofficial | 200 | Google News RSS |
| NewsAPI.org | Developer plan is "for development and testing only", not staging or production; 24-hour delay; 100 requests/day; paid from $449/month | ToS | 401 (live) | Exa, Tavily |
| GNews.io | Free plan is non-commercial, 12-hour delay, 100 requests/day; paid from EUR 49.99/month | ToS | 400 (live) | Exa, Tavily |
| Mediastack | 100 calls/month, delayed data, non-commercial | ToS | 401 (live) | Exa, Tavily |
| Event Registry / NewsAPI.ai | Free tier is evaluation only and may not be used for any project with commercial or institutional value, even indirectly | ToS | 401 (live) | Exa, Tavily |
| SerpAPI | Scrapes search engines (unofficial); 250 searches/month; free-plan commercial use not stated | unofficial | 401 (live) | Exa, Tavily |
| DuckDuckGo | No official search API; the Instant Answer API returns topic summaries only | n/a | 200 | Exa, Tavily |
| GDELT GEO 2.0 | Both documented-style paths return 404; adds nothing to P(YES) | broken | 404 | GDELT DOC |
| X (Twitter) API v2 | No free tier: pay per use (posts read $0.005 each), credits bought upfront by card; free tier closed to new signups in Feb 2026 (secondary sources). Nitter and scrapers violate X terms | paid | 401 (live) | xtracker for tweet counts (§5) |
| Reddit public `.json` / `.rss` | `robots.txt` is `Disallow: /`; Data API Terms require written approval for commercial use and AI/ML use; `.json` returns 403 (datacenter IPs blocked harder) | ToS | 403 (`.rss` 200 but disallowed) | none |
| Reddit Data API (OAuth) | New apps need manual approval (~7 days); free tier is non-commercial; our use is commercial | ToS | 403 (HTML block page, not a clean auth error) | none |
| Telegram public preview (`t.me/s/<channel>`) | Telegram's AI-scraping terms prohibit scraping; no legitimate API for reading arbitrary public channels | ToS | 200 (HTML) | none |
| pytrends / trendspyg | Unofficial scrapers of Google's private endpoints; `pytrends` is archived (last push 2024-08-10) | unofficial | not probed | Google Trends official alpha (§5) |
| SEC EDGAR full-text search (`efts.sec.gov`) | Undocumented backend of the EDGAR UI, not a supported API | unofficial | 403 | `data.sec.gov` submissions (§6.3) |
| FRED `fredgraph.csv` | Website download link, not an API; timed out on both tests | broken | timeout (retest timeout ⚠️) | FRED keyed API (free) |
| Knesset OData | Geo-gated: 303 redirect to a maintenance page, HTML returned. Retest from the isolated server before ever using | geo | 303 | none |
| ProPublica Congress API | Believed to have stopped issuing new keys (not verified) | retired | 401 | Congress.gov, GovTrack |
| Pyth Hermes price API | API key required for Core and Hermes since mid-2026, paid tier unclear (`/v2/price_feeds` listing still works keyless) | paid | 401 | Binance klines for crypto; skip stock and metal up/down markets |
| Meteostat (bulk and API) | Data licensed CC BY-NC 4.0 (non-commercial); the JSON API is RapidAPI-keyed; redundant with IEM and NOAA | ToS | 200 (bulk), 401 (API) | IEM, AWC |
| Synoptic Data API | Open Access tier is not for commercial use. It feeds the weather.gov page, which embeds its own token: reusing it is unauthorised | ToS | 401 | AWC METAR (same observations) |
| Weather Underground / The Weather Company API | Free keys exist only for personal-weather-station owners; airport history pages have no legitimate free API; the page's embedded key must not be copied | paid | 401 | AWC METAR for the same ICAO (§7) |
| Metaculus API | Authentication required for everything; the free token exposes community predictions on only ~50 questions; terms bar training, evaluating or developing AI/ML models and any commercial use without written permission (terms reportedly modified 2026-09-21) | ToS | 403 | Kalshi (§8) |
| Good Judgment Open | No API; terms prohibit scraping, data mining, derivative and commercial use | ToS | 404 | none |
| Polymarket public subgraphs (Goldsky, The Graph) | Deprecated 2026-04-28 with the v2 contracts; they return incomplete or incorrect data. Not queried | retired | docs 200 | Data API v2 (§2) |
| GitHub Models | Retired 2026-07-30 (research/06) | retired | not tested | Gemini, OpenRouter |
| Together AI, Fireworks, Cohere, SambaNova | No usable free tier: Together needs a $5 minimum purchase; Fireworks a card; Cohere's trial key is evaluation-only (1,000 calls/month); SambaNova docs conflict (20 RPD vs a payment method) (research/06) | paid / ToS | not tested | Free roster in §3 |
| Cerebras | One-time $5 credit, valid card needed, expires 30 days after activation. Optional single 30-day burst in the paper phase only | paid | not tested | Free roster in §3 |
| Z.ai (GLM free models, direct) | Limits not published; China-based provider; optional extra family only after measurement | unofficial | not tested | OpenRouter `glm-5.3-flash` |

**Dropped from the previous draft of this file** (not re-verified on 2026-10-08, and no v1 need). That draft was produced by a read-only retest agent that should not have written it (FAILURE_LOG F002):

| Source | Why dropped |
|---|---|
| `data-api-rs.polymarket.com`, `wss://ws.polymarket.com` | Wrong hosts. The tested hosts are `data-api.polymarket.com/v2/*` and `wss://ws-subscriptions-clob.polymarket.com/ws/market` (§2) |
| GDELT Cloud (`gdeltcloud.com`, `GDELT_API_KEY`) and GDELT BigQuery | Not tested. GDELT DOC 2.0 is **keyless** (the old draft wrongly said it needs OAuth) |
| Finnhub news sentiment | Premium only |
| IEA data portal | No endpoint tested; EIA covers US energy |
| Bitquery GraphQL, Kafka, Parquet | Needs a key; 7-day realtime window; Data API v2 covers positions and trades. Revisit only for on-chain audit |
| Polygon RPC (`polygon-rpc.com`), Allium | Not a bet-decision input (wallet balance checks belong to ops, RISK_AND_CUSTODY) |
| Alpha Vantage, Yahoo Finance (unofficial `query1.finance.yahoo.com`) | Equity quotes are not a P1, P6 or P3 input; Alpha Vantage commercial use needs a subscription; Yahoo is unofficial |
| GitHub API, `agent-next/polymarket-paper-trader` | Reference code, not decision data (see research/05) |
| Polymarket Builder API credentials | Trading credentials, not a data source (RISK_AND_CUSTODY) |

**Figures corrected versus the previous draft:** Gemini free RPD 1,000 → **500**; Open-Meteo "commercial use allowed, 100 calls/min" → **non-commercial on the free tier**, 600/min and 10,000/day; BLS "25 requests/hour" → **25 queries/day keyless, 500/day with a free key**; Gamma "300 req/10s" → 4,000/10s general, 300/10s on `/markets` only; Mistral "100/day" → **~200/day (ARGUS measured)**; NWS 404 in the LLM-category test came from non-US coordinates (US points return 200).

---

## 11. Test log


| Source | Example URL | Researcher | Retest | Verdict |
|---|---|---|---|---|
| Gamma markets/events keyset | `gamma-api.polymarket.com/markets/keyset` | 200 | agrees | working |
| Gamma tags, series, sports, teams | `gamma-api.polymarket.com/tags/slug/geopolitics` | 200 | agrees | working |
| Gamma public-search | `gamma-api.polymarket.com/public-search?q=` | 200 (3.7 s) | agrees | working |
| Gamma comments, public-profile | `gamma-api.polymarket.com/comments?...` | 200 | agrees | working |
| CLOB book, books | `clob.polymarket.com/book?token_id=` | 200 | agrees | working |
| CLOB prices (price, midpoint, spread, batch) | `clob.polymarket.com/midpoint?token_id=` | 200 | agrees | working |
| CLOB market parameters | `clob.polymarket.com/clob-markets/{cid}` | 200 | agrees | working |
| CLOB prices-history (legacy) | `clob.polymarket.com/prices-history` | 200 | agrees | working |
| CLOB rewards | `clob.polymarket.com/rewards/markets/current` | 200 | agrees | working |
| CLOB time, ok | `clob.polymarket.com/time` | 200 | agrees | working |
| CLOB market WebSocket | `wss://ws-subscriptions-clob.polymarket.com/ws/market` | 101 | agrees | working |
| Sports WebSocket | `wss://sports-api.polymarket.com/ws` | 101 | agrees | working |
| Data API v2 prices-history | `data-api.polymarket.com/v2/prices-history` | 200 | agrees | working |
| Data API v2 trades | `data-api.polymarket.com/v2/trades?condition=` | 200 | agrees | working |
| Data API v2 oi, holders, positions | `data-api.polymarket.com/v2/oi?condition=` | 200 | agrees | working |
| Data API v2 resolutions, tokens | `data-api.polymarket.com/v2/resolutions?condition=` | 200 | agrees | working |
| Data API v2 wallet analytics | `data-api.polymarket.com/v2/leaderboard` | 200 | agrees | working |
| Data API v2 status, openapi | `data-api.polymarket.com/v2/status` | 200 | agrees | working |
| Bridge supported-assets | `bridge.polymarket.com/supported-assets` | 200 | agrees | working |
| `data-api-rs` host from the old draft | `data-api-rs.polymarket.com/users/position_history` | not tested (0) | 0 ⚠️ | broken (wrong host; use `data-api.polymarket.com/v2`) |
| Gemini API | `generativelanguage.googleapis.com/v1beta/models/gemini-3.5-flash-lite/generateContent` | not tested (0) | GET 404 ⚠️ | auth-required-live (unproven: path form `:generateContent`, POST, key needed) |
| Mistral API | `api.mistral.ai/v1/chat/completions` | not tested (0) | 401 ⚠️ | auth-required-live |
| OpenRouter models list | `openrouter.ai/api/v1/models` | 200 (468 models) | agrees | working |
| Groq API | `api.groq.com/openai/v1/chat/completions` | not tested (0) | GET 404 ⚠️ | auth-required-live (unproven: POST-only) |
| Cloudflare Workers AI, DeepSeek, OpenAI | docs only (research/06, research/10) | not tested | n/a | untested |
| NVIDIA NIM model list | `integrate.api.nvidia.com/v1/models` | listed in research/06 | n/a | untested this round |
| Jev (TypeSafe) | `api.typesafe.ai/health`, `/v1/models` | 200, 403 (research/11) | n/a | auth-required-live |
| Google News RSS | `news.google.com/rss/search?q=` | 200 (83, 79 items) | agrees | working |
| GDELT DOC 2.0 | `api.gdeltproject.org/api/v2/doc/doc?...` | 200 (~half of calls 429) | **429** ⚠️ | working (flaky; spacing and backoff) |
| GDELT Context 2.0 | `api.gdeltproject.org/api/v2/context/context?...` | 200 | **timeout** ⚠️ | unconfirmed (retest could not connect) |
| GDELT GEO 2.0 | `api.gdeltproject.org/api/v2/geo/geo?...` | 404 | agrees | broken |
| Wikipedia Current events | `en.wikipedia.org/w/api.php?action=parse&page=Portal:Current_events/...` | 200 | agrees | working |
| Publisher RSS (BBC, Guardian, Al Jazeera) | `feeds.bbci.co.uk/news/world/rss.xml` | 200 | agrees | working |
| Bing Search API v7 | `api.bing.microsoft.com/v7.0/news/search` | 401 | agrees | broken (retired) |
| Bing News RSS | `bing.com/news/search?q=&format=rss` | 200 (5 items) | agrees | working (unofficial) |
| NewsAPI.org | `newsapi.org/v2/everything` | 401 | agrees | auth-required-live |
| GNews.io | `gnews.io/api/v4/search` | 400 | agrees | auth-required-live |
| NewsData.io | `newsdata.io/api/1/latest` | 401 | agrees | auth-required-live |
| Currents API | `api.currentsapi.services/v1/search` | 401 | agrees | auth-required-live |
| Mediastack | `api.mediastack.com/v1/news` | 401 | agrees | auth-required-live |
| The Guardian Open Platform | `content.guardianapis.com/search` | 401 | agrees | auth-required-live |
| NYT Article Search | `api.nytimes.com/svc/search/v2/articlesearch.json` | 401 | agrees | auth-required-live |
| Event Registry | `eventregistry.org/api/v1/article/getArticles` | 401 | agrees | auth-required-live |
| Exa Search | `api.exa.ai/search` | GET 404 (POST-only) | agrees | auth-required-live (⚠️ no auth proof) |
| Tavily Search | `api.tavily.com/search` | 401 | agrees | auth-required-live |
| Brave Search | `api.search.brave.com/res/v1/web/search` | 422 | agrees | auth-required-live |
| Serper.dev | `google.serper.dev/search` | 403 | agrees | auth-required-live |
| SerpAPI | `serpapi.com/search.json` | 401 | agrees | auth-required-live |
| SearXNG public instance | `searx.be/search?q=&format=json` | 200 (HTML only) | agrees | broken for JSON (self-host needed) |
| DuckDuckGo Instant Answer | `api.duckduckgo.com/?q=&format=json` | 200 (no news) | agrees | working (not a search API) |
| Jina Reader | `r.jina.ai/https://...` | 200 | agrees | working |
| Jina Search | `s.jina.ai/ceasefire` | 401 | agrees | auth-required-live |
| Firecrawl | `api.firecrawl.dev/v1/scrape` | GET 405 | agrees | auth-required-live (⚠️ no auth proof) |
| Wikipedia REST summary | `en.wikipedia.org/api/rest_v1/page/summary/Federal_Reserve_System` | 200 | agrees | working |
| Wikipedia Action API | `en.wikipedia.org/w/api.php?action=query&...` | 200 | agrees | working |
| Wikimedia Pageviews (per article, top) | `wikimedia.org/api/rest_v1/metrics/pageviews/...` | 200 | agrees | working |
| Wikimedia EventStreams | `stream.wikimedia.org/v2/stream/recentchange` | 200 | agrees | working |
| Wikidata SPARQL | `query.wikidata.org/sparql` | 200 | agrees | working |
| Wikidata Action API | `wikidata.org/w/api.php?action=wbsearchentities` | 200 | agrees | working |
| Hacker News Algolia | `hn.algolia.com/api/v1/search_by_date` | 200 | agrees | working |
| Hacker News Firebase | `hacker-news.firebaseio.com/v0/topstories.json` | 200 | agrees | working |
| Bluesky searchPosts (unauthenticated) | `api.bsky.app/xrpc/app.bsky.feed.searchPosts` | 200 (cursor 403; `public.api.bsky.app` 403) | agrees | working (fragile) |
| Bluesky searchPosts (authenticated) | `bsky.social/xrpc/app.bsky.feed.searchPosts` | 401 | agrees | auth-required-live |
| Bluesky public AppView feeds | `public.api.bsky.app/xrpc/app.bsky.feed.getAuthorFeed` | 200 | agrees | working |
| Mastodon public API | `mastodon.social/api/v1/timelines/tag/bitcoin` | 200 | agrees | working |
| xtracker (Polymarket) | `xtracker.polymarket.com/api/users/elonmusk/trackings` | 200 | agrees | working (undocumented) |
| Google Trends trending RSS | `trends.google.com/trending/rss?geo=US` | 200 | agrees | working |
| Google Trends API (alpha) | `developers.google.com/search/apis/trends` | 200 (docs page) | agrees | not callable until accepted |
| YouTube Data API v3 | `googleapis.com/youtube/v3/channels` | 403 | agrees | auth-required-live |
| Twitch Helix | `api.twitch.tv/helix/streams` | 401 | agrees | auth-required-live |
| X (Twitter) API v2 | `api.x.com/2/tweets/search/recent` | 401 | agrees | auth-required-live (paid) |
| Reddit `.json`, OAuth | `reddit.com/r/politics/new.json` | 403 (`.rss` 200) | agrees | broken |
| Telegram `t.me/s/<channel>` | `t.me/s/durov` | 200 (HTML) | agrees | working (ToS bars use) |
| pytrends | GitHub API repo metadata only | 200 (archived) | agrees | not probed |
| FRED API | `api.stlouisfed.org/fred/series/observations` | 400 (key missing) | agrees | auth-required-live |
| FRED `fredgraph.csv` | `fred.stlouisfed.org/graph/fredgraph.csv?id=CPIAUCSL` | timeout | **timeout** ⚠️ | broken |
| BLS Public Data API v1 | `api.bls.gov/publicAPI/v1/timeseries/data/CUUR0000SA0` | 200 | agrees | working |
| BEA API | `apps.bea.gov/api/data?method=GETDATASETLIST` | 200 (JSON error without key) | agrees | auth-required-live |
| Census Data API | `api.census.gov/data/2023/acs/acs1` | 302 to `missing_key` | agrees | auth-required-live (docs say keyless; ⚠️ docs wrong) |
| Treasury FiscalData | `api.fiscaldata.treasury.gov/.../debt_to_penny` | 200 | agrees | working |
| Treasury par yield CSV | `home.treasury.gov/.../daily-treasury-rates.csv/2026/all` | 200 | agrees | working |
| TreasuryDirect auctions | `treasurydirect.gov/TA_WS/securities/auctioned` | 200 | agrees | working |
| NY Fed Markets API | `markets.newyorkfed.org/api/rates/unsecured/effr/last/1.json` | 200 | agrees | working |
| Fed press RSS, FOMC calendar | `federalreserve.gov/feeds/press_monetary.xml` | 200 | agrees | working |
| Fed `json/calendar.json` | `federalreserve.gov/json/calendar.json` | 200 | agrees | working (undocumented) |
| Cleveland Fed nowcast | `clevelandfed.org/-/media/files/webcharts/inflationnowcasting/nowcast_month.json` | 200 | agrees | working |
| Atlanta Fed GDPNow | `atlantafed.org/rss/GDPNow` | 200 | agrees | working |
| ECB Data Portal | `data-api.ecb.europa.eu/service/data/EXR/...` | 200 | agrees | working |
| Eurostat | `ec.europa.eu/eurostat/api/dissemination/statistics/1.0/data/prc_hicp_minr` | 200 | agrees | working |
| World Bank v2 | `api.worldbank.org/v2/country/US/indicator/...` | 200 (first call timed out) | agrees | working |
| IMF SDMX | `api.imf.org/external/sdmx/2.1/dataflow/all/all/latest` | 200 | agrees | working |
| IMF DataMapper | `imf.org/external/datamapper/api/v1/NGDP_RPCH/USA` | 403 (custom UA), 200 (curl UA) | agrees | working with default UA only (⚠️ fragile) |
| OECD SDMX | `sdmx.oecd.org/public/rest/data/...` | 200 | agrees | working |
| DBnomics | `api.db.nomics.world/v22/series/ECB/EXR/...` | 200 | agrees | working |
| ONS time series | `ons.gov.uk/.../timeseries/d7g7/mm23/data` | 200 | agrees | working |
| Bank of Canada Valet | `bankofcanada.ca/valet/observations/FXUSDCAD/json` | 200 | agrees | working |
| Bank of Japan stat-search | `stat-search.boj.or.jp/api/v1/getDataCode` | 200 | agrees | working |
| IBGE Agregados | `servicodados.ibge.gov.br/api/v3/agregados` | 200 | agrees | working |
| Argentina Series de Tiempo | `apis.datos.gob.ar/series/api/series?ids=...` | 200 | **301** ⚠️ | working (redirect; follow it) |
| Statistics Canada WDS | `www150.statcan.gc.ca/t1/wds/rest/getAllCubesListLite` | 200 | agrees | working |
| EIA Open Data v2 | `api.eia.gov/v2/petroleum/pri/spt/data/` | 403 (`API_KEY_MISSING`) | agrees | auth-required-live |
| Federal Register | `federalregister.gov/api/v1/documents.json` | 200 | agrees | working |
| Congress.gov | `api.congress.gov/v3/bill` | 403 (`API_KEY_MISSING`) | agrees | auth-required-live |
| GovInfo | `api.govinfo.gov/collections` | 401 (`API_KEY_MISSING`) | agrees | auth-required-live |
| House Clerk roll-call XML | `clerk.house.gov/evs/2026/roll001.xml` | 200 | agrees | working |
| GovTrack | `govtrack.us/api/v2/bill` | 200 | agrees | working |
| CourtListener v4 | `courtlistener.com/api/rest/v4/search/` | 200 (anonymous) | agrees | working |
| DOJ press releases | `justice.gov/api/v1/press_releases.json` | 200 | agrees | working |
| OpenFEC | `api.open.fec.gov/v1/candidates/` | 403 (`API_KEY_MISSING`) | agrees | auth-required-live |
| SEC EDGAR data APIs | `data.sec.gov/submissions/CIK0000320193.json` | 200 (`company_tickers.json` 403) | agrees | working (compliant UA needed) |
| SEC EDGAR full-text search | `efts.sec.gov/LatestSearch/rest/search-index` | 403 | agrees | broken |
| ReliefWeb v2 | `api.reliefweb.int/v2/reports` | 403 (unapproved appname) | agrees | auth-required-live |
| ACLED | `acleddata.com/api/acled/read` | 403 | agrees | auth-required-live |
| UNHCR | `api.unhcr.org/population/v1/population/` | 200 | agrees | working |
| HDX HAPI | `hapi.humdata.org/api/v2/metadata/location` | 403 (invalid app id) | agrees | auth-required-live |
| OFAC Sanctions List Service | `sanctionslistservice.ofac.treas.gov/.../SDN.XML` | 200 | **302** ⚠️ | working (redirect; follow it) |
| NASA Open APIs | `api.nasa.gov/neo/rest/v1/feed/today` | 403 (`API_KEY_MISSING`) | agrees | auth-required-live |
| Launch Library 2 | `ll.thespacedevs.com/2.3.0/launches/upcoming/` | 200 | agrees | working |
| USGS earthquakes | `earthquake.usgs.gov/fdsnws/event/1/query` | 200 | agrees | working |
| NOAA SWPC | `services.swpc.noaa.gov/products/noaa-planetary-k-index.json` | 200 | agrees | working |
| IMF PortWatch (ArcGIS) | `services9.arcgis.com/.../Daily_Chokepoints_Data/FeatureServer/0/query` | 200 | agrees | working |
| LMArena dataset (Hugging Face) | `huggingface.co/api/datasets/lmarena-ai/leaderboard-dataset` | 200 | agrees | working |
| Binance Spot | `data-api.binance.vision/api/v3/klines` | 200 | agrees | working (**isolated server retest**) |
| Pyth Hermes | `hermes.pyth.network/v2/updates/price/latest` | 401 | agrees | auth-required-live (paid) |
| CoinGecko | `api.coingecko.com/api/v3/simple/price` | 200 | agrees | working |
| UK Parliament APIs | `commonsvotes-api.parliament.uk/data/divisions.json/search` | 200 | agrees | working |
| Brazil TSE CKAN | `dadosabertos.tse.jus.br/api/3/action/package_list` | 200 | agrees | working |
| Knesset OData | `knesset.gov.il/Odata/ParliamentInfo.svc/KNS_Bill` | 303 (geo page) | agrees | broken |
| ProPublica Congress | `api.propublica.org/congress/v1/...` | 401 | agrees | auth-required-live (likely dead) |
| AWC METAR | `aviationweather.gov/api/data/metar?ids=` | 200 | agrees | working |
| NWS API | `api.weather.gov/stations/KLGA/observations` | 200 | agrees | working |
| IEM ASOS archive | `mesonet.agron.iastate.edu/cgi-bin/request/asos.py` | 200 | agrees | working |
| NOAA GEFS (NOMADS, AWS) | `nomads.ncep.noaa.gov/cgi-bin/filter_gefs_atmos_0p50a.pl` | 200 | agrees | working |
| ECMWF Open Data ENS | `data.ecmwf.int/forecasts/.../enfo/...-ef.index` | 200 (Range 206) | agrees | working |
| Open-Meteo forecast and ensemble | `ensemble-api.open-meteo.com/v1/ensemble` | 200 | agrees | working (licence open) |
| Open-Meteo historical APIs | `historical-forecast-api.open-meteo.com/v1/forecast` | 200 | agrees | working |
| HKO Open Data API | `data.weather.gov.hk/weatherAPI/hko_data/regional-weather/latest_since_midnight_maxmin.csv` | 200 | agrees | working |
| HKO Daily Extract | `weather.gov.hk/cis/dailyExtract/dailyExtract_202610.xml` | 200 | agrees | working (undocumented) |
| NCEI Access Data Service | `ncei.noaa.gov/access/services/data/v1` | 200 (recent days missing) | agrees | working |
| NCEI CDO v2 | `ncei.noaa.gov/cdo-web/api/v2/datasets` | 400 (token required) | agrees | auth-required-live |
| DWD Open Data ICON-EPS | `opendata.dwd.de/weather/nwp/icon-eps/grib/12/t_2m/` | 200 | agrees | working |
| ECCC GEPS, GeoMet | `dd.weather.gc.ca/today/ensemble/geps/grib2/raw/00/024/` | 200 | agrees | working |
| UK Met Office DataHub | `data.hub.api.metoffice.gov.uk/sitespecific/v0/point/hourly` | 401 | agrees | auth-required-live |
| Meteostat bulk, API | `bulk.meteostat.net/v2/daily/10637.csv.gz` | 200 (API 401) | agrees | working (ToS bars use) |
| Synoptic Data API | `api.synopticdata.com/v2/stations/timeseries` | 401 | agrees | auth-required-live (ToS bars use) |
| Weather Underground (TWC) API | `api.weather.com/v1/location/KLGA:9:US/observations/historical.json` | 401 | agrees | auth-required-live (paid) |
| Kalshi public market data | `api.elections.kalshi.com/trade-api/v2/events` | 200 | agrees | working |
| Kalshi candlesticks | `api.elections.kalshi.com/trade-api/v2/series/KXFED/markets/.../candlesticks` | 200 | agrees | working |
| Manifold Markets | `api.manifold.markets/v0/search-markets` | 200 | agrees | working |
| PredictIt | `predictit.org/api/marketdata/all/` | 200 | agrees | working |
| Smarkets v3 | `api.smarkets.com/v3/events/` | 200 | agrees | working |
| The Odds API v4 | `api.the-odds-api.com/v4/sports/` | 401 (`MISSING_KEY`) | agrees | auth-required-live |
| Metaculus | `metaculus.com/api/posts/` | 403 | agrees | auth-required-live (ToS bars use) |
| Good Judgment Open | `gjopen.com/api/questions` | 404 | agrees | broken (no API) |
| Polymarket subgraphs | docs page only | 200 (docs) | agrees | not queried (deprecated) |



## See also

- [ARCHITECTURE.md](ARCHITECTURE.md) §8 (LLM router, budgets), §15 PM2 (untrusted text), §18 (env files).
- [BUILD_PLAN.md](BUILD_PLAN.md) 1.4 (adapters), 1.8 (geoblock and health checks), 2.1 (evidence stack), 2.3 (limit measurement), 2.9 (weather).
- [STRATEGIES.md](STRATEGIES.md): per-strategy needs. [DECISIONS.md](DECISIONS.md): D-015 (LLM spend), D-024 (separate keys), D-029 (weather licence), D-035 (Jev), D-036 (LLM roster), D-037 (area coverage), D-038 (non-commercial sources).
- [research/09](research/09-areas-and-sources.md) §3: per-area source tables (303 rows) behind §1b. [research/12](research/12-vercel-free-tier-models.md): Vercel free-tier model probe behind D-036.
