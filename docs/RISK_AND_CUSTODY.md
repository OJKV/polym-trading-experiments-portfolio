# Risk, custody, legal and geo

> Canonical. Supersedes `docs/design/04-risk-and-custody.md` (frozen input), updated with
> [`research/05`](research/05-venue-technical-facts.md) (venue facts verified 2026-10-07).
> Not legal or financial advice. Defaults marked *provisional* await owner confirmation (D-025).

## 1. Custody

### 1.1 Facts that shape it

- New Polymarket wallets are **Deposit Wallets** (signature type 3). A plain EOA trades only if allowlisted, and there is no public process for that.
- A **session key** "cannot withdraw funds". It is scoped `CLOB` / `COMBOSRFQ` / `ALL` (the SDK defaults to `ALL`: always pass `CLOB`), lasts a fixed **180 days** and is revocable. It needs a **Builder API key**, and during rollout an email to builder@polymarket.com. It is in beta.
- A session key sees only its own orders; the owner key cannot see them. Reconcile with the session-key credentials.
- **Redemption is not documented as a session-key capability.** It is an owner-signed Deposit Wallet batch via the Relayer. An `AutoRedeemer` contract exists, but whether it covers our wallet is unverified → tested with ~$1 in the canary.
- **No KYC** on polymarket.com international (VP Eng statement, 2026-05-27/28). Monitor.
- A builder profile at `polymarket.com/settings?tab=builder` (Unverified tier) starts with no approval and can derive and deploy a Deposit Wallet for a **fresh local signer** (onboarding path B). Session-key authorisation still needs the email.

### 1.2 Layout

| Tier | Holds | Key lives | Can withdraw |
|---|---|---|---|
| **Session key** (plan A) | trading authority, scope `CLOB` | isolated server, `exec.env` 0600, user `execution-process` (T-014) | **no** |
| Fallback hot key (plan B) | owner key of a **separate** Deposit Wallet holding only the float | isolated server, same protections | yes (accepted override of ntx R2, D-002) |

**Float rule:** the trading wallet holds at most the current phase cap (canary ≤ $10, L1 $50, then ×2 steps). Anything above cap + 20% is swept to cold weekly by the owner.

### 1.3 Key handling (R1, R61; ARGUS F-070, F-133)

- Never print, log or commit a key; check presence, never value. Never in argv.
- Scrub URLs and exception text before logging.
- Generate the session key **on the isolated server**; only its public address leaves the box for the owner to authorise.
- Alert at T−14 days before session-key expiry; revoke immediately on any suspicion.
- **Leak response.** Session key → revoke; nothing can be withdrawn. Hot key → move the float out with the owner key from another machine, then rotate.

### 1.4 Funding (D-027, owner action; route unverified)

- Collateral is **pUSD** (ERC-20 on Polygon, 6 decimals, backed by USDC). `POST https://bridge.polymarket.com/deposit {"address": <wallet>}` returns deposit addresses on several chains, and sent USDC or USDT auto-converts to pUSD.
- Live minimums: **$2** on Polygon, Tron, Solana, Arbitrum, Base and BSC (`GET /supported-assets`; the docs table is stale). Call it before every deposit. Withdrawals are "instant and free".

## 2. Parameters and limits (provisional defaults, D-025)

The single home for every tunable key (ARCHITECTURE §18). Every limit is enforced in code and drilled by a test (R50). Names carry their denominator (R4). Risk state (HWM, KILL latch, daily loss, flows baseline) survives restarts (R55). Changing a money-path default needs the owner (R14).

**Sizing and exposure**

| Key | Default | Why |
|---|---|---|
| `kelly_fraction` | 0.25 | Full Kelly on a mis-calibrated q is ruinous (ARGUS F-024) |
| `min_lot_policy` | `full_kelly_ceiling` (paper, now); live uses it only once D-028 is final, else `strict` | Round up to the 5-share lot only if its cost ≤ the full-Kelly stake and ≤ every cap; else reject (ARCHITECTURE §9). At $50, a strict reject-below-minimum would block almost every trade |
| `max_market_equity_pct` | 10% | One wrong oracle ruling costs at most 10%. Note: the per-market cap fits a 5-share lot at any price at $50, but Kelly does not; see `min_lot_policy` |
| `max_event_equity_pct` | 15% | Per `cluster_id`, worst case across mutually exclusive outcomes |
| `max_category_equity_pct` | 40% | One news shock hits a category |
| `max_rules_bet_equity_pct` (P2b) | 5% | A bet on how a disputed reading resolves |
| `max_total_exposure_equity_pct` | 70% | On exposure-if-filled (positions + resting buys) |
| `strategy_budget_equity_pct` | P1 70%, P6 30% | Fractions of **equity**, inside the total cap |
| `max_open_markets` | 8 | A human must be able to review every position |
| `max_depth_take_ratio` | 0.5 | Take at most half the displayed depth within our limit |
| `min_depth_multiple` | 2.0 | Taker: depth at our limit ≥ 2 × our shares. Scanner pre-filter: depth within 5¢ of mid ≥ 2 × the min lot |
| `shrink_reject_ratio` | 0.5 | A SHRINK below 50% of the intended size becomes REJECT |
| `paper_start_equity_usd` | 50 | Paper meets the same min-lot and cap constraints as live |
| `phase_cap_usd` | canary 10, live 50 (owner sets each step-up) | Exec refuses new entries when NAV + pending buys would exceed it; drilled |
| `float_sweep_margin_pct` | 20% | The owner sweeps anything above cap + margin weekly |

**Edge, forecast and exits**

| Key | Default | Why |
|---|---|---|
| `min_edge_prob` | 0.05 | Above fee, spread and model error |
| `max_edge_prob` | 0.25 | Larger "edges" usually mean a misread rule or stale news |
| `adverse_sel_margin_prob` (maker) | 0.02 | Maker fills skew toward being wrong; refit from paper fills |
| `slip_margin_prob` (taker) | 0.01 | Book moves between snapshot and fill |
| `w_prior` | P1 0.30, P6 0.50 | Blend weight; fixed through G2 (ARCHITECTURE §8) |
| `calib_min_resolved` | 500 | Identity calibrator below this (research/02) |
| `p_clip` | [0.03, 0.97] | Calibrated-probability bounds |
| `max_sample_spread_logit` | 1.0 | Sample disagreement above this → INSUFFICIENT (≈ 0.38 vs 0.62 at mid) |
| `forecast_ttl_hours` | 48 | Older forecasts never trade (R20); not a re-forecast trigger |
| `forecast_material_move_prob` | 0.05 | Mid moved more than this since T → forecast stale (gate rule 3) |
| `reforecast_move_prob` | 0.05 | Re-forecast trigger for held or quoted markets |
| `blackout_hours` | 24 | No new entries this close to the end date |
| `exit_margin_prob` | 0.05 | Sell early only if f < best bid − fee − margin on 2 consecutive forecasts |
| `min_hold_hours` | 12 | No early exit before this |

**Loss limits and halts**

| Key | Default | Why |
|---|---|---|
| `daily_loss_equity_pct` → PAUSE | 10% | ntx escalation ladder |
| `drawdown_from_hwm_equity_pct` → KILL | 30% | On the flow-adjusted HWM; never auto-resume |
| `pnl_drift_usd` / `pnl_drift_equity_pct` → PAUSE | $5 / 2% | NAV must equal baseline + flows + journaled PnL |
| `category_loss_streak` → PAUSE category | 3 | Three straight losses in one category pauses it for review |
| `max_order_attempts_per_hour` | 20 | R19 |

**LLM budget**

| Key | Default | Why |
|---|---|---|
| `llm_daily_call_ceiling` | 120 | Production calls; hard stop in code |
| `llm_daily_call_ceiling_nonprod` | 40 | Golden, ablation, probe and shadow calls |
| `p1_daily_new_markets` | 15 | Seeded sampler quota (ARCHITECTURE §8) |
| `max_markets_per_event_per_day` | 1 | Sampler stratification; limits correlated G2 units |
| `llm_headroom_ratio` | 0.9 | Skip a model at 90% of its measured RPD/TPD |
| `llm_usd_cap_month` | 5 | D-015: the **binding** total of cash spend (`cost_basis = cash`), checked before each call on worst-case tokens; charged to strategy PnL |
| `llm_usd_cap_month_by_provider` | DeepSeek 2, OpenAI 1.5, OpenRouter 1, TypeSafe (Jev) 0.5 | Sub-caps under the total (they sum to 5) |
| `vercel_credit_headroom_ratio` | 0.9 | Stop using Vercel's free $5 credit at 90%, so it never tips into a paid state |
| `llm_usd_cap_day` | 0.33 | Stops one bad day from spending the month |

**Live, canary and approvals**

| Key | Default | Why |
|---|---|---|
| `approval_first_n_trades` | 20 | Live trades only; canary approvals do not count (D-017) |
| `approval_expiry_minutes` | 15 | An ignored proposal expires as REJECT |
| `canary_total_usd` | 10 | Plumbing test only (D-013) |
| `canary_order_usd_max` | 2.5 | One 5-share lot at price ≤ 0.50 |
| `canary_max_orders` | 10 | Each one owner-approved |
| `paper_redeem_lag_hours` | 168 | Weekly owner redemption until the canary proves automatic redemption, then the measured lag |
| `paper_approval_drop_ratio` | 0.2 | Share of would-need-approval decisions dropped in paper, mirroring expired live proposals |

## 3. Venue and market risks

| Risk | What happens | Mitigation |
|---|---|---|
| Oracle / UMA dispute | $750 bond, 2-hour challenge, a second dispute goes to a DVM vote of ~48 h; 50-50 resolutions are possible; whale votes have decided markets against the plain reading | Per-market caps; AMBIGUOUS → REJECT; avoid subjective resolution sources; blackout window |
| Rule clarification after entry | "Additional context" can be added on-chain after trading starts | Rules hash checked every pass; a change blocks adds and forces a re-forecast |
| Thin books | Exiting costs spread plus fee; partial fills | Maker-first; size ≤ half the depth; hold to resolution |
| Capital lock | Money sits until resolution and redemption; a disputed market can take 4–6 days to settle | 3–45-day window; 70% exposure cap; paper models a redemption lag |
| Venue minimums | 5 shares in every sampled market; a $1 marketable minimum is reported but unverified | Read per market; reject, never widen (R6) |
| No stops / reduce-only | — | Every stake sized as a total loss |
| SDK churn | py-sdk 0.10.0 broke positions, trades and price-history APIs; 0.7.1 and 0.8.0 changed session-key APIs | Hash-pinned `polymarket-client==0.12.0`; contract tests; changelog read before any bump |
| Engine restarts / delays | HTTP 425 then 2 min post-only; 503 cancel-only; taker delays on some markets | Order state machine handles each code; restarts are announced ~2 days ahead on Polymarket's Telegram and Discord |
| Unknown submission | A restart does not cancel a signed order | One exec per wallet (R51); reconcile before any retry; intention-based client ids |
| Hostile ecosystem | Fake bot repos with key-stealing packages; malicious npm/PyPI names; a $230k Telegram-bot hack | PM9: only the pinned official SDK; no third-party bot code on the isolated server; dependencies hash-pinned |
| Edge decay | Famous bot edges lasted ~4–5 months ([research/04](research/04-earnings-and-capital-reality.md)) | Monthly re-run of G2 metrics; cut caps when the edge decays |

## 4. LLM failure modes

| Failure | Defence (code, not prompt) | Lesson |
|---|---|---|
| Hallucinated facts or sources | Cited evidence ids must exist in the prompt; uncited → INSUFFICIENT | ARGUS `ground()` |
| Prompt injection from market text, comments or news | Untrusted `<evidence>` blocks, regex flags, no URL following, no tools, no order fields; flag → REJECT + alert; adversarial golden cases | ARGUS F-019 |
| Overconfidence | One calibrator on the aggregate (identity until 500 resolved), clipping, market blend with a fixed prior weight | KalshiBench ECE 0.120–0.395 across models; Gemini 3.8 Flash ECE 0.042 |
| Anchoring on price | Price hidden in pass 1 | ARGUS F-119 |
| Forced trades | Abstention verdicts are first-class | ARGUS F-001 |
| Arithmetic | Code computes dates, fees, odds | ARGUS F-102, F-111 |
| Free-tier churn, 429s, empty replies | Measured limits, 90% headroom, cooldowns, `max_tokens` set, attempt cap, family-diversity check | ARGUS F-016, F-063, F-064, F-107 |
| Silent stub | Snapshot reports `forecaster=llm` with model ids; stub → alert | ARGUS F-145 |
| Evaluation leakage | Frozen forward forecasts; markets resolve after model cutoffs | Paleka et al. |
| Cost blow-up | Daily ceiling + USD cap enforced before each call | ARGUS (its cap was config only) |
| Shared quota with ARGUS | Separate keys and Google projects (D-024) | [research/07](research/07-argus-reuse-review.md) |

## 5. Legal and geographic restrictions

Venue restrictions and applicable laws may change; verify them for the relevant jurisdiction before operation. A geoblock response is a technical signal, not a legal determination. The proposed system checks eligibility at startup and daily, halts new entries and alerts when eligibility is not confirmed, and follows a reviewed wind-down procedure. Do not use a VPN, proxy or relocation to evade restrictions.
