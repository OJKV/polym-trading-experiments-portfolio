> Sanitized public copy of an earlier design input; the canonical specifications may supersede it.

# 03 — Strategy candidates for $50–100

> Research only. No strategy here has a backtest or a paper record. Every number belongs to
> the cited source; anything marked "derived" is arithmetic on a cited formula; anything
> marked "proposal" is a design choice for the owner.

Ranked for this project (LLM + small capital + free data):

| ID | Strategy | Uses LLM | Capital fit | Evidence | Priority |
|---|---|---|---|---|---|
| **P1** | Market-anchored LLM forecast on long-tail markets | core | good | moderate (literature), none of our own | **1: the thesis** |
| **P2** | Resolution-rules reading ("rules lawyer") near resolution | core | good | anecdotal + mixed bias studies | **2** |
| P3 | Neg-risk sum ≠ 1 scanner | none | fine | strong historically, fee-eroded now | 3: observe-only |
| P4 | Passive liquidity rewards | none | poor (cents/day) | venue-subsidised, no audited PnL | 4: optional learning |
| P5 | Near-resolution favourite (calibration bet) | optional | fine | **mixed sign** | 5: offline study only |

Common rules for every candidate:

- Positions are binary. Every stake is sized as if it goes to **0**: there is no stop order on Polymarket (N1 in `01`).
- Hold to resolution unless `02` §5 says exit.
- PnL is reported **after** taker fees, spread paid, and LLM cost.

---

## P1 — Market-anchored LLM forecast on long-tail markets

**Mechanism.** For shortlisted markets, the LLM ensemble estimates P(YES) without seeing the
price. Code calibrates that estimate on our own record and blends it with the market price.
We trade only when the blended q differs from the executable price by more than fee plus
slippage plus margin.

**Where to look.** The evidence says LLMs lose to *liquid* markets on their own ([L2]), so the
hypothesis is that the edge, if any, sits in markets where few informed traders look:

- low volume, but enough book depth for our 5-share minimum;
- resolution in **3–45 days**, so evidence accumulates within the owner's 2–3 month window (proposal);
- text-heavy rules;
- the **geopolitics** category is fee-free ("Geopolitical and world events markets are fee-free", https://docs.polymarket.com/trading/fees). That removes the largest cost at our size.

**Evidence for and against.**

- For: ForecastBench (2026-01-29) puts the best LLMs at Brier 0.102 and 0.017 behind superforecasters. Using the market price as an input improved one ensemble by "nearly 0.01" ([L1]). The AIA Forecaster plus market beats the market alone ([L2]).
- Against: KalshiBench found "systematic overconfidence across all models" and "only one model achieves a positive Brier Skill Score" ([L5]). In Alpha Arena, LLMs given full autonomy lost 30–63% (ARGUS F-015).
- Our own evidence: none.

**How to test on free data.** **Forward only.** Resolved-market backtests leak through
training data, retrieval and question selection ([L3]).

1. Phase F (forecast-only, no money). Every day, forecast the shortlist and freeze the forecast in the journal with its timestamp, model ids, prompt hash and evidence hashes. Record the executable ask, bid and depth at that moment from the CLOB.
2. When each market resolves, score Brier and log loss for three forecasts: (a) market mid at T, (b) raw LLM, (c) blended q.
3. Paper PnL: simulate taking at the recorded ask with the fee formula. Count a fill only if the recorded depth covered it.
4. Control: the same pipeline with the LLM replaced by **market mid + noise of the LLM's observed dispersion**. If the control earns the same, the "edge" is noise.

**Kill criteria** (proposal; the owner fixes the numbers before the data exists, as in ntx R10/R37):

- After the owner-set minimum of resolved forecasts (suggest ≥ 60), the blended forecast does **not** beat market mid on Brier, with the paired-bootstrap CI excluding 0. → Kill P1.
- Paper PnL after fees, spread and LLM cost is ≤ 0. → Kill.
- Calibration ECE of the blended q is worse than market mid's. → Kill.
- Three re-tuning cycles on the same forward data contaminate it; archive instead (ntx R10).

---

## P2 — Resolution-rules reading near resolution

**Mechanism.** Two variants, both about the *rules*, not the world.

- (a) **Resolved in substance.** The event has already happened per public, allowlisted sources, but the market still trades below about 0.97 because resolution is pending (proposal threshold).
- (b) **Title-versus-rules mismatch.** The market title suggests one thing; the rules text, clarifications and resolution source say another. Example shape: "by date X" measured in a specific time zone, or from a specific source.

The LLM's job is reading comprehension, not prediction. Its output is `RESOLVED_IN_SUBSTANCE`
or `AMBIGUOUS`, plus quoted rule clauses.

**Evidence.**

- Polymarket: "the market title describes the question, but the **rules** define how it resolves… Always read the resolution rules before trading" (https://docs.polymarket.com/concepts/resolution).
- Clarifications can be added "after trading begins" via the on-chain bulletin board (same page). That is a risk *and* a signal.
- The favourite-longshot evidence is mixed. On Polymarket, Nov 2022–Mar 2026, pooled buys at 90¢ or above earned 0.83% before fees, but the sign is fragile across weightings (Cardozo & Rivero-Wildemauwe, https://arxiv.org/html/2609.12878v1, via ntx research PM-3). A 4-week study found 65–95% prices overpriced one day out (https://github.com/shirleyshen0106/polymarket-calibration, not peer-reviewed).

**Risk that dominates.** The oracle.

- Resolution goes through UMA: a proposal bond "typically $750 pUSD", a "2-hour challenge period", and on a second dispute a DVM token-holder vote of "approximately 48 hours". A market can resolve 50-50 (https://docs.polymarket.com/concepts/resolution).
- A UMA whale vote resolved a $7M market to Yes with no deal signed, and refunds were refused (OddsShopper, https://www.oddsshopper.com/articles/prediction-markets/uma-oracle-polymarket-disputes, via ntx research [S14]).
- Variant (b) is *betting on how a disputed reading resolves*. Cap it hard (`04` §2).

**Test on free data.**

- Variant (a) can be partly backtested on Gamma resolved markets plus CLOB `prices-history`. Measure the price at T−24h, T−6h and T−1h against the outcome. This is a deterministic study and the LLM is not needed for it (the PM-3 design in ntx research).
- The LLM reading itself is forward-tested in Phase F, the same as P1.

**Kill criteria.**

- Forward hit-rate on `RESOLVED_IN_SUBSTANCE` calls below the owner-set minimum (suggest ≥ 98%, since each loss is about −100% against gains of about +3%; proposal).
- Any single loss from a misread rule triggers a mandatory review before the next trade.

---

## P3 — Neg-risk "sum ≠ 1" scanner (deterministic, observe-only)

**Mechanism.** In a multi-outcome (neg-risk) event exactly one YES pays $1. If the sum of best
YES asks plus fees is below 1, buying the full set locks in profit.

**Evidence.**

- Saguillo et al. estimate about $40M of realised arbitrage on Polymarket, Apr 2024–Apr 2025, and call it an upper bound (https://arxiv.org/abs/2508.03474).
- NBA markets, Feb–Mar 2026: 7 executable single-market episodes (median 3.6 s) and 290 combinatorial ones (median 101 bp; 76.9% averaged about 15 executable shares) (https://arxiv.org/html/2605.00864v1).
- Both via ntx research PM-2.

**Why priority 3.**

- Fees now apply per leg. Derived: a full set of 10 equal outcomes at rate 0.05 costs 0.05 × (1 − Σp²) = 4.5¢ per $1 set (ntx research).
- Episodes last seconds and are contested by faster bots.
- Legs are not atomic.
- Merge exists (Deposit Wallet split/merge/redeem, N1 in `01`), but it needs on-chain operations.

**Test.** Run the scanner on the live book for 4+ weeks and log every episode with its depth
and duration. It is free, needs no key, and uses no LLM. Treat any episode shorter than our
measured order round trip as lost.

**Kill.** If the episode log shows no fee-clearing gap that lasts longer than our round trip,
never trade it.

---

## P4 — Passive liquidity rewards (learning instrument)

**Mechanism.** Rest post-only GTD bids inside each market's reward band.

**Rules** (https://docs.polymarket.com/market-makers/liquidity-rewards):

- Score `S = ((v − s)/v)² · b`, where v is the max spread and s is the distance from the size-adjusted midpoint.
- Single-sided quotes score at one third when the midpoint is in [0.10, 0.90], and earn nothing outside that band.
- Paid daily at midnight UTC. "The minimum reward payout is $1."

**Fit.** Poor.

- ntx research measured, unrecorded: median `min_size` 20 shares and a median daily reward of about $3 per market.
- A two-sided 20-share quote near 50¢ ties up about $20, so $100 covers 4–5 markets. The share of rewards is cents per day, near the $1 floor.
- Adverse selection on news can erase days of rewards in one fill.
- It also needs a book-level engine (`01` §6), which this project deliberately does not build first.

**Kill.** Defer it. Revisit only above about $1–2k, and then with NautilusTrader 2.x.

---

## P5 — Near-resolution favourite (offline calibration study)

The deterministic twin of P2(a): buy 85–97¢ favourites in the last days before resolution.
The bias sign is contested ([S11]/[S13] above). Run it as an **offline study only**, with
IS = markets resolved before 2026-01-01 and OOS = Jan–Sep 2026, untouched until the end (ntx
R37). Use the free Gamma API plus `prices-history`.

Its result feeds P2's thresholds. It is not traded alone unless the OOS result is positive
after fees under all three weightings.

---

## Also note

- **Holding rewards.** Polymarket pays a variable annualised rate on positions in 18 listed events, "currently set at 3.25%", sampled hourly and paid daily (https://help.polymarket.com/en/articles/13364459-holding-rewards). It is a small carry on long-dated P1 positions in those events. Never trade *for* it.
- **Not proposed:** news-latency trading. Polymarket added dynamic taker fees in Jan 2026 to stop latency arbitrage (secondary source via ntx research [S16]), and a once-an-hour LLM loop cannot win a latency race anyway.

## Sources

`01` source list ([L1] ForecastBench / FRI, [L2] AIA, [L3] Paleka et al., [L5] KalshiBench, [N1] NautilusTrader docs), plus the URLs inline above, plus ntx
`docs/strategies/research/prediction-markets.md` (PM-1..3, sources S8–S16 there).
