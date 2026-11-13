# 08 — ntx rules to port


Column "ntx text" is verbatim, extracted by script (pipes stay escaped as in the source). "Polymarket adaptation" is this report's proposal (derived), not ntx text. "as-is" = copy unchanged.

## Coverage check

| Check | Result |
|---|---|
| Ids listed by the pack (copy 46 + adapt 4) | 50 |
| Listed ids missing from ntx CLAUDE.md | **none**: every listed id exists exactly once |
| ntx id range / numbering gaps | R1-R76, 74 rules; gaps: R38, R39 (never defined; not referenced by the pack) |
| Order in ntx file | R1-R32 ascending, then R41, R44-R50, then R76 down to R33 in mixed order (R33-R37 sit last). Grep by id, not by position |
| Pack pairs | R5 "analogue", R9/R42, R10/R37, R35/R40, R48/R58 are listed as pairs; each pair member is given separately below |

## 1. Secrets and safety

| Id | ntx text (verbatim) | Polymarket adaptation |
|---|---|---|
| R1 | **Never write, print, log, or commit a real API key.** Check presence only, never value. | Scope widens: wallet private key, session key, CLOB L2 API creds (key/secret/passphrase), Builder API key, every LLM provider key. Presence checks only. |
| R61 | **Credentials never travel in a process's argv.** `/proc/<pid>/cmdline` is world-readable and this box is shared: signed venue calls go through `ops/okx_rest.py`, which writes headers to curl's stdin (L-14). | `ops/okx_rest.py` -> our Polymarket signing client; keys load from the systemd `EnvironmentFile` (0600) inside the process, never via argv or a subprocess command line. |
| R14 | **Ask before:** spending real money, changing a risk default, adding a paid dependency, or any irreversible venue action. **AMENDED 2026-09-24 (owner):** deploying code to the live node is pre-authorised — run `make deploy-live` (or the three phases with `--yes`) without asking, and report what it did. Everything else in R14 stands: arming, disarming, moving money and flattening a live position are still the owner's. | Drop the 2026-09-24 ntx owner amendment (live-deploy pre-authorisation) until the owner re-grants it for this repo. "Irreversible venue action" includes on-chain txs: token approvals, redemptions, transfers, session-key authorisation/revocation. "Flattening" -> selling a held outcome token into the book. |
| R56 | **Real money needs two independent switches.** The unit pins the venue; `data/LIVE_ARMED` (typed `ARM LIVE`, `ntx.live.arm`) permits orders at all. `--observe`, `--dry-run` and the kill switch never need arming. | `data/LIVE_ARMED` / `ARM LIVE` / `ntx.live.arm` -> the pack's `ARMED` file (`05` line 70) and our arm command. "The unit pins the venue" -> the systemd unit pins mode (paper vs mainnet CLOB). Observe/paper/KILL never need arming. |

## 2. Venue is truth

| Id | ntx text (verbatim) | Polymarket adaptation |
|---|---|---|
| R11 | **Measure venue claims, never infer them.** (Learned: F002.) | As-is. Polymarket claims to measure, not infer: per-market fee rate, tick size, min order size, `neg_risk`, geoblock status, whether a CLOB-scoped session key can redeem (`04` line 47: unverified). |
| R16 | **Never hardcode fee rates.** Fetch from the venue, reconcile against real fills. | "Fetch from the venue" -> per-market/token fee rate from the CLOB at decision time; reconcile against the fee recorded on each fill. Category rates in `02` lines 96-99 are documentation, not constants. |
| R17 | **Never `round()` an order quantity or price.** Use instrument precision helpers, and assert qty > 0 before submit. | "Instrument precision helpers" -> the market's tick size and min order size from the CLOB. Assert shares > 0 and 0 < price < 1 on the tick grid before submit. |
| R18 | **Validate every venue response before arithmetic** (`filled_qty > 0`, `price > 0`, instrument matches). | Checks become: token_id / condition_id match the request, 0 < price < 1, matched size > 0, side and outcome match. |
| R27 | **The journal is analytics, never truth about open positions.** The venue is authoritative; a journal-derived position count must not gate trading unverified. | as-is; "venue" = CLOB open orders + data-api positions + on-chain token/pUSD balances of our wallet. |
| R64 | **Anything the venue can change under us is asserted at start-up, not recorded in a doc.** Account mode (`acctLv`, `posMode`), per-instrument leverage, fees. A fact measured once is not a fact (F055). | Replace "Account mode (`acctLv`, `posMode`), per-instrument leverage, fees" with: wallet address, pUSD balance, token allowances/approvals, session-key validity and expiry (180 d, `04` line 18), per-market fee rate, tick size, min size, `neg_risk`, accepting-orders/closed state, geoblock check. |
| R72 | **An empty cache is not a flat account.** nautilus fills its position cache from a reconciliation task `on_start` does not wait for, so "no position in the cache" may mean "not read yet". Nothing may journal a position as closed, or free an instrument for a new entry, until the **venue itself** says flat; a disagreement or an unanswered venue refuses to start (F066). This is L-09 applied to positions, where it matters more: a stop wrongly believed absent is replaced, a position wrongly believed closed is opened again on top of itself. | Drop nautilus-specific clause ("reconciliation task `on_start` does not wait for"). Rule: nothing frees a market for a new entry or journals a position closed until data-api / chain says the token balance is zero. F066, L-09 are ntx ids. |
| R74 | **Never send a close on the strength of our own cache.** A close is a market order the other way, so closing a position that is not there OPENS one — real money, wrong direction, no stop (F069, twice on live). Worse, it is self-sustaining: every manual close leaves a phantom in the cache and the node buys the position straight back. The venue must confirm the position, on the same side, before any close goes out; a venue that cannot be read does not send. This is R72 applied to the path that spends money instead of the path that writes a row. | Rewrite mechanism: on Polymarket an exit is a SELL of the held outcome token, and a SELL without balance is presumably refused by the CLOB balance check (unverified). The live danger shifts to a phantom cache causing a RE-BUY of a position already held (double exposure). Rule: no exit or re-entry on the strength of our cache; venue/chain must confirm the token balance first; unreadable venue -> do not send. |

## 3. State machines

| Id | ntx text (verbatim) | Polymarket adaptation |
|---|---|---|
| R19 | **Cooldowns and rate limits key on attempts, not successes.** Record the attempt before submitting. | as-is; applies to CLOB order posts, cancels, redemption txs, and LLM calls (attempt counted against the USD cap before the call). |
| R20 | **Every pending/waiting state needs a TTL.** No state may block the pipeline indefinitely. | as-is; Polymarket pending states: resting limit order (use GTD expiry), awaiting fill, awaiting resolution, UMA dispute window, awaiting redemption. |
| R23 | **Handlers iterate ALL pending items per pass** — never return after the first fill. | as-is. |
| R49 | **A position is forgotten only when the venue says flat.** Exits mark `closing`, keep the venue stop, and finalise on confirmed fills; a refused close retries then halts loudly (C-01). | "Venue says flat" -> "market resolved AND redeemed (token balance 0, pUSD credited)". No venue stop exists: drop "keep the venue stop"; exits mark `closing` and finalise on confirmed fills or a confirmed redemption tx; a refused sell/redeem retries then halts loudly. |

## 4. Risk

| Id | ntx text (verbatim) | Polymarket adaptation |
|---|---|---|
| R4 | **Parameters carry their denominator:** `*_price_pct` (asset move), `*_roi_pct` (return on margin), or `*_equity_pct` (account equity). Never a bare `_pct`. Guards the silent 5×-leverage bug. | Denominators change: `*_price_pct` / `*_roi_pct` (perp-specific) are replaced by `*_prob` (probability points, e.g. `min_edge_prob`), `*_equity_pct`, `*_usd`, `*_shares`. Never a bare `_pct`. Drop the "5x-leverage" rationale. |
| R5 | **Sizing is risk-based.** `qty = (equity × risk_equity_pct) / stop_distance`. Leverage is a ceiling that only rejects or shrinks. | Analogue (pack wording "size from equity and risk, caps only shrink"): `stake_usd = kelly_fraction x f_kelly x equity`, then per-market / per-category / phase / book-depth caps only shrink or reject. No stop distance, no leverage on Polymarket. |
| R6 | **Never widen a position to reach a venue minimum.** Reject the trade instead. | "Venue minimum" -> CLOB min order size (shares). A Kelly stake below it is rejected, never rounded up. |
| R50 | **A configured limit that no code enforces is a lie.** Loss/drawdown limits are enforced in the entry path and drilled, not only printed by `doctor` (C-02). | as-is; limits = daily loss, drawdown-from-HWM KILL (`04` line 62), per-market exposure, LLM USD cap; drilled in paper. `doctor` -> our health command. |
| R55 | **Risk state that gates money survives a restart.** High-water mark seeded from the journal, KILL latched on disk and cleared only by hand; guards run on a timer, not only on an entry signal (B-04/B-05). | as-is; latched `KILL` file (`02` C11). |
| R60 | **A cap that shrinks an order is counted** (`size_caps`), not silent — otherwise a 1x run and a 3x run look identical in a results table while one trades smaller (R40). | Example "1x run and a 3x run" -> e.g. a 0.25-Kelly vs 0.5-Kelly run, or a run with vs without the depth cap. Keep the `size_caps` counter. |

## 5. Ops

| Id | ntx text (verbatim) | Polymarket adaptation |
|---|---|---|
| R21 | **All paths absolute.** systemd's cwd is not the repo. Validate at startup, not first use. | as-is. |
| R22 | **Monitoring never uses cached prices.** `/status` and `/stats` compute from live marks. | "Live marks" -> CLOB order book (best bid for marking a long) fetched at query time, not Gamma's cached `outcomePrices` or last trade price. |
| R32 | **Never pipe a command whose exit status matters** — `$?` is the pipe's last stage, so `cmd \| tail` reports success for a crashed `cmd` (F012). Redirect to a file and capture the code. **And read the output before reporting a result.** | as-is. |
| R51 | **One node per strategy per account.** A pid lock refuses a second start; two nodes fight over one position and the venue cannot tell them apart (F046). | "One node per strategy per account" -> one executor process per wallet / session key (pack wording "one process per wallet"). Keep the pid lock. |
| R54 | **The journal never goes first and never raises.** Safety action (flatten, re-attach, kill) first, `_jwrite`/`_jevent` second (O-04). | as-is; safety action = cancel resting orders / latch KILL. |
| R57 | **Deployment is pull-based and verified on the box**: the isolated server fetches, runs the suite itself, and only then restarts; success means a healthy state snapshot (`ntx.ops.snapshot_check`), not exit 0. The LIVE unit is never restarted automatically (docs/24). | `ntx.ops.snapshot_check` and `docs/24` -> our snapshot check and deployment doc. The live executor is never auto-restarted. |
| R63 | **"Refusing to start" must actually stop the process.** nautilus re-raises from `on_start` but leaves the node up; `watch_startup` exits 3 if the trader is not running 90 s in (L-02). | Drop the nautilus mechanism (`on_start` re-raise, `watch_startup` exit 3). Rule: a start-up refusal exits non-zero. Scope it with R73: only whole-process faults (wrong wallet, missing creds, mode mismatch) stop the process; per-market faults refuse that market. |
| R65 | **Never restart the node through a signal.** History replays without trading, so a restart in the first minutes of a bar silently drops that bar's entry (F054). | Rationale is nautilus/bar-specific (history replay drops a bar's entry). Keep the rule: restarts only via the deploy path or systemd, never `kill -HUP` etc.; Polymarket rationale = a mid-cycle restart can drop or double-submit a decision unless order ids are idempotent. |
| R70 | **A check may not report absence when it means "I could not tell".** A monitoring check whose failure path reads as normal ("not installed yet", "expected until go-live") is proven against a positive case, never only a negative one. `systemctl \| grep -q` under `pipefail` returned 141 on a match and called a running unit absent (F060). | as-is. |
| R73 | **A safety check may refuse an action; it may not refuse to run.** A guard that exits the process turns a transient disagreement into an outage that outlives it — systemd's start limit makes it permanent, and the node then protects nothing at all (F068: 65 min of soak lost; `ntx-ui.service` 16 h the same way). Retry what may be transient, then narrow the refusal to the smallest thing that is actually unsafe — one instrument, one order — and keep running, loudly. | "One instrument, one order" -> "one market, one order". |
| R75 | **Alert on change, not on state.** A condition that has not moved is one notification, not one every tick: 1,700 pushes in 72 hours carrying four facts is how an owner learns to ignore the channel, and the next real alarm goes with it (F070). A new fact fires at once; an unchanged one repeats at a human interval saying how long it has been true; a cleared one is forgotten so its return is news. Deduplicate the **push** only — the log keeps everything, because that is what lets a flood be diagnosed. And never swallow on error: too many alerts is a bad day, a missing one is a bad month. | as-is; same ntfy channel. |
| R76 | **A framework default is a decision we made by not making it.** nautilus's `position_check_lookback_mins` defaults to 60 minutes; S003 holds positions for days, so every restart more than an hour after the last fill silently stopped managing whatever was open — stops still capped the loss, but no exit and no trailing stop ran (F071). Any upstream window, timeout, retry count or lookback must be read against **our** holding period and cadence, written down with that reasoning, and re-checked on a version bump. "We never set it" is not a default, it is an unexamined choice. | Replace the nautilus `position_check_lookback_mins` example with SDK defaults to read against our cadence: CLOB client retries/timeouts, order type defaults (GTC vs GTD/FOK), signature type, Polygon RPC timeouts, LLM SDK retries/timeouts, Gamma pagination limits. Re-check on every version bump. |

## 6. Research hygiene

| Id | ntx text (verbatim) | Polymarket adaptation |
|---|---|---|
| R9 | **Every backtest result records config + git SHA + data hash.** Unreproducible is not a result. | "Every backtest result" -> every backtest, paper and calibration result; also record prompt version, model id per ensemble member and evidence-bundle hash. |
| R42 | **Every number quoted in a doc must have a recorded run** under `reports/runs/` with config, source hash, data fingerprint and environment. No result may exist only in terminal output. Experiments live in `src/ntx/experiments/`, never `/tmp`. | Paths: `reports/runs/` keeps; `src/ntx/experiments/` -> `src/<pkg>/experiments/`. |
| R10 | **Do not re-tune on IS after an OOS failure** without logging it. Three cycles contaminates OOS; the strategy is then archived. | IS/OOS -> calibration-fit set vs held-out forward-resolved markets. Keep "three cycles contaminates OOS; strategy archived". |
| R37 | **Check the IS/OOS split before running, not after the result looks interesting** (F017). Exploration and sweeps use 2020-2023; OOS is for final candidate verification only. | Replace "2020-2023" with a split by market resolution date. Add: evaluation markets must resolve AFTER every ensemble model's training cutoff, else the LLM may already know the outcome (lookahead). |
| R12 | **Never fill in a Results section speculatively.** Empty until a run exists. | as-is. |
| R31 | **Verify the read-back, not just the write.** A pipeline that writes correctly and reads wrongly produces plausible, meaningless results. | as-is. |
| R35 | **A backtest that received zero bars must raise, not report "0 trades".** "No signal" and "no data" are indistinguishable in a results table and mean opposite things (F016). | "Backtest that received zero bars" -> any scan, backtest or calibration run that received zero markets, zero books or zero resolved forecasts must raise, not report "0 trades". |
| R40 | **When a knob produces no change, prove the knob was turned.** A silent no-op and a genuine null result are indistinguishable in a results table (F016, F022, F023). Count the vetoes/effects, and assert the call site exists. | as-is. |
| R41 | **Write a FAILING test that reproduces a bug BEFORE fixing it.** Then fix, then watch it pass. A test written after the fix only proves the fix compiles, not that it addresses the bug. Name the test in the failure-log entry. | as-is. |
| R43 | **Run `make mutation` after adding tests to a load-bearing module.** A surviving mutant is a missing test at that exact line. The harness verifies the baseline passes first — otherwise every mutant falsely appears killed. | `make mutation` -> equivalent harness (e.g. mutmut) on load-bearing modules: fee, edge, sizing, risk gates, calibration. |
| R48 | **Anything that can touch the live account needs an observe path first.** `--observe` never orders, never adopts, never cancels (G6). | `--observe` "never orders, never adopts, never cancels" -> observe/paper reads markets, forecasts, journals hypothetical fills, never signs anything. |
| R58 | **Observe mode issues NO venue command.** Not just no orders: no cancel, no re-attach, no flatten, no stale-stop sweep (L-08). Every venue write goes through `_venue_writes_allowed`. | Venue-write list -> no order post, no cancel, no approval tx, no redemption, no transfer, no session-key action. Every venue/chain write goes through one gate function (ntx: `_venue_writes_allowed`). |

## 7. Adapt (pack section "Adapt")

| Id | ntx text (verbatim) | Adaptation (pack wording + notes) |
|---|---|---|
| R2 | **Never enable withdrawal permission** on any key, and never advise it. | Pack (`05` line 113, `04` section 1): "prefer a no-withdraw session key; else a float-only hot wallet". The fallback hot key CAN withdraw, so it is an accepted, documented override of R2 (`04` line 28), bounded by the float rule. |
| R15 | **A position must never exist without a venue-side stop.** Attach at entry; watchdog repairs; `positions_unprotected` must always be 0. | Pack: "every stake sized as total loss". Polymarket has no venue stop orders; a binary position can go to 0. `positions_unprotected` alert has no analogue; replace with an exposure-vs-cap alert. |
| R25 | **Send a browser User-Agent from every HTTP client.** OKX's Cloudflare returns 403 `error code: 1010` for default library UAs — a client-fingerprint ban that is easily misread as a geo-block (F006). | Pack: keep for all HTTP clients; untested against Polymarket's edge. Never use UA/fingerprint changes to get around a geoblock (`04` line 114): if the geoblock check says blocked, stop. |
| R26 | **Vary the client before concluding anything about the network.** An HTTP failure that looks like geo-blocking may be a UA, TLS-fingerprint or header issue. | Same as R25. Polymarket geoblock is a real, documented condition, so "looks like geo-blocking" must first be checked against the venue's own geoblock answer before blaming the client. |

## 8. Not listed by the pack: recommended additions

The pack's "Do not port" list is R3, R7, R8, R59, R66, R62, R71. These ids are neither ported nor excluded:

| Id | Short gist (see ntx file for text) | Recommendation |
|---|---|---|
| R36 | risk metrics from the mark-to-market curve | **Port.** The pack already cites it (`05` line 97) but omits it from the port list. |
| R13 | update status + failure log before ending a session | **Port.** It is the rule behind the doc habits below. |
| R24 | a new strategy is a signal, not a system | **Port.** Owner plans several strategies (ARGUS-style first). |
| R28 | named SQL table constraints only | Port (journal is SQLite, `02` C9). |
| R29 | never delete the journal or reports | Port; adapt paths. |
| R44 | a backtest cannot validate execution plumbing | Adapt: paper cannot validate CLOB signing, partial fills, redemption; rehearse with minimum-size real orders before scaling. |
| R47 | deployed params in one registry used by gate and node | Port. |
| R68 / R69 | override may shrink, never stand in for funds; registry beats `.env` | Port. |
| R30, R33, R34, R45, R46, R52, R53, R67 | nautilus catalog, bar warm-up, `on_stop`, stop-metric specifics | Skip, or keep only the principle (R30: assert a filtered read returned what was asked). |

## 9. ntx doc-habit files

| File | Format (3 lines) |
|---|---|
| `docs/04-status.md` | Reverse-chronological log: one `## YYYY-MM-DD — <headline>` section per session, newest on top (1,033 lines now).<br>Each section: **Live:** state (deployed SHA, positions, health), outcome of the session's work with failure ids, ranked **Remaining** list (severity + id), **Owner steps** with exact commands.<br>Read at every session start (CLAUDE.md read order step 2); R13 makes updating it mandatory before a session ends. |
| `docs/05-failure-log.md` | Append newest first, never delete; ids `F###` (F002 ... F079; grouped ids like `F074..F079` allowed).<br>Template: `## F### — title`, then **Date / Phase / Severity**, **Symptom**, **Root cause**, **Fix**, **Prevention** (test/doc/rule added), **Cost** (time, money).<br>Severity scale: critical = money lost or could have been; high = blocked or wrong results; medium = wasted time; low = annoyance. Rules cite the F-id that taught them. |
| `docs/12-adr/ (README.md + NNNN-slug.md)` | One file per structural decision, `ADR-NNNN — <decision>`, header `**Status:** accepted · **Date:**`; README is an index table (ADR, Decision, Status).<br>Sections: Context, Decision, Rationale (numbered, with a worked example where numbers matter), Consequences, Revisit if.<br>Never edited after acceptance: a changed mind is a new ADR that supersedes (e.g. 0015 supersedes 0009; 0013 "partly superseded by 0014"). |
| `docs/26-phone-alerts.md` | Owner-facing setup + catalogue for ntfy pushes (`src/ntx/obs/alerts.py`, `deploy/ntx-alert.timer`); no-op until `NTX_ALERT_NTFY_URL` is set.<br>Sections: 1 What you have to do (4 steps; topic name treated as a password), 2 What arrives (tables: urgent incidents vs routine), 3 What is deliberately not in them, 4 If it goes quiet.<br>Each alert row = alert key + plain meaning + rule/F-id; dedup and repeat cadence follow R75. |
| `docs/27-open-rehearsals.md (also in read order)` | Tracks every change running on demo that must be promoted or reverted, so demo and live cannot drift silently.<br>One `## OPEN — <change>` section per item with a 2-column table: Started, owner, what to look at, decision + deadline.<br>An entry is deleted only on promote/revert, never for age; checked at session start with `04-status.md`. Polymarket analogue: paper-vs-live divergences. |

Code habits the pack also names (`05` line 121), for reference:

| File | Pattern |
|---|---|
| `src/ntx/obs/atomic.py` (67 lines) | Write: serialise -> temp file in the same dir -> parse-validate -> `os.replace()`. Read: on corruption delete the file and return None, never retry a broken state forever. |
| `src/ntx/ops/snapshot_check.py` (97 lines) | Deploy health = the node's 15 s state snapshot is fresh, connected, warm, protected; exit 0 healthy / 1 unhealthy (reasons on stdout) / 2 no snapshot. Exit 0 from `systemctl restart` is not health (R31). |

All paths relative to `[local source path omitted]`.
