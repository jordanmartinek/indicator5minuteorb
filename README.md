# NQ 5-Minute ORB Breakout Probability Engine — v2

A probability-based **forecasting engine** for TradingView (Pine Script **v6**) for the
first five-minute New York RTH **Opening Range (ORB)**. It is **not** a buy/sell
indicator. It estimates how likely a breakout is, which boundary is likely to break
*first*, whether the move is likely to continue or fail, and — honestly — **how
trustworthy those numbers are**.

- **File:** [`NQ_ORB_Breakout_Probability_Engine.pine`](./NQ_ORB_Breakout_Probability_Engine.pine)

> **Read this first.** The engine is defined on **confirmed 1-minute bars** and must be
> run on a **1-minute chart**. On any other timeframe the dashboard shows a
> `USE 1-MIN CHART` warning and **suppresses forecasts**, because the outcome labels
> would change meaning. This is deliberate. You can keep this on a dedicated 1-minute
> chart/layout tab for analysis while you execute on whatever timeframe you trade.
>
> The displayed probabilities are a **transparent heuristic and are NOT statistically
> validated**. Until you perform the calibration in [Phase 5](#phase-5--historical-calibration--validation),
> the dashboard labels them `UNCALIBRATED` and uses *score/estimate* wording. No trained
> coefficients, backtests, or accuracy claims are fabricated.

---

## 1. The forecasting contract

### Timestamp & modes
- The forecast operates on **each confirmed 1-minute bar after the ORB completes**.
- **Rolling mode** (default): re-estimated every confirmed 1-min bar.
- **Snapshot mode**: a single estimate frozen at ORB completion (09:35), never updated.

### The three pre-breakout outcomes (mutually exclusive & exhaustive, over horizon H)
| Outcome | Definition |
|---|---|
| **UPSIDE FIRST** | a confirmed 1-min close **above** ORB high occurs **before** any confirmed close below ORB low, **within H** |
| **DOWNSIDE FIRST** | a confirmed 1-min close **below** ORB low occurs first, **within H** |
| **NO BREAKOUT** | neither confirmed close occurs within H |

`P(UPSIDE) + P(DOWNSIDE) + P(NO BREAKOUT) = 100%`.
A **wick** beyond a boundary is **not** a break. If a single candle closes beyond **both**
boundaries, the bar is **AMBIGUOUS** and excluded from the calibration dataset.
A first confirmed close that lands **after H** is correctly labeled **NO BREAKOUT**, not
directional.

### The six probability concepts (kept separate)
A. P(any break within H) B. P(upside-first) C. P(downside-first) D. P(no breakout)
E. **Conditional** continuation after a confirmed break F. **Conditional** failure.
Directional *evidence* is reported separately from *event probability* — a high
upside-first share does **not** imply a high unconditional breakout probability if the
chance of any break is low.

---

## 2. Model mathematics (every displayed number defined)

### Features → evidence (uncalibrated)
Seven feature groups each produce a normalized score — signed in `[-1, 1]` (+bullish /
−bearish) or magnitude in `[0, 1]`:

1. **ORB boundary pressure** — normalized distance to each boundary, boundary-test counts,
   rejection depth, higher-low/lower-high structure; **persistent pressure** (shallow
   rejection + supportive structure) is separated from **repeated rejection** (deep
   rejection), so repeated tests alone do **not** raise breakout odds.
2. **Candle anatomy** — body %, wick %, close location, consecutive closes, body
   expansion, weakening countertrend wicks. Correlated cues are **combined inside the
   group** (not stacked as independent weights) to limit double-counting.
3. **Momentum & volatility** — ROC, momentum acceleration, ATR change, compression vs
   expansion, and an **exhaustion** flag (big move + contracting vol). Three regimes are
   distinguished: compression (energy+), healthy expansion (energy+), exhaustion (energy−).
4. **Relative volume** — current volume vs a **time-of-day baseline** (same intraday slot
   over the last *N* sessions, ~390-min RTH), plus up/down volume tilt. Degrades to a short
   SMA when multi-session history is thin; shows `N/A` when volume is unavailable.
5. **VWAP & structure** — side of VWAP, slope, crossings, HH/HL vs LH/LL (confirmed
   pivots). **Chop** requires corroboration: many VWAP crossings **and** high candle
   overlap — one crossing does not label the market choppy.
6. **Room-to-travel** — distance from each boundary to the nearest opposing proxy level
   (PMH/PML, ON H/L, PDH/PDL/PDC, confirmed swing pivots, user levels). Proxies only —
   **not** resting liquidity.
7. **Session conditions** — time since ORB, width, early volatility, optional news blackout.

### Evidence → probability simplex
```
D   = Σ wᵢ·featureᵢ              (net directional evidence, clamped to [-3, 3])
pUp = sigmoid(k · D / 3)          pDn = 1 − pUp          (directional split)
E   = clamp( energy · (0.4 + 0.6·timeLeft) · (1 − 0.4·chop) + 0.1·relVolScore , 0, 0.98 )
P(UPSIDE)   = E · pUp
P(DOWNSIDE) = E · pDn             → renormalized to sum to 100%
P(NO BREAK) = 1 − E
Breakout potential = 100% − P(NO BREAK)
```
The logistic maps evidence to a (0,1) split; it does **not** prove those numbers match
observed frequencies — that is what calibration is for.

### Post-breakout (its own labels)
- **Continuation**: price reaches `boundary ± contATR·ATR` (configurable) **after** the
  break bar, before the failure condition and before the post-breakout horizon.
- **Failure**: price meets the return-inside condition (confirmed close back inside the
  ORB by default, or a stricter touch) before reaching the continuation target.
- **Unresolved**: neither occurs before the horizon → labeled `UNRESOLVED` (not forced).
- **Ambiguous**: both occur on the same candle → excluded from calibration.
Continuation/failure also have supporting **uncalibrated evidence scores** for the dashboard.

---

## 3. Install & recommended NQ settings (1-minute)

1. Open a **1-minute** chart (e.g. `CME_MINI:NQ1!`).
2. Pine Editor → paste [`NQ_ORB_Breakout_Probability_Engine.pine`](./NQ_ORB_Breakout_Probability_Engine.pine) → **Save** → **Add to chart**.

| Input | Default |
|---|---|
| ORB window / timezone | `0930-0935` / `America/New_York` |
| Forecast mode / horizon | Rolling / 15 min |
| ATR / logistic k | 14 / 3.0 |
| Touch tol / test lookback | 0.15×width / 20 |
| Rel-vol baseline | 20 sessions |
| Post-breakout horizon / target | 15 min / 1.0×ATR |
| Min-dwell (anti-flicker) | 2 bars |
| Calibration status / map | Uncalibrated / off |

Weights: pressure 1.0, candle 1.0, momentum 1.0, VWAP 0.8, room 0.6.

---

## 4. Dashboard & market states

**Rows:** symbol·timeframe·horizon, timeframe warning (if not 1-min), **State**, **Mode**,
**Calibration status**, ORB high/low, width, price location, **Breakout potential**,
**Upside-first**, **Downside-first**, **No breakout**, **Directional evidence**,
**Continuation**, **Failure risk**, VWAP alignment, Volatility, Relative volume, Chop, Note.
Calibrated probabilities show `%`; uncalibrated outputs show `(score)` — they are never
mixed to look equivalent. Colors: green = bullish, red = bearish, amber = uncertainty /
failure risk, gray = range / insufficient.

**States (explicit entry/exit):**
`ORB FORMING → PRE-BREAKOUT → RANGE / NO EDGE ↔ BREAKOUT CONFIRMED → CONTINUATION DEVELOPING | FAILED BREAKOUT`, plus `INSUFFICIENT DATA`.
`RANGE / NO EDGE` is entered when evidence is weak/contradictory/choppy or the horizon
elapses without a break. Event-critical states commit immediately; soft states respect a
min-dwell to avoid per-candle flicker.

**Chart:** anchored ORB high/low (drawn with `line.new`, so they stay pinned on pan/zoom),
optional midpoint/VWAP/PM/ON/PD levels, small confirmed breakout / failed-breakout / retest
markers. Historical markers do not move when new bars arrive.

---

## 5. Alerts
Breakout potential / upside-first / downside-first crossing thresholds; confirmed close
above/below the ORB; failed breakout; successful retest; deteriorating continuation; and
entering/leaving `RANGE / NO EDGE`. All are **1-minute-gated** and fire once per transition
unless *repeated alerts* is enabled. Intrabar alerts are a clearly-labeled opt-in. Messages
include symbol, state, horizon, ORB boundaries, and the relevant score/probability.

---

## Phase 5 — historical calibration & validation

Pine does the **live** features/dashboard/plots/alerts. Robust statistical calibration is
done **externally**; the indicator exposes the data you need.

### A. Export (in TradingView)
Enable **Research Export → "Plot machine-readable export series"**. This adds, in the Data
Window / exported CSV, per bar:
`EXP_pUp, EXP_pDn, EXP_pNone, EXP_dirEv, EXP_contScore, EXP_failScore, EXP_preOutcome, EXP_postOutcome`.
`EXP_preOutcome` is emitted **only when the outcome is resolved** (after H): `1` upside-first,
`-1` downside-first, `0` no-breakout, `8` ambiguous. `EXP_postOutcome`: `1` continuation,
`-1` failure, `9` unresolved, `8` ambiguous. Use *Export chart data* (or alert/webhook logging)
across as many NQ 1-min sessions as available (target several hundred+).

### B. Calibrate (external: Python/R)
1. Join each forecast row to its **realized** `EXP_preOutcome` label.
2. **Chronological split**: early = develop, middle = calibrate, later untouched = test.
   Use walk-forward where feasible. **Never randomly mix** past/future.
3. Group observations **by session** (overlapping rolling forecasts are correlated — do not
   treat them as independent samples).
4. Fit a reference first (historical base rates, a simple directional model), then a
   **multinomial logistic regression** on the three outcomes. Only adopt complexity that
   beats the reference **out-of-sample**.
5. Build the probability map and read off empirical hit-rates for score buckets
   0/25/50/75/100.

### C. Evaluate (out-of-sample)
Brier score, log loss, reliability diagram, accuracy by probability bucket, up/down/no-break
frequencies, false-breakout rate, continuation/failure rates, MFE/MAE, and performance by
**volatility regime** and **time of day**. Any profitability statement must include
commissions, slippage, and the full outcome distribution — predicting direction is not profit.

### D. Load back in
Enter the bucket hit-rates in the **Probability Calibration Map** inputs, toggle **Apply
calibration map**, and set **Calibration status** to *Calibrated – limited data* or
*Calibrated – validation available* per your evidence. The dashboard then reports
`CALIBRATED` and switches scores to `%`. If you can't complete this, leave it `Uncalibrated`.

> The `Calibration status` input is an **honesty flag you set** — it is not computed and does
> not itself validate anything. Do not set it to *validated* without out-of-sample evidence.

---

## Phase 7 — non-repainting protections
Confirmed-bar logic by default (`barstate.isconfirmed`); breakouts are confirmed **closes**,
never wicks; `request.security` daily levels use `lookahead_off` on prior-bar values;
structure uses **lagged** symmetric pivots; per-session state resets on the first ORB bar;
divisions guarded; probabilities clamped & renormalized; graceful handling of missing volume,
insufficient history, and non-1-min timeframes. A historical forecast reflects only
information available at its timestamp.

---

## Tests performed (honest status)
**Performed (static / structural):** Pine v6 structural validation (balanced
parentheses/brackets, no tabs, no `ta.*`/`math.sum` inside ternary branches, `plotshape`
locations are constants, no `na` booleans, no global mutation inside functions); two
behavioral review passes incorporated (three-outcome horizon bound, latched NO-BREAKOUT
resolution, continuation requires post-break follow-through, explicit 1-min alert gating,
simplex-degeneracy fallback, time-of-day volume baseline, stale-level expiry, anchored
non-floating ORB lines).

**NOT yet performed (requires the external workflow above):** empirical calibration; Brier/
log-loss/reliability; out-of-sample accuracy; regime/time-of-day slicing; MFE/MAE; feature
ablation. **No accuracy or profitability claims are made.** The dashboard reports
`UNCALIBRATED` until you complete Phase 5.

---

## Known limitations & roadmap
**Limitations:** heuristic until calibrated; 1-minute-native by design; TradingView volume ≠
order flow; "liquidity" = price-structure proxies; one ORB per session; RTH assumed ~390 min.

**Prioritized roadmap:**
1. Complete the external calibration pipeline (highest value).
2. Multinomial-logistic coefficients transferred into the calibration map; versioned.
3. Feature ablation to drop non-contributing groups and reduce double-counting.
4. Regime-conditional weights / logistic `k` once calibrated.
5. Optional ES confirmation for NQ — only if it improves out-of-sample results.
6. Calendar-exact same-timestamp volume baseline (vs the current ~390-min approximation).

*This tool is for analysis and education. It is not financial advice.*
