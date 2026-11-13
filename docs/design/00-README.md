> Sanitized public copy of an earlier design input; the canonical specifications may supersede it.

# Polymarket LLM + deterministic trading bot: design pack

> Prepared 2026-10-06 as a hand-off to a new, separate repo. Self-contained: a new agent
> should need only this folder plus read access to the two sibling projects named below.
> Nothing here has been built, backtested or traded.

## What this project is

A small (**$50–100 start, ≤ $100–200 over 2–3 months**) bot for Polymarket prediction
markets. It is "ARGUS + NTX":

- **LLM layer**, adapted from ARGUS. It reads market rules and evidence, estimates P(YES), and flags ambiguous or already-decided markets. It never places, sizes or prices an order.
- **Deterministic engine**, carrying over the ntx rulebook. It calibrates the LLM against its own forward record, blends with the market price, and computes edge after Polymarket's taker fee and spread. It then sizes with capped fractional Kelly, gates, signs with a no-withdraw session key, reconciles against the venue, journals and alerts.

**Sibling projects** (read-only references):

- ntx (deterministic OKX bot, NautilusTrader 1.230.0): `[local source path omitted]`. Its `CLAUDE.md` has rules R1–R76.
- ARGUS (LLM-agent crypto bot): `[local source path omitted]`. See `docs/LLM_AGENTS.md` and `docs/FAILURE_LOG.md`.

## Read order


1. `00-README.md`: this file, decisions and open questions
2. `04-risk-and-custody.md` §5 first (legal and geo), because it can stop the project; then the rest of `04`
3. `01-framework-comparison.md`: why not NautilusTrader, what to use, free data
4. `02-architecture.md`: components, data flow, LLM boundaries, cost budget
5. `03-strategies.md`: P1–P5, evidence, test method, kill criteria

## Decisions already made

| # | Decision | By | Where |
|---|---|---|---|
| 1 | Separate repo and project from ntx | owner | — |
| 2 | Owner accepts a signing key that can withdraw (overrides ntx R2 here only). Still: float-only wallet, key off argv/logs/git, and **prefer a no-withdraw session key** if obtainable | owner + this pack | `04` §1 |
| 3 | Budget: $50–100 start, ≤ $100–200 for 2–3 months; free data; low-cost isolated hosting | owner | `05` |
| 4 | LLM for forecasting and selection only; deterministic code for execution and risk | owner | `02` §3 |
| 5 | **Framework: official `polymarket-client` (py-sdk, MIT) + thin custom engine + ARGUS's LLM routing, not NautilusTrader**; revisit if the strategy becomes book-level and NautilusTrader 2.0 is stable | **owner, 2026-10-07 (confirmed)** | `01` |
| 6 | First strategy is P1 (market-anchored LLM forecast on long-tail, preferably fee-free geopolitics markets); P3 scanner observe-only in parallel; P4/P5 deferred or offline | this pack (proposal) | `03` |
| 7 | Validation is a **forward** forecast-only test. Resolved-market LLM backtests are not evidence (temporal leakage) | this pack | `01` §4, `05` Phase 2 |
| 9 | **Legal and geographic restrictions:** verify applicable rules and venue eligibility; geoblock checks are technical checks, not legal determinations | owner, 2026-10-07 | `04` §5 |
| 8 | Never bypass a geo-block: no VPN, proxy or relocation. A block means wind down | this pack, consistent with Polymarket ToS | `04` §5 |

## Open decisions for the owner

| # | Decision | Default if silent | Blocks |
|---|---|---|---|
| ~~D1~~ | ~~Legal read~~ — **closed 2026-10-07: owner confirms it is lawful for them** | — | — |
| **D3** | Apply for a Builder API key and session-key approval (builder@polymarket.com), or accept a hot float wallet | Fallback: hot float wallet ≤ phase cap | Phase 3 |
| **D4** | Fix the G2 gate numbers before Phase 2 starts (≥ 60 resolved forecasts, Brier CI, ECE, PnL > noise control) | Use the `05` proposals | Phase 2 |
| **D6** | Monthly LLM USD cap (`PM_LLM_USD_CAP_MONTH`) and whether any paid model is allowed | Free tiers only; cap $0 | Phase 2 |
| **D7** | Risk defaults in `04` §2 (¼ Kelly, 10% per market, 30% DD → KILL, …) | Use them | Phase 3 |
| **D8** | Whether KILL may auto-flatten (sell into thin books) or only cancel and stop | Cancel and stop only | Phase 3 |
| **D9** | Git phase for the new repo (owner makes the first commits, per working style) | Owner commits first | — |

## Key facts an implementer must not get wrong (all sourced in 01–04)

- **CLOB V2** went live on 2026-04-28. V1 SDKs and V1-signed orders no longer work; collateral is **pUSD**.
- New account wallets since 2026-05-04 are **Deposit Wallets** (signature type 3).
  - A plain EOA trades only if allowlisted, with no documented process.
  - NautilusTrader 1.230.0 supports signature types 0–2 only.
- **Session keys cannot withdraw.** They expire after 180 days, are revocable, and need Builder API approval.
- Official Python SDK = **`polymarket-client`** on PyPI. The PyPI packages `polymarket` and `polymarket-sdk` are not it.
- Taker fee = `shares × rate × p × (1 − p)`. Rates run 0.04–0.07 by category; geopolitics is 0; makers pay none. Fetch the rate at runtime.
- No stop orders, no reduce-only. Size every position as a total loss.
- Resolution runs on the UMA optimistic oracle: $750 bond, 2-hour challenge, DVM vote of about 48h on a second dispute. A market can resolve 50-50.
- Market text, rules, comments and news are **untrusted input** to the LLM.
