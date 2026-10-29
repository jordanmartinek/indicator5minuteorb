# NQ 5-Minute ORB Breakout Probability Engine

A **probability-based analytical engine** for TradingView (Pine Script **v6**) that
estimates — in real time and on history, **without repainting confirmed signals** —
how a session is likely to break out of its first five-minute Opening Range (ORB).

It is built for **Nasdaq futures (NQ)** on a **1-minute chart**, and works on ES and
other liquid futures. It is **not** a buy/sell indicator: it quantifies the *evidence*
for a breakout, estimates *first-passage direction*, and flags conditions associated
with *chop* and *failed breakouts*.

> **Honesty first.** The probabilities shipped here are a **transparent heuristic**.
> Until you supply a calibration map derived from out-of-sample NQ data (see
> [Validation](#6-validation-procedure)), the dashboard clearly labels them
> **`UNCALIBRATED (heuristic)`**. Do not treat a heuristic score as a statistically
> validated probability.

- **File:** [`NQ_ORB_Breakout_Probability_Engine.pine`](./NQ_ORB_Breakout_Probability_Engine.pine)

---

## 1. What it answers

1. **Is a breakout becoming more likely?** → *Breakout likelihood* and the per-feature evidence.
2. **Which boundary breaks first?** → *UP-first* / *DOWN-first* / *NEITHER* first-passage probabilities.
3. **Is the move real?** → *Continuation* score (displacement, volume, follow-through, VWAP, room).
4. **Is this chop / a fake-out?** → *Failed-breakout risk* and a *RANGE / NO EDGE* state.
5. **How reliable are the numbers?** → an explicit `CALIBRATED` vs `UNCALIBRATED` tag.

---

## 2. The model (concise)

### Event definitions (precise, non-repainting)

- **ORB window** — bars inside the session `09:30–09:35 America/New_York`. On a 1-min
  chart this is the single 09:30 candle. The high/low are **frozen** when the window ends.
  DST is handled automatically by the named timezone.
- **Breakout (default)** — a **confirmed 1-min close** strictly beyond an ORB boundary,
  evaluated only on `barstate.isconfirmed`. A **wick** beyond the range is tracked as a
  *probe* and is **never** treated as a breakout.
- **First-passage outcomes** over the horizon *H* ∈ {5, 15, 30} min after the ORB:
  - `UP_FIRST` — upper boundary closed through first, within H.
  - `DOWN_FIRST` — lower boundary closed through first, within H.
  - `NEITHER` — no confirmed close outside either boundary within H.
  These are **mutually exclusive and exhaustive → they sum to 100%**.
  *Overall breakout likelihood = 100% − P(NEITHER).*

### Why this is not "RSI+MACD = probability"

The engine keeps four concepts **strictly separate**:

1. **Evidence** — each feature group produces a normalized score in `[-1, 1]` (signed)
   or `[0, 1]` (magnitude). These are explicitly *heuristic*.
2. **Direction** — net directional evidence `D` is mapped to UP vs DOWN by a **logistic**
   (`pUp = sigmoid(k·D)`), producing a proper two-way split.
3. **Volatility / compression / energy** — a separate `E ∈ [0, 1]` "escape energy"
   term governs *whether any* break is likely, scaled by time remaining in the horizon.
   Compression that precedes a break is distinguished from **exhaustion** that precedes
   a reversal; compression alone is never directional evidence.
4. **Failure risk** — a separate post-breakout score.

The first-passage simplex is then:

```
P(UP_FIRST)  = E · pUp
P(DOWN_FIRST)= E · pDn            (renormalized to sum to 100%)
P(NEITHER)   = 1 − E
```

### Stage A — pre-breakout (feature groups)

| Group | What it measures | Signed? |
|---|---|---|
| 1. Boundary pressure | tests of each boundary, rejection depth, higher-low/lower-high, persistent pressure **vs** repeated rejection | signed |
| 2. Candle price action | body %, wick %, close location, consecutive closes, body expansion, weakening opposing wicks | signed |
| 3. Momentum & volatility | ROC, momentum acceleration, ATR change, compression vs expansion, **exhaustion** detection | signed + energy |
| 4. Relative volume | **time-of-day** baseline (same slot over N sessions), up/down volume tilt | magnitude + tilt |
| 5. VWAP & structure | side of VWAP, VWAP slope, repeated crossings → chop, HH/HL vs LH/LL (confirmed pivots) | signed |
| 6. Liquidity / levels | room-to-travel to PMH/PML, ON H/L, PDH/PDL/PDC, swing pivots, user levels | signed |
| 7. Session conditions | time elapsed, ORB width, early volatility, optional news blackout | context |

### Stage B — post-breakout

Once a confirmed close occurs outside the ORB, the engine switches to **breakout-quality**:

- **Continuation score** — breakout body %, displacement beyond range (×ORB width),
  follow-through candle count, breakout relative volume, VWAP alignment, room-to-travel,
  retest success.
- **Failure risk** — close back inside the range, rejection wick against the break, weak
  relative volume, absent follow-through, VWAP contradiction, no room to travel.

### State machine

`ORB FORMING → PRE-BREAKOUT → BREAKOUT CONFIRMED → CONTINUATION DEVELOPING | FAILED BREAKOUT`,
plus `RANGE / NO EDGE`. Soft states use a configurable **min-dwell** to avoid flicker;
**event-critical** states (breakout confirmed, continuation, failed breakout) commit
**immediately** so a fast failure is never hidden.

---

## 3. Install on TradingView

1. Open a **1-minute** chart of your instrument (e.g. `CME_MINI:NQ1!`).
2. Open **Pine Editor** (bottom panel) → **Open** → *(or)* paste the full contents of
   [`NQ_ORB_Breakout_Probability_Engine.pine`](./NQ_ORB_Breakout_Probability_Engine.pine).
3. Click **Save**, then **Add to chart**.
4. (Optional) Set chart timezone to *Exchange* or *America/New York* for visual alignment —
   the engine's session logic already uses `America/New_York` internally regardless of the
   chart's display timezone.

The script auto-scales to other chart timeframes by converting the horizon from minutes
to bars, but it is **designed and tuned for 1-minute**.

---

## 4. Default settings (NQ, 1-minute)

| Input | Default | Notes |
|---|---|---|
| ORB window | `0930-0935` | first 5-min NY RTH candle |
| Session timezone | `America/New_York` | DST-safe |
| Horizon | `15 min` | options 5 / 15 / 30 |
| ATR length | `14` | volatility normalization |
| Logistic steepness `k` | `3.0` | sharpness of UP↔DOWN mapping |
| Boundary-test lookback | `20` bars | |
| Touch tolerance | `0.15` × ORB width | |
| Rel-volume baseline | `20` sessions | time-of-day baseline |
| Swing pivot length | `5` | confirmed (lagged) pivots |
| Follow-through window | `3` bars | |
| State min-dwell | `2` bars | anti-flicker for soft states |
| Calibration map | **off (identity)** | → `UNCALIBRATED (heuristic)` |
| Intrabar alerts | **off** | confirmed-bar logic only |
| Repeated alerts | **off** | one alert per transition |

Weights (directional evidence): pressure `1.0`, candle `1.0`, momentum `1.0`,
VWAP `0.8`, levels `0.6`.

---

## 5. Dashboard outputs

| Row | Meaning | Color logic |
|---|---|---|
| **State** | current market state (see state machine) | green = breakout/continuation, orange = failed, gray = range, blue = forming/pre |
| **Probabilities** | `CALIBRATED` or `UNCALIBRATED (heuristic)` | yellow until you calibrate |
| **ORB High / Low** | frozen opening-range boundaries | neutral |
| **ORB Width (pts)** | range size in points | neutral |
| **Price location** | ABOVE / INSIDE / BELOW the range | neutral |
| **UP-first P** | P(upper boundary breaks first) | green if dominant |
| **DOWN-first P** | P(lower boundary breaks first) | red if dominant |
| **NEITHER P** | P(no break within horizon) | gray |
| **Breakout likelihood** | 100% − P(NEITHER) | green ≥ threshold |
| **Continuation** | post-breakout quality (score, or % if calibrated) | green ≥ threshold, else yellow |
| **Failed-breakout risk** | post-breakout failure risk (score, or %) | orange ≥ threshold |
| **VWAP alignment** | BULLISH / BEARISH / MIXED | by side + slope |
| **Volatility** | EXPANDING / COMPRESSING / NORMAL | |
| **Relative volume** | HIGH / ABOVE AVG / LOW / N/A | green/red by level |
| **Note** | `NEWS BLACKOUT`, `Volume unavailable`, or `Heuristic model` | |

Color key: **green** = bullish evidence, **red** = bearish evidence,
**yellow/amber** = uncertainty / elevated failure risk, **gray** = range / insufficient evidence.
Use *Minimal dashboard* to collapse to the core rows, and choose any corner position.

### Chart plots (each toggleable)
ORB high, ORB low, optional ORB midpoint, session VWAP, optional PM/ON/PD levels,
small confirmed **breakout** markers, **failed-breakout** markers, and optional **retest** markers.

---

## 6. Validation procedure

**The indicator ships UNCALIBRATED.** Here is the repeatable workflow to validate and
calibrate it against real NQ sessions, out-of-sample and in chronological order.

### A. Define outcomes (already matches the engine)
- Pre-breakout: for each session, label `UP_FIRST` / `DOWN_FIRST` / `NEITHER` within the horizon.
- Post-breakout: label **successful continuation** vs **failed breakout** using documented
  thresholds (e.g. continuation = price advances ≥ 1 ORB width beyond the boundary before
  returning inside; failure = confirmed close back inside within N bars).

### B. Export the heuristic scores
On a 1-minute NQ chart, read, per session, at the moment the horizon window opens:
`Breakout likelihood`, `UP-first P`, `DOWN-first P`, `Continuation`, `Failed-breakout risk`.
Export via TradingView alerts/webhooks or by stepping the replay and recording the dashboard
values (or add temporary `plot()`/`label` lines and use *Export chart data*).

### C. Build the calibration map (external)
With the exported `(heuristic_score, realized_outcome)` pairs:
1. **Split chronologically** — earlier sessions = fit, later sessions = out-of-sample test.
   **Never randomly mix** future and past observations.
2. Bucket scores (e.g. 0–25, 25–50, 50–75, 75–100) and compute the **empirical hit rate**
   per bucket on the fit set.
3. Those empirical rates become the calibration knot `y`-values.

### D. Evaluate (on the out-of-sample set)
- **Brier score** and **log loss** (lower = better).
- **Reliability diagram** / probability-bucket calibration table (predicted vs realized).
- **Accuracy by probability bucket** and **false-breakout rate**.
- Slice by **volatility regime** and **time of day**.
- **MFE / MAE** after breakout; account for **transaction costs & slippage**.
- Treat overlapping prediction windows and multiple observations from the same session as
  **correlated** — do not count them as independent samples.

### E. Load the calibration back in
Enter the per-bucket rates in the **Probability Calibration Map** inputs
(`Calibrated P at score=0/25/50/75/100`) and toggle **Enable calibration map** on.
The dashboard flips to `CALIBRATED`. The piecewise-linear map is applied to the breakout,
continuation, and failure scores; the directional split is preserved and renormalized.

> If you cannot complete calibration, leave it off. The dashboard will keep telling the
> truth: **UNCALIBRATED (heuristic)**.

---

## 7. Non-repainting guarantees

- Model state, breakouts, and state transitions use **confirmed bars** by default
  (`barstate.isconfirmed`). An opt-in *Intrabar alerts* switch is clearly labeled as
  potentially repainting alert timing.
- Breakouts are **confirmed closes**, never wicks.
- `request.security` (previous-day H/L/C) uses `lookahead=barmerge.lookahead_off` on
  prior-bar values → no future leakage.
- Structure uses **symmetric, lagged** pivots that only resolve after the fact → no
  forward peek in live predictions.
- Per-session state (ORB, breakout flags, retest) resets on the first ORB bar of each session.
- All divisions are guarded; probabilities are clamped and renormalized; volume absence is
  handled gracefully.

---

## 8. Known limitations

- **Heuristic until calibrated.** The shipped probabilities are not statistically validated.
- **Volume is not order flow.** TradingView volume ≠ true bid/ask flow; relative volume uses
  a time-of-day baseline and degrades to a short SMA when multi-session history is thin, and
  shows `N/A` when volume is unavailable (common on some FX/index feeds).
- **"Liquidity" is a proxy.** PM/ON/PD levels and swing pivots are **price-level proxies**,
  not real resting institutional orders or a full order book.
- **One ORB per session.** Designed for a single RTH opening range per day.
- **Timeframe.** Tuned for 1-minute; other timeframes auto-scale the horizon but are not tuned.
- **Session assumptions.** Rel-volume baseline assumes a ~390-minute RTH; adjust for products
  with different session lengths.

---

## 9. Prioritized remaining work

1. **Calibration data pipeline** — the single highest-value item; follow Section 6 to turn
   heuristic scores into validated probabilities.
2. **True same-minute volume baseline** — the current time-of-day baseline approximates the
   session as 390 minutes; a calendar-exact same-timestamp baseline would be more precise.
3. **Regime-conditional weights** — allow the feature weights / logistic `k` to vary by
   volatility regime once calibration data exists.
4. **Multi-timeframe confirmation** — optional higher-timeframe trend context behind a
   `lookahead_off` request.
5. **Automated outcome logging** — a companion export mode (plots/labels) to make Section 6
   data collection one click.

---

*This tool is for analysis and education. It is not financial advice.*
