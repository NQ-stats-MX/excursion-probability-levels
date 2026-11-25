# Excursion Probability Levels

A TradingView indicator (Pine Script v6) that draws intraday levels based on **how likely price is to touch them before the session ends**, rather than where price is likely to close.

Most volatility bands (standard-deviation bands, expected-move lines) answer the question *"where will price finish?"*. Intraday traders usually care about something else: *"will price get there at some point today?"* Those two probabilities are not the same. A level that price has a 25% chance of **closing** beyond has roughly a 50% chance of being **touched** at some point in the session. This indicator draws levels from the second probability.

<img width="1835" height="1004" alt="image" src="https://github.com/user-attachments/assets/23e73d15-85b3-4836-b84e-71551f1a00a0" />

---

## Features

- **Anchored levels**: three pairs of levels (above and below) set at the session open, each with a chosen touch probability (defaults 50% / 25% / 10%). They don't move during the session.
- **Session open line**: a reference line at the open price.
- **Mean excursion levels**: the expected maximum move away from the open on each side (≈ 0.80σ).
- **Rolling cone** *(optional)*: the same probabilities recomputed every bar from the current price and the volatility still left in the session. It narrows into the close.
- **Live touch labels**: at each anchored level, the current probability that it still gets touched, or `hit` if it already has been.
- **Session high/low take-out**: the probability that the current session high or low gets exceeded before the close.
- **Calibration table**: how often each level was actually touched over the sessions on your chart, next to the probability the model gave it. This lets you check the model against real data.

---

## How it works

### 1. First-passage probability

Treat log price as a driftless diffusion with $V$ variance remaining in the session. The reflection principle gives the probability of touching a level at log-distance $d$ before the session ends:

$$
P(\text{touch}) = 2\left(1 - \Phi\!\left(\frac{d}{\sqrt{V}}\right)\right)
$$

Inverting this, the level with touch probability $p$ sits at

$$
d = \Phi^{-1}\!\left(1 - \tfrac{p}{2}\right)\sqrt{V}
$$

and is drawn at $\text{open} \cdot e^{\pm d}$.

| Touch probability | Distance (in session σ) |
|---|---|
| 50% | 0.67σ |
| 42.5% (mean excursion) | 0.80σ |
| 25% | 1.15σ |
| 10% | 1.64σ |

### 2. The volatility clock

Intraday volatility isn't spread evenly. The open and the close are much more active than midday. So the indicator doesn't scale variance with clock time. It learns an **intraday variance profile**:

- Each bar's variance is estimated with the **Parkinson high-low estimator**: $\sigma^2_{\text{bar}} = \frac{(\ln H/L)^2}{4\ln 2}$
- Variance is stored **per bar position in the session** (bar 0, bar 1, …) and averaged across sessions with an exponentially weighted mean (span = *Profile memory*).
- The remaining variance at any point is the sum of the profile over the bars still to come.

### 3. Implied-vol scaling (optional, on by default)

With **Scale by implied vol** on, the profile is stored per unit of implied variance (`VOLI²`) and multiplied by today's `VOLI²`:

- the **shape** of the day comes from history
- the **level** comes from today's implied vol

The levels react immediately to a change in volatility regime, and the indicator gives usable levels after only a couple of sessions of history.

---

## Installation

1. Open TradingView → **Pine Editor**.
2. Paste the contents of `ExcursionProbabilityLevels.pine`.
3. Click **Add to chart**.
4. Use an **intraday** timeframe (1–15 min works well) on a symbol whose chart **includes bars outside the session**, e.g. NQ/MNQ with extended hours turned on. The script detects the session start and end from those bars.

The indicator needs at least **2 completed sessions** of chart history before it draws levels.

---

## Inputs

### Session
| Input | Default | Description |
|---|---|---|
| Session | `0930-1600` | The session whose excursions are measured. |
| Timezone | `America/New_York` | Timezone of the session string. |

### Touch probabilities
| Input | Default | Description |
|---|---|---|
| Level 1 / 2 / 3 (%) | 50 / 25 / 10 | One-sided probability of each level being touched before the session ends. 25% means the upper level gets touched in roughly 1 session out of 4. |

### Volatility
| Input | Default | Description |
|---|---|---|
| Scale by implied vol | On | Use implied vol for the level and history for the intraday shape. |
| Implied vol symbol | `NASDAQ:VOLI` | Implied vol index matching your instrument (see below). |
| Profile memory (sessions) | 20 | EWMA span of the intraday variance profile. Lower values adapt faster but are noisier. |

### Display
Toggles for each component (anchored levels, cone, live labels, high/low take-out, calibration table, open line, mean excursion) and their colors.

---

## Reading the chart

- **Anchored levels at session open**: the `25% ↑` line means *"at the open, there was a 25% chance price would trade up to here before the close."*
- **Live labels**: `25% ↑  now 8%` means the level was a 25% touch at the open, and given where price is now and how much session is left, it is now 8%. When price reaches the level, the label switches to `hit`.
- **Session H taken: 31%** means a 31% chance that the current high of day gets exceeded before the close.
- **Rolling cone**: the levels you would draw if the session started now. It is useful late in the day, when the anchored levels are far from price.
- **Calibration table**: if the model is well calibrated, the `Up hit` and `Down hit` columns should land close to the `Model p` column. Hit rates far above the model suggest it is underestimating volatility. Persistent asymmetry between up and down suggests drift or a trending regime.

---

## Using it on other instruments

The implied-vol symbol should match the underlying:

| Instrument | Implied vol symbol |
|---|---|
| NQ / MNQ (Nasdaq-100) | `NASDAQ:VOLI` |
| ES / MES (S&P 500) | `CBOE:VIX` |
| RTY (Russell 2000) | `CBOE:RVX` |

If there is no matching index, turn **Scale by implied vol** off and the indicator will use the purely realized profile.

---

## Assumptions and limitations

This is a model and shouldn't be used as a signal by itself.

- **Driftless diffusion**: there are no jumps, no fat tails and no trend. Real markets have news spikes and trend days, so the low-probability levels (10% and below) are usually hit more often than the model says. Use the calibration table to check this.
- **Parkinson estimator**: it ignores gaps between bars and tends to slightly underestimate variance on discretely sampled bars.
- **Bar-indexed profile**: the profile is keyed by bar position, so changing the timeframe rebuilds it from scratch. Half-day sessions also distort it slightly.
- **Calibration sample**: the calibration table only covers the sessions loaded on your chart, so higher timeframes or less history mean fewer sessions and noisier hit rates.
- **Symmetric levels**: up and down levels are the same distance from the open in log terms.

---

## License

MIT. See [LICENSE](LICENSE).

Author: **Aidan**
