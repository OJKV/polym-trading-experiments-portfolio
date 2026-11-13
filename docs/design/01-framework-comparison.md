> Sanitized public copy of an earlier design input; the canonical specifications may supersede it.

# 01 — Framework comparison: is NautilusTrader the best choice for Polymarket?

> Researched 2026-10-06. GitHub stars, last-push dates and licences were read from
> `api.github.com/repos/<owner>/<repo>` and PyPI JSON on that date. Every other figure is
> attributed to the source cited next to it. Nothing here was backtested.

## 1. Short answer

**No, not for this project. NautilusTrader is the strongest open-source Polymarket
*execution engine*, but this project is an LLM-driven, low-frequency forecaster.**

For that, the best fit is:

- Polymarket's official Python SDK (`polymarket-client`, repo `Polymarket/py-sdk`, MIT);
- a thin, custom, deterministic risk and execution engine (a few hundred lines, with the ntx rules ported in);
- the LLM layer adapted from ARGUS.

**When the answer flips to NautilusTrader 2.x** (§6):

- the strategy becomes book-level (passive market making PM-1, or neg-risk arbitrage PM-2); **and**
- NautilusTrader 2.0 ships as a stable release.

The three strongest reasons:

1. **Its strengths do not reach this edge.** NautilusTrader's value is an event-driven L2 engine and a backtester that replays book deltas. An LLM forecaster's edge lives in a probability estimate that changes over hours or days, not in the book.
   - No free historical Polymarket order book exists: the adapter's loader is trades-only ([N1]; local 1.230.0 code, see ntx `docs/strategies/research/prediction-markets.md`).
   - LLM forecasts cannot be honestly backtested on resolved markets because of temporal leakage ([L3], [L4]).
   - So the validation that matters is a **forward paper test**, and that needs no engine.
2. **The ntx-pinned version cannot trade a 2026 account, and the version that can is a pre-release.**
   - Polymarket account wallets deployed on or after 4 May 2026 are Deposit Wallets (signature type 3). A plain EOA can trade only if it is "allowlisted for trading", and no allowlisting process is documented ([P3]).
   - NautilusTrader 1.230.0, pinned in ntx, documents signature types 0–2 only (local `adapters/polymarket/config.py` l.39–43). It imports `py_clob_client_v2` (`execution.py` l.24–31), which Polymarket now marks as superseded by `py-sdk` ([P5]).
   - Deposit Wallet, session-key and split/merge/redeem support exists only in the Rust adapter on the `develop` branch, shipped as **2.0.0rc6 (uploaded 2026-10-05)** ([N1]; PyPI). Putting real money on a release candidate breaks the ntx habit of pinning tested versions.
3. **The complexity costs more than it buys at $50–100 with a few orders a day.**
   - NautilusTrader is LGPL-3.0 and a large Rust/Python engine, run as one node per process.
   - Its Polymarket guide alone runs to 2,745 lines of operational warnings: 600-second signed-batch deadlines, "do not restart and blindly retry an unknown submission", and more ([N1]).
   - A once-an-hour LLM decision loop fits poorly inside an event loop built for microsecond dispatch.
   - The official SDK already covers what a thin engine needs: REST, WebSockets, orders, split/merge, and session keys ([P5], [P4]).

## 2. Candidates and evidence

| Candidate | Licence | Stars / last push (2026-10-06) | Status | Polymarket coverage | Fit here |
|---|---|---|---|---|---|
| **NautilusTrader 1.230.0** (ntx pin) | LGPL-3.0 | 29,659 / 2026-10-06 | Stable, pinned | Python adapter: L1/L2 data, LIMIT/MARKET, GTC/GTD/FOK/IOC, post-only with GTC/GTD, `neg_risk` signing flag, `PolymarketFeeModel`. Signature types **0–2 only**; no split/merge/redeem; trades-only history loader; depends on `py-clob-client-v2` (local code read, ntx research note) | Cannot trade a post-May-2026 Deposit Wallet |
| **NautilusTrader 2.0.0rc6** (develop) | LGPL-3.0 | same repo | **Pre-release** | Rust adapter. Signature types 0–3 incl. Deposit Wallet; **session-key administration**; split/merge/redeem for Deposit Wallet only; user-channel order states; batch submit at most 15; GTD at least 3 min; no reduce-only, stops, OCO or iceberg ("not supported by Polymarket"); no configured minimum size, where "active markets commonly require five shares minimum and marketable orders rejected below $1 notional" ([N1]). macOS arm64 wheels exist for rc6 (PyPI), unlike 1.231.0 (ntx F008) | Technically the most complete; overkill for LLM cadence; RC |
| **Official `polymarket-client` (py-sdk)** | MIT | 135 / 2026-10-06 | 0.x, "minor releases may include breaking changes" ([P5]) | Unified REST + WebSockets, trading, wallet workflows, merges; Polymarket recommends it for new projects ([P5]). Session-key docs give Python examples ([P4]) | **Recommended base** |
| `py-clob-client-v2` | MIT | 167 / 2026-10-06 | Maintained, deprecation notice pointing to py-sdk ([P5]) | CLOB V2 orders (GTC/FOK/FAK), cancel; no WebSocket in README | Fallback only |
| `py-clob-client` (V1), `clob-client` (TS V1), `rs-clob-client` | MIT | 1,225 / 514 / 682 | **Archived**; V1 orders rejected since CLOB V2 went live on 2026-04-28 ([P1]) | — | Dead |
| `clob-client-v2` (TS), `rs-clob-client-v2` (Rust) | MIT | 75 / 143 | Maintained | CLOB V2 | Only if not Python |
| **Polymarket Agents** (`Polymarket/agents`) | MIT | 3,792 / 2024-11-05 | **Archived 2026-05-11** (GitHub notice) | LangChain/RAG CLI that can place trades. Its README covers no risk limits, backtest or position caps | Ideas only; do not base on it |
| **Hummingbot** | Apache-2.0 | 20,322 / 2026-10-06 | Active | A PR "Add Polymarket Event Connector Support" opened 2025-09-02 (search result). Merge state **not verified** | MM-oriented; not for LLM forecasting |
| `Polymarket/poly-market-maker` | MIT | 325 / 2024-07-05 | Stale, pre-V2 | Bands/AMM market maker on V1 client | Dead after V2 |
| `warproxxx/poly-maker` | MIT | 1,506 / 2026-07-09 | Active | Market maker; README warns it "can lose money" (ntx research [S12]). V2 compatibility not verified | Reference for PM-1 only |
| `pmxt-dev/pmxt` | MIT | 2,166 / 2026-09-30 | Active | "CCXT for prediction markets": unified data and trading across venues, hosted or self-hosted ([O1]) | Extra abstraction we don't need for one venue; candidate for cross-venue later |
| `cryptuon/polybot` | MIT | 28 / 2026-07-16 | Small | Agent framework, paper (shadow) mode by default, human approval for live orders (search result) | Too young; ideas only |

**Supply-chain warning.** The official SDK's PyPI name is **`polymarket-client`** (py-sdk `pyproject.toml`, v0.12.0). The PyPI packages `polymarket` (v0.1.2, "export historical odds") and `polymarket-sdk` (v0.2.0) are **not** published from the official repo. Pin by name and hash; never `pip install polymarket`.

## 3. Criteria scored

| Criterion | NautilusTrader 1.230.0 | NautilusTrader 2.0 rc | py-sdk + thin engine | Polymarket Agents |
|---|---|---|---|---|
| Free / open licence | LGPL ✅ | LGPL ✅ | MIT ✅ | MIT ✅ |
| Maintained, current API | ⚠️ no Deposit Wallet | ✅ but pre-release | ✅ official, 0.x | ❌ archived |
| LLM low-frequency fit | ⚠️ heavy | ⚠️ heavy | ✅ | ⚠️ demo-grade |
| Market-making fit | ✅ | ✅✅ | ⚠️ build it yourself | ❌ |
| Backtest support | trades-only + fee model | trades-only + fee model | none built in (not needed for forward test) | none |
| Live-safety features | reconciliation, typed states | + session keys, submission recovery | **we build them** (ntx rules) | none |
| Custody (no-withdraw key) | ❌ (EOA/proxy keys withdraw) | ✅ session keys | ✅ session keys ([P4]) | ❌ |
| Complexity cost | high | high | low | low but unsafe |
| Language | Python on Rust core | Rust core, Python API | Python | Python |

## 4. Backtest data: what is free

| Source | What | Cost | Use |
|---|---|---|---|
| Gamma API (`gamma-api.polymarket.com`) | Market/event metadata, rules text, resolution | free | Universe, rules ingestion, resolved-market labels |
| CLOB `GET /prices-history` | Price time series per token. `interval` max/all/1m/1w/1d/6h/1h; `fidelity` in minutes, default 1 ([P7]) | free | Price at forecast time T, for the PM-3 study and paper marking |
| data-api trades | Trades, at most 10k per request (local NautilusTrader `loaders.py`, ntx research) | free | Fill realism, volume filters |
| `warproxxx/poly_data` | V2 trades from the CTF Exchange V2 contract via Envio HyperSync. The old Goldsky-subgraph pipeline "no longer returns complete data" after the 2026-04-28 migration ([D1]) | free | Bulk trade history |
| `SII-WANGZJ/Polymarket_data` | Community dump: "107GB … 1.1 billion records across 268K+ markets" ([D2]) | free | Research; verify provenance |
| Goldsky | Positions, OI, orders filled; Starter plan "no credit card" ([D3]) | free tier | Optional |
| Historical **order book** | Not free. CryptoStruct sells L2 at €1 per instrument-day from July 2025 (ntx research [S15]); Telonex, PMData and others are commercial ([D4]) | paid | Only for PM-1/PM-2; otherwise record our own |


## 5. Does the LLM have an edge at all? (Evidence that shapes the architecture)

- ForecastBench, as of 2026-01-29: the best LLMs score Brier 0.102 and superforecasters lead them by 0.017 ([L1]).
  - The tied-#2 entry `ensemble_2_crowdadj` used the **market price** as input to adjust its forecast. That step was worth "nearly 0.01 Brier points" ([L1]).
- AIA Forecaster: equal to superforecasters on ForecastBench, but it "underperforms" on a liquid-market benchmark when used alone. "An ensemble combining AIA Forecaster with market consensus outperforms consensus alone" ([L2]).
- KalshiBench (Dec 2025): "systematic overconfidence across all models"; the best model's ECE was 0.120; "only one model achieves a positive Brier Skill Score" ([L5]).
- ARGUS F-024: Kelly sizing on raw LLM confidence inflates size on overconfident calls (`argus-project/docs/FAILURE_LOG.md` l.34).

**Design consequence:**

- The LLM is a **market-anchored adjuster**, not an oracle.
- Its output is recalibrated on our own forward record before it can size anything.
- Liquid markets are the worst place to look for an edge ([L2]). Long-tail, rules-heavy markets are the hypothesis to test, not an assumption.

## 6. When the answer would change

| Trigger | New answer |
|---|---|
| PM-1 (reward MM) or PM-2 (neg-risk arb) becomes the main strategy | NautilusTrader 2.x: post-only GTD quoting, batch submit, L2 deltas, fee model |
| NautilusTrader 2.0 released stable **and** ntx moves to it | Reconsider sharing one engine across both projects |
| `polymarket-client` breaks often in 0.x | Pin harder, or fall back to `py-clob-client-v2` |
| Session keys unavailable to us (no Builder approval) | Unchanged; custody falls back to a hot float wallet (`04`) |
| Cross-venue (Kalshi etc.) is wanted | Evaluate `pmxt` |

## Sources

- [N1] NautilusTrader Polymarket integration (develop) — https://github.com/nautechsystems/nautilus_trader/blob/develop/docs/integrations/polymarket.md ; https://nautilustrader.io/docs/latest/integrations/polymarket/
- [P1] Polymarket, Migrating to CLOB V2 — https://docs.polymarket.com/v2-migration ; launch date 2026-04-28 per https://crypto.news/polymarket-rolls-out-clob-v2-with-1m-liquidity-rewards-to-harden-prediction-markets/
- [P3] Polymarket, Wallets and Authentication — https://docs.polymarket.com/trading/wallets-auth ; Deposit Wallets — https://docs.polymarket.com/trading/deposit-wallets
- [P4] Polymarket, Session Keys — https://docs.polymarket.com/trading/session-keys
- [P5] Polymarket py-sdk — https://github.com/Polymarket/py-sdk ; py-clob-client-v2 deprecation notice — https://github.com/Polymarket/py-clob-client-v2
- [P7] Polymarket, prices history — https://docs.polymarket.com/developers/CLOB/timeseries
- Polymarket Agents — https://github.com/Polymarket/agents ; Hummingbot PRs — https://github.com/hummingbot/hummingbot/pulls
- [O1] pmxt — https://github.com/pmxt-dev/pmxt ; polybot — https://github.com/cryptuon/polybot ; poly-maker — https://github.com/warproxxx/poly-maker
- [D1] poly_data — https://github.com/warproxxx/poly_data ; [D2] https://github.com/SII-WANGZJ/Polymarket_data ; [D3] https://goldsky.com/chains/polymarket ; [D4] https://telonex.io/ , https://pmdata.dev/
- [L1] Forecasting Research Institute, "LLMs Are Closing the Gap on Human Superforecasters" (2026-01-29) — https://forecastingresearch.substack.com/p/llms-are-closing-the-gap-on-human
- [L2] AIA Forecaster technical report — https://arxiv.org/abs/2511.07678
- [L3] Paleka et al., Pitfalls in Evaluating Language Model Forecasters (ICLR 2026) — https://arxiv.org/abs/2506.00723
- [L4] Temporal Leakage in LLM Backtesting (2026) — https://arxiv.org/html/2608.02985
- [L5] KalshiBench — https://arxiv.org/abs/2512.16030
- Local: ntx `.venv/.../nautilus_trader/adapters/polymarket/` (1.230.0); ntx `docs/strategies/research/prediction-markets.md`.
