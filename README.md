# Cumulative Delta / Volume Ratio (Pine v6)

`cumdelta_volume_ratio.pine` plots `cumulative delta / cumulative volume`, a
bounded `[-1, +1]` read on how one-sided the traded volume has been since the
anchor (session / day / week / month, or running).

## Why the timeframe ladder

Delta is not available directly, it is reconstructed by classifying **intrabars**
as buy- or sell-side. The coarser the intrabar, the more the classification is
distorted: a 4h candle that closes up gets its entire volume signed positive,
including the sell-side volume inside it. So the script asks for the finest
intrabars it can afford and walks this ladder in order:

```
1s → 1m → 5m → 15m → 1h → 4h → 1d
```

It takes the **first (finest) rung strictly below the chart timeframe that fits
the intrabar budget** (`Max intrabars per chart bar`, default 600). Every rung is
an exact multiple of the one before it, so intrabars tile the parent bar with no
spillover; the table reports `Tiling: exact` when the chosen rung also divides
the chart timeframe evenly.

Why a budget at all: `request.security_lower_tf()` returns at most ~100,000
intrabars in total, so history covered ≈ `100000 / intrabars-per-bar`. Raising the
cap buys resolution and costs history.

Selection with the default cap of 600:

| Chart TF | Rung chosen | Intrabars/bar | ≈ bars covered |
|---|---|---|---|
| 1m  | 1s  | 60  | 1600 |
| 5m  | 1s  | 300 | 330  |
| 15m | 1m  | 15  | 6600 |
| 1h  | 1m  | 60  | 1600 |
| 4h  | 1m  | 240 | 410  |
| 1D  | 5m  | 288 | 340  |
| 1W  | 1h  | 168 | 590  |

A 1s chart has no finer rung, so it signs the chart bar itself. If the finer rung
has no history for a bar (common with 1s data, which is retained only briefly),
the script falls back one rung for that bar and says so in the table.

## Classification methods

- **Tick rule** (default) — sign of each intrabar's close-to-close change,
  evaluated in the lower-timeframe context so `close[1]` is the previous
  intrabar, including across parent-bar boundaries. Falls back to candle
  polarity, then to range position, when flat.
- **Candle polarity** — close vs open of the intrabar (what TradingView's own
  volume-delta code does).
- **Proportional** — volume split by where the close sits inside the intrabar
  range; no ties, smoothest output.

None of these is true bid/ask delta — that needs tick data TradingView does not
expose to Pine. At 1s the approximation is close; treat coarser rungs as
directional, not exact.

## Why the ratio used to pin at ±1

`ratio = cumDelta / cumVolume` reaches exactly ±1 only when every sample in the
anchor period is signed the same way **and** each sample's `|delta|` equals its
full volume. Two places in the original code guaranteed that, and both cluster at
the point where intrabar history runs out — which is exactly where the rung
switch happens:

1. **Single fully-signed sample.** On the last-resort fallback the chart bar was
   signed by polarity, so `delta = ±volume` by construction. One such bar is
   enough for `|ratio| = 1`.
2. **No sample-size floor.** On the first bar after each anchor reset, `cumVolume`
   is one bar of volume, so the cumulative ratio *is* that bar's ratio. Combined
   with (1) it prints a clean ±1 spike at every reset in the degraded region.

Three changes remove it:

- The chart-bar fallback is now **always** a proportional split of the bar's
  range, never a full sign, so a single sample can no longer contribute
  `|delta| = volume` unless the bar closes exactly on its extreme.
- `Min bars before plotting` (default 3) blanks the plot until the period has
  enough bars for the ratio to mean anything. It counts chart bars, not intrabars,
  so the guard is equally strict on a fine rung and a coarse one.
- `Blank out bars with no intrabar data` blanks and shades any bar not served by
  the chosen rung, so the switch point is visible as a gap instead of a spike.
  `Reset cumulation when the intrabar source changes` keeps one cumulative sum
  from mixing two rungs' delta magnitudes.

Shading: amber = served by the coarser fallback rung, deeper orange = bar-level
estimate only, grey = warming up after an anchor reset. Alerts fire off the gated
series, so they stay silent wherever the plot is blank.

**Amplitude is not comparable across regimes.** A coarse sample signs its whole
volume in one direction, so less cancels internally: 218 one-minute intrabars in a
4h bar largely offset each other, while a single bar-level estimate cannot offset
anything. The same order flow therefore reads as a larger |ratio| on a coarser
rung. Do not compare swing sizes across a shading boundary — only within one
regime.

## Why the plot starts partway into history

`request.security_lower_tf()` serves roughly 100,000 intrabars in total, counted
back from the right edge, so coverage in chart bars is about
`100000 / intrabars-per-bar`. Older bars come back with empty arrays — there is no
intrabar data there, so there is no delta to compute, and with
`Blank out bars with no intrabar data` on (the default) the line simply stops.
That boundary is the same place the ±1 spikes used to print.

The table's **Intrabar data from** row gives the exact date the coverage starts and
how many bars it spans, and **Budget** shows the estimate for the current rung. If
the price chart has bars before that date but the indicator does not, this is why —
not a plan history limit.

Levers, in order of usefulness:

1. **Lower `Max intrabars per chart bar`.** This climbs the ladder to a coarser
   rung, which covers far more history. The tradeoff on a 4h chart:

   | Cap range | Rung | Intrabars/bar | Coverage |
   |---|---|---|---|
   | ≥ 240  | 1m  | 240 (~218 on NQ) | ~460 bars (~3.5 months) |
   | 48–239 | 5m  | 48 (~46)  | ~2,170 bars (~1.4 years) |
   | 16–47  | 15m | 16        | ~6,250 bars (~2.8 years) |
   | 4–15   | 1h  | 4         | beyond most chart history |

   The rung flips at the cap value where `chartTF / rung` stops fitting, so cap 60
   still selects 5m (48 ≤ 60), not 15m — you need cap 47 or lower for that. 4h
   itself is never a candidate, since a rung must be strictly below the chart
   timeframe.

   Fewer intrabars per bar is not simply worse: a uniform coarse rung across the
   whole chart is often more useful than a fine rung spliced onto a coarse one,
   because amplitude is only comparable within a regime.
2. **Uncheck `Blank out bars with no intrabar data`.** Older bars then plot from
   the coarser fallback rung, or from the bar-level proportional estimate, shaded
   orange so you can see it is an estimate. It no longer pins at ±1.
3. **Expect 1s coverage to be short.** Second-based history is retained for far
   less time than the intrabar budget implies, so if the ladder picked 1s, coverage
   is days-to-weeks regardless of the cap.

Also watch the **Anchor** row: an anchor no coarser than the chart timeframe (e.g.
`Day` on a daily chart) resets every bar, so the "cumulative" ratio collapses to a
single bar's ratio. The table flags that in orange.

## Choosing the reset period

The anchor has to be coarser than the chart timeframe, and coarse enough that a
period holds enough bars for the ratio to mean something. Bars per period:

| Chart TF | Day | Week (24/7) | Week (session) | Month (24/7) |
|---|---|---|---|---|
| 5m  | 288 | 2016 | 390 | 8640 |
| 1h  | 24  | 168  | 32  | 720  |
| 4h  | 6   | 42   | 30  | 180  |
| 1D  | resets every bar | 7 | 5 | 30 |

**On a 4h chart, `Day` is the wrong choice**: 6 samples per period means the first
bar of each day *is* the whole ratio, and it takes three or four bars before the
denominator carries any weight. You spend a sixth of the series looking at reset
artifacts. `Session (regular)` is worse — on a 4h chart the regular-hours open
generally falls inside a bar, so the reset lands on whichever bar happens to
contain it.

Use `Rolling window (bars)` at 42 (one week, 24/7) or 30 (session-based). It has
no boundary discontinuity at all and its denominator is constant, so sensitivity
stays flat instead of decaying. `Week` is the reasonable calendar-anchored
alternative and is the default. `Month` (~180 bars) works for position context but
is slow to turn.

Avoid `None (running)` on any timeframe unless you specifically want it: the
denominator grows without bound, so the ratio loses sensitivity bar by bar and
converges to the whole-sample mean.

One interaction to watch on 4h: at the default cap the ladder picks 1m (240
intrabars/bar), which covers only ~410 bars ≈ 10 weeks — barely two `Month`
periods. Dropping the cap to 200 moves it to the 5m rung (48/bar, ~2,000 bars ≈ a
year) if you need a longer anchor to be usable.

When rolling, a window that spans a rung change is shaded purple: for those
`rollLen` bars the sum mixes two delta scales. (`Reset cumulation when the
intrabar source changes` only affects the calendar-anchored path, since a rolling
sum has no reset to move.) A window containing any bar without intrabar data is
likewise treated as untrusted in full, not just the missing bars — a partial window would understate
volume and inflate the ratio.

If you switch `Blank out bars with no intrabar data` off to see history past the
coverage boundary, turn `Reset cumulation when the intrabar source changes` back
on. With blanking off, that setting is doing more work, not less: without it a
single period can sum fully-signed intrabar deltas and bar-level proportional
estimates into one number, and the two are not on the same scale.

## Notes

- The realtime bar's intrabars update live, so the last value repaints until the
  bar closes.
- `Max intrabars per chart bar` is the one knob to tune: lower it if the script
  errors out or loses history, raise it for finer delta on higher timeframes.
