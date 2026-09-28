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
- `Min intrabars before plotting` (default 30) blanks the plot until the period
  has enough samples for the ratio to mean anything.
- `Blank out bars with no intrabar data` blanks and shades any bar not served by
  the chosen rung, so the switch point is visible as a gap instead of a spike.
  `Reset cumulation when the intrabar source changes` keeps one cumulative sum
  from mixing two rungs' delta magnitudes.

Orange shading = no intrabar data for those bars. Grey shading = warming up after
an anchor reset. Alerts fire off the gated series, so they stay silent in both.

## Notes

- The realtime bar's intrabars update live, so the last value repaints until the
  bar closes.
- `Max intrabars per chart bar` is the one knob to tune: lower it if the script
  errors out or loses history, raise it for finer delta on higher timeframes.
