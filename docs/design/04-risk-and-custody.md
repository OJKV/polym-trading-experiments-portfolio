> Sanitized public copy of an earlier design input; the canonical specifications may supersede it.

# 04 — Risk, custody, legal and geo

> Not legal or financial advice. Figures are attributed to sources; limits marked
> "proposal" are design defaults for the owner to accept or change.

## 1. Wallet custody

### 1.1 The good news: a no-withdraw trading key exists (verify eligibility)

The owner has accepted that Polymarket signing needs a key that can withdraw. That overrides
ntx R2 for this project.

**But Polymarket now offers a better option.** It should be tried first because it may make
the override unnecessary:

- Since 4 May 2026, new account wallets are **Deposit Wallets** (signature type 3) (https://docs.polymarket.com/trading/wallets-auth).
- A Deposit Wallet owner can authorise a **Session Key**: "A Session Key cannot withdraw funds from the Deposit Wallet" (https://docs.polymarket.com/trading/session-keys).
- Session keys are scoped to `CLOB`, `COMBOSRFQ` or `ALL`. They expire after 180 days, and the owner can revoke one with `revoke_session_key()` (same page).
- **Catch:** "Authorizing a Session Key requires a Builder API key". During rollout you must "contact builder@polymarket.com to authorize your Builder API key". Eligibility for a solo trader is **unverified**. This is open decision D3 in `00`.

### 1.2 Custody layout

| Tier | Holds | Where the key lives | Can withdraw |
|---|---|---|---|
| **Session key** (preferred) | trading authority only, scope `CLOB` | isolated server, `EnvironmentFile` 0600, separate Linux user | **no** |
| **Fallback hot key** (if no session key) | owner key of a *separate* Deposit Wallet holding only the float | isolated server, same protections | **yes**: this is the accepted R2 override |

**Float rule** (proposal): the trading wallet holds at most the **current phase cap** (`05`),
e.g. $50 in Phase L1. Anything above the cap plus 20% is swept to cold weekly.

### 1.3 Key-handling rules (ported from ntx R1, R61; ARGUS F-070/F-133)

- Never write, print, log or commit a key. Check presence only, never value (R1).
- Never put a key in argv. `/proc/<pid>/cmdline` is world-readable on a shared box (R61).
- Load keys from a 0600 `EnvironmentFile` owned by a dedicated user (`pm`), separate from `ntx`.
- Scrub URLs and exception text before logging. ARGUS leaked secrets through both (F-070, F-133).
- The LLM process (`pm-research`) **never** has a key in its environment (`02` §2).
- Generate the session key on the isolated server. Only its **public address** leaves the box, for the owner to authorise. The NautilusTrader docs describe the same pattern: "Pass only its public address to the authorization method" (https://github.com/nautechsystems/nautilus_trader/blob/develop/docs/integrations/polymarket.md).
- Rotation: re-authorise before the 180-day expiry; alert at T−14 days. Revoke at once on any suspicion.
- **Leak response.** A session key leak → revoke it, and nothing can be withdrawn. A fallback hot key leak → move the float out with the owner key from another machine, then rotate.

### 1.4 Profit sweep

- Weekly, by the **owner** (not the bot): transfer anything above float + 20% from the Deposit Wallet to cold. A session key cannot withdraw by design, which is the point.
- Redemption of resolved positions (to pUSD) is a separate job, `02` §9. Whether a CLOB-scoped session key can redeem is **unverified**; if it cannot, redemption is an owner action and joins the weekly sweep.

## 2. Position limits (proposal defaults; the owner confirms)

| Limit | Default at $50 | Why |
|---|---|---|
| `kelly_fraction` | 0.25 | Full Kelly on a mis-calibrated q is ruinous (ARGUS F-024) |
| `max_market_equity_pct` | 10% | One wrong oracle ruling costs at most 10% |
| `max_event_equity_pct` | 15% | Correlated markets inside one event |
| `max_category_equity_pct` | 40% | One news shock hits a category |
| `max_rules_bet_equity_pct` (P2b) | 5% | A bet on how a disputed reading resolves |
| `max_total_exposure_equity_pct` | 70% | Keeps cash for redemption delays and new edges |
| `max_open_markets` | 8 | Must stay reviewable by a human |
| `daily_loss_equity_pct` → PAUSE | 10% | ntx escalation ladder |
| `drawdown_from_hwm_equity_pct` → KILL | 30% | Stop and re-evaluate; never auto-resume (R55) |
| `min_edge_prob` | 0.05 | Above fee + spread + model error, to be refitted from data |
| `min_days_to_resolution` | 1 | Avoids the dispute window and last-hour informed flow |

- Parameters carry their denominator (ntx R4): `_equity_pct`, `_usd`, `_prob`. Never a bare `_pct`.
- Every limit is **enforced in code and drilled by a test** (R50).
- Risk state (HWM, KILL) survives restarts (R55).

## 3. Venue and market risks

| Risk | What happens | Mitigation |
|---|---|---|
| **Oracle / UMA dispute** | Bond "typically $750 pUSD", 2-hour challenge, second dispute → DVM vote of about 48h; can resolve 50-50 (https://docs.polymarket.com/concepts/resolution). A whale vote resolved a $7M market to Yes with no deal signed (OddsShopper, via ntx research) | Per-market caps; LLM flags ambiguous rules as REJECT; avoid markets whose resolution source is subjective |
| **Rule clarification after entry** | "Additional context" can be issued on-chain after trading begins (same page) | Re-ingest rules on every pass; a changed rules hash forces a re-forecast and blocks adds (gate rule 6) |
| **Liquidity** | Thin books: exiting costs spread plus a taker fee; partial fills | Size against recorded depth; hold-to-resolution default; FOK at limit for taking |
| **Capital lock** | Money sits until resolution and redemption | Resolution window 3–45 days (proposal); exposure cap 70% |
| **Venue minimums** | "Active markets commonly require five shares minimum and marketable orders rejected below $1 notional" (NautilusTrader docs) | Read per market at runtime; reject, never widen (R6) |
| **No stops, no reduce-only** | "Not supported by Polymarket" (NautilusTrader docs) | Size every position as total loss; ntx R15 does not transfer |
| **API / SDK churn** | V1 SDKs stopped working at the V2 cutover (v2-migration doc); py-sdk is 0.x | Pin versions and hashes; assert the venue contract at start-up (R64); canary on upgrade |
| **Unknown submission** | A restart does "not cancel a signed transaction" (NautilusTrader docs) | One submitting process per wallet (R51); reconcile before retry |
| **Account-level KYC** | "Active traders may be subject to mandatory KYC checks" (Biometric Update, 2026-06) | Owner prepares; the bot halts cleanly if the venue refuses |

## 4. LLM failure modes

| Failure | Defence (code, not prompt) | Source of the lesson |
|---|---|---|
| Hallucinated facts or sources | Every cited evidence id must exist in the prompt (grounding check); an uncited forecast is INSUFFICIENT | ARGUS `lessons.py ground()` |
| **Prompt injection from market text, comments or news** | All market descriptions, rules, comments and fetched pages are **untrusted data**. They are wrapped in `<evidence>` blocks, regex-flagged, and never followed as URLs. The LLM has **no tools and no side effects**, and its output schema has no order fields. A flagged injection → REJECT the market for the day + alert | ARGUS F-019, `argus/data/scrub.py`; adversarial tests mandatory |
| Overconfidence | Calibration on our forward record; market blend weight starts at 0 | KalshiBench (ECE 0.120 for the best model); ARGUS F-024 |
| Anchoring on the price | The price is hidden in pass 1 | ARGUS F-119 (anchoring on the classifier) |
| Forced trades | Abstaining (AMBIGUOUS/INSUFFICIENT) is a first-class answer | ARGUS F-001 |
| Arithmetic errors | Code computes dates, fees and odds | ARGUS F-102/F-111 |
| Free-tier churn, 429s, empty replies | Router with measured limits, 429 cooldown, max_tokens always set, attempt cap | ARGUS F-016/F-063/F-064/F-107 |
| Silent stub instead of LLM | Snapshot reports `forecaster=llm` with model ids; alert if a stub ran | ARGUS F-145 |
| Leakage in evaluation | Forward-only scoring with frozen timestamps | Paleka et al. (`01` [L3]) |
| Cost blow-up | Daily call ceiling and monthly USD cap **enforced in code** | ARGUS (its cap was design-only) |


**Facts (as checked 2026-10-06):**

- `GET https://polymarket.com/api/geoblock` returns `blocked`, `ip`, `country` and `region` for the **requesting IP**. Eligibility is judged on the order-placing IP (same page).
- ToS §2.1.4: using a VPN to bypass restrictions violates the terms (ntx research [S2]). Polymarket has tightened enforcement against VPN use (Biometric Update, 2026-06).
- In Sept 2026 ESMA said the largest prediction platforms lack the EU authorisation generally required, and questioned the partial EU geo-blocks (The Block, 2026-09-11: https://www.theblock.co/news/regulation/2026-09-11-esma-says-major-prediction-platforms-lack-eu-authorization-questions-polymarket-and-kalshis-partial-geo-blocks-414239). France, Germany, Italy, Poland, Belgium and Slovakia are already close-only (geoblock doc). **Finland could be added without notice.**

**Rules for this project:**

2. The bot calls the geoblock endpoint at start-up and daily. `blocked:true` → no new orders, and cancel resting orders (allowed under close-only). Alert. **Never route around it.**
4. The owner gets a local legal read on Kazakh law before Phase L1 (open decision D1).
