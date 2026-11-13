# Build plan, gates and metrics

This is a proposed, unimplemented plan. No project forecasts, paper results or live results exist. The estimates are not deadlines. All gates and parameters remain subject to confirmation before data collection.

## Gate rule

Fix thresholds before the corresponding evaluation data exists. Log retuning; after three cycles on the same forward data, start a new window. Record a failed gate without loosening it.

## Phase 0 — Design

Review the research specifications and resolve provisional decisions before implementation. The project has no code yet.

## Phase 1 — Skeleton and observe-only data

No private trading key is required during this phase. Use public venue data and a throwaway paper-signing key. The scanner and recorder start the observation period before forecasting.

| Task | Deliverable and acceptance criteria |
|---|---|
| 1.1 | Package, CLI, typed core, configuration and checks: the suite passes and missing or unknown configuration is rejected. |
| 1.2 | Domain types, fee and tick calculations, and book reader: test fee examples and compute best bid and ask from actual book ordering. |
| 1.3 | Journal, migrations, stored canonical hash chains and risk state: test tampering, migration, recovery and lock contention. |
| 1.4 | Public market and book adapters plus source-call ledger: contract tests on recorded fixtures, read-back, pinned SDK and correct volume units. |
| 1.5 | Scanner and per-strategy universes: fixture-test routing, eligibility and reject codes; record every seen market and alert on data loss. |
| 1.6 | Separate research and execution permissions, service recovery and backups: test isolation, restart and a backup-restore drill. |
| 1.7 | Book/trade recorder and P3 negative-risk observer: journal episodes and alert on missing data. |
| 1.8 | Locking, pause/kill/arming, venue restrictions, health checks, watchdog and dead-man alert: exercise refusal and stale-state drills. |
| 1.9 | Read the SDK capabilities without keys: verify signing/posting, order identity, authentication, heartbeat, cancellation and order types; record unresolved issues. |
| 1.10 | G2 power simulation on **at least seven days** of scanner data, including prices, spreads, resolution times and event clustering. Freeze `N_final`, the minimum detectable effect and the sampler specification **before any LLM forecast exists**. |

**Gate G1:** seven days of unattended operation, a demonstrated stale-case dead-man alert, API read-back, passing contract tests, a successful restore drill and the recorded task 1.10 power decision. These are planned checks, not completed results.

## Phase 2 — Forecasting and paper evaluation

| Task | Deliverable and acceptance criteria |
|---|---|
| 2.1 | Evidence discovery and reading: test injection handling, allowlists, caching and restrictions on fetching URLs from untrusted text. |
| 2.2 | Model routing and cost limits: test ceilings, rate-limit handling and refusal when model-family diversity is insufficient. |
| 2.3 | Measure available provider capacity and record dated limits before committing to a cohort. |
| 2.4 | Versioned forecast prompts and typed schemas: test agreement, ambiguity, resolution and injection cases with a golden suite. |
| 2.5 | Aggregate, calibrate, blend, freeze cohorts and score: test Brier, log loss, ECE and event-clustered bootstrap against known examples. |
| 2.6 | P1 signal, edge, sizing and gates: drill each risk rule and ensure a post-only order cannot cross the spread. |
| 2.7 | Paper venue, order state machine, reconciliation and NAV: enforce the declared fill model and do not re-sign an unknown submission before terminal status is established. |
| 2.8 | Settlement, reporting and controls: show Brier versus market, net paper PnL, permutation null and NO-below-65-cent baseline; label exploratory slices. |
| 2.9 | P6 weather paper study: freeze its power calculation and primary T−12h horizon; validate station templates, data licences, bias correction and cluster caps. Other horizons are exploratory. |

### Gate G2 — P1 (proposed; provisional until D-014 is confirmed)

| Item | Definition |
|---|---|
| Unit | One scored forecast per **market**: its **first** frozen forecast in the cohort. Re-forecasts are not scored. N counts **markets**, and the bootstrap resamples **events** (cluster bootstrap, T-009). Void and 50-50 markets are excluded and counted |
| Primary (confirmatory) test | One test only: P1, all slices pooled, paired Brier difference (blended `q` − market mid, both at T). One-sided 95% event-clustered bootstrap CI |
| Looks | **Early kill** at 100 resolved markets (one per event per day by the sampler): kill if the CI lies entirely above 0 (worse than the market). **Final** at `N_final` (from task 1.10; default 350): PASS if the CI lies entirely below 0. If the point estimate favours us but the CI straddles 0: **one** extension to `N_max` = 600, then PASS or INCONCLUSIVE |
| PnL condition (also required) | Paper PnL after fees, spread and LLM cost > 0 **and** better than the permutation null (shuffle the calibrated probabilities across markets in the cohort, re-run the pipeline and the same fill model, 1,000 reps, p < 0.10) **and** > the NO-below-65¢ baseline's PnL |
| Secondary (must not be worse) | Log loss not worse than the market; ECE (10 bins) ≤ market ECE |
| Exploratory only | Slices (geopolitics, politics, non-English, niche), touch-fill PnL, `w = 0.5` shadow, horizons. They never pass the gate (Holm correction if one is ever promoted to a test) |
| Frozen system | The cohort (roster, prompt hash, calibrator version, `w`) is fixed for the whole test. Any change starts a new cohort and a new count |
| Outcomes | PASS → P1 is eligible for Phase 3 (with Gate C). FAIL → archive P1 and log it. INCONCLUSIVE → "no detectable large edge": P1 does not go live; it may keep running in paper while another strategy proceeds |
| Minimum detectable effect | About 0.02 Brier at N ≈ 350 (recomputed in task 1.10). The blend literature reports gains of 0.002–0.005 on **liquid** markets, which G2 **cannot** detect, and that is accepted: the hypothesis is a *large* edge in the long tail. A small real edge will read as INCONCLUSIVE |
| Rules reading (P2 path) | Hit rate on RESOLVED_IN_SUBSTANCE calls ≥ 98%; informational until ≥ 20 calls |
| Ops | 0 stub runs; injection flags, 429 burn and cap hits all alerted and reviewed |

At ~15 new markets a day, mostly resolving within 21 days, 350 resolved markets is roughly 4–7 weeks after forecasting starts (estimate).

## Phase 2c — Plumbing canary and Gate C

A canary is proposed only after the paper execution components exist and applicable authorization is obtained. It checks plumbing, not strategy quality. Planned caps are at most $10 total, $2.50 per order and ten orders; each canary order requires approval.

| Task | Deliverable and acceptance criteria |
|---|---|
| 2c.1 | Live venue: sign once, journal before send and reconcile by order hash; test unknown-submission handling. |
| 2c.2 | Custody and authorization state: verify wallet approvals, key scope and expiry without exposing credentials. |
| 2c.3 | Approval channel: reject unauthorized senders and expire stale proposals. |
| 2c.4 | Heartbeat: drill cancellation of resting orders within 15 seconds when the five-second heartbeat stops. |
| 2c.5 | Journal at least one observation each of post-only acceptance and crossing rejection, maker fill, taker FAK fill, cancellation, reconciliation, fee check against `feeSchedule`, redemption or its manual path, a v2-market order if available, and a sweep. |
| 2c.6 | Drill the wind-down procedure: cancel, handle close-only exits, redeem and sweep. |

**Gate C:** every applicable 2c.5 observation is journaled, there are zero unresolved reconciliation mismatches and unknown submissions, and the wind-down drill passes. G2 and Gate C must both pass before any strategy is eligible for live consideration.

## Phases 3–4 — Prospective live assessment and scale

After G2 and Gate C, the proposed first live stage requires explicit authorization, a maximum $50 float, approval of the first 20 live trades with a fresh-book recheck, and continued scoring against paper results. G3 requires at least 20 resolved live trades, no unresolved reconciliation or unknown-submission incidents, live realized edge not significantly worse than paper on the same cohort, and nonnegative net PnL. This is a plumbing and safety check: twenty trades do not prove an edge. Later scale-ups are capped at doubling and require a new risk decision after 20 additional resolved trades with G3 holding. P6 requires its own paper gate and canary; any new strategy starts in paper with its own pre-registered gate.

## Evaluation metrics

Compare paired Brier score against the contemporaneous market, log loss, calibration and Brier skill score. Report PnL after fees, spread and model costs; realized versus predicted edge; maker fill rate and post-fill drift; simulated versus shadow fills; drawdown; and cost per forecast and trade.
