[README.md](https://github.com/user-attachments/files/34303503/README.md)
# First Presented FVG (FPFVG)

A TradingView indicator that marks the first Fair Value Gap presented inside each of up to three session windows — and nothing else.

Pine Script v6. Overlay. Licensed under [MPL 2.0](LICENSE).

---

## Why this exists

Every FVG indicator I have used paints every gap on the chart. That is not how the concept gets traded. The gap that matters is the one price leaves behind first after a session opens, because it becomes the reference the rest of the session is measured against. Once you have twenty gaps on a 1-minute chart you no longer have a reference, you have wallpaper.

This does one thing: finds the first gap per session window per day, draws it, and leaves it alone.

---

## The model

### Detection

Standard three-candle Fair Value Gap:

- **Gap up** — candle 1's low is above candle 3's high
- **Gap down** — candle 1's high is below candle 3's low

There is deliberately **no middle-candle close confirmation**. Some implementations require the middle candle to close beyond the far candle's extreme. That filter silently rejects valid gaps where the middle candle is a wide-range bar that closed back inside its own range. On a general FVG tool you lose one zone out of many. On a *first-presented* tool, rejecting the real first gap promotes the second one to "first" — a wrong answer rather than a missing one.

### Session membership follows the middle candle

A gap belongs to whichever window its **middle candle** falls inside — the candle that holds the imbalance and that the gap is named for.

This is the rule that makes the session times mean what they look like. A `0931` window makes the earliest possible formation **09:30 / 09:31 / 09:32**: the gap is at 09:31, so the box anchors at 09:30 and the trio confirms at 09:32.

Gating on the confirming candle instead would let a `0931` window admit the trio 09:29/09:30/09:31 — a gap sitting at 09:30, a minute before the window opens.

### Session strings are literal

No hidden offset. `0301` begins detection at 03:01. What is typed is what runs.

The shipped defaults sit one minute after each session open (`0301`, `0931`, `1001`) so the opening candle cannot be the gap. Set them to `0300` / `0930` / `1000` if you want it to be.

### First is first

Once a session's FPFVG is registered it is permanent. A later gap in the same window never replaces it, and price trading through the zone changes nothing.

There is no mitigation rule, no inversion, and no invalidation. The zone is a static reference that price is free to cut across.

### Volume imbalance encompassing

Where a body gap sits beside the middle candle, the zone boundary expands from the wick level out to the neighbouring candle's body. Each side is tested independently; where both exist, both are taken into a single zone.

| | Gap up | Gap down |
|---|---|---|
| Base bounds | `high[2]` → `low` | `high` → `low[2]` |
| Lower expansion | down to candle 3's body top | down to candle 1's body top |
| Upper expansion | up to candle 1's body bottom | up to candle 3's body bottom |

Toggleable, on by default.

### No direction

Gap up and gap down are both simply *the first gap*. There is no polarity in the label and no bull/bear colour pair. Colour and label text identify the **session slot**, nothing else.

### Size filter

`Min FVG Size` is applied **before** the first-decision, so the FPFVG is the first gap that *clears the filter* — not the first gap, subsequently discarded for being too small. Without it a 2-point gap takes the slot and the real one five minutes later never gets drawn.

It measures the **drawn zone height after volume-imbalance encompassing**, not the raw wick gap. Those are not the same number: a 1-point wick gap flanked by body gaps can draw a 5-point zone, and filtering on the wick would discard something perfectly tradeable.

Three bases:

| Mode | Measures against | Use when |
|---|---|---|
| `ATR multiple` (default, 0.4) | 1m ATR(14) | Default. Self-scaling across instruments and volatility regimes |
| `Points` | Price units | Tradeability should not shrink in a quiet session |
| `% of price` | Current price | Comparing across very different price levels |

`0` disables the filter.

On 1m index futures the separation is wide: zone-to-ATR ratios for obvious noise sit at 0.3 and below, and the gaps worth marking run from roughly 0.45 upward. `0.4` sits just under that floor.

ATR warm-up at the very start of a dataset yields `na`, which passes rather than blocks.

---

## Timeframe handling

Detection always runs on the **1-minute series** via `request.security`, whatever chart you are on.

| Chart | Behaviour |
|---|---|
| 1m – 5m | Draws the 1m zones |
| 6m and above | Draws nothing, shows a notice |
| Seconds, daily, weekly, non-minute | Blocked |

The gate is intentional. The first presented gap is a 1-minute reference; on a 5-minute chart the "first gap of the session" is a different gap, so the level would be silently wrong rather than absent.

**Known cosmetic limitation:** zone left edges use `xloc.bar_time` and therefore snap to the chart's bar grid. A gap anchored at 09:33 draws from the 09:30 bar on a 5m chart. Price levels remain exact; only the horizontal anchor rounds.

---

## Architecture

Worth reading before patching, because two decisions here are not obvious.

### Scalars cross the timeframe boundary, not arrays

`request.security` returns a **fresh copy of any returned array on every chart bar** and retains it in the series history. An earlier build returned the full record store that way and hit `RE10139: Memory limits exceeded` — first on 5m, where the underlying 1m dataset is five times the chart's bar count.

The 1m context now returns **twelve scalars**: each session's current-instance record as `(top, bottom, start, end)`. The historical set is accumulated chart-side in a single `var` store, so memory stays flat regardless of timeframe.

If you extend this, do not return collections from `request.security`.

### Record ingestion

A slot whose start time changes has produced a new FPFVG and is appended to the store. An unchanged start time only refreshes that record's right edge.

Pine rolls `var` state back on each realtime tick, so ingesting on unconfirmed bars is safe and lets the current session's zone appear live.

### The extend window is the sole authority on the right edge

Each slot owns its own window. Nothing carries a zone past it, and nothing cuts it short:

- Another session firing inside the window does **not** truncate it. Overlapping sessions draw over each other and are told apart by colour and label.
- While a window is open, the stored right edge tracks the live bar, so the zone grows rightward on its own and freezes when the window closes.

Want a zone to run longer, lengthen its window. That is the only control.

Earlier builds had three separate mechanisms competing to set the right edge — a `Historical right edge` mode, an `Extend latest FPFVG to current bar` toggle, and cross-session truncation. Each quietly overrode the extend window from a different direction. All three are gone.

### Overlapping windows

Session 3 (`1001-1100`) sits inside Session 2 (`0931-1200`). If a single gap is the first for both, it registers **once** — the lowest-numbered claiming slot owns it — and every claimant is latched so none goes hunting for a later gap as its own "first". One box, not two stacked at doubled opacity.

Where two *different* zones overlap in price, the translucent fills stack and the band reads darker than either alone. Lower the session swatch opacity if that gets in the way.

### Drawing

Zones are static references with absolute time coordinates, so the whole drawing set is torn down and rebuilt on the last bar rather than maintained per bar. The draw set is sorted by formation time, because slots are ingested in slot order rather than clock order.

### Pine gotchas guarded against

- `for k = 0 to m - 1` counts **downwards** when `m` is 0, indexing position −1. Every such loop is guarded by a non-empty check.
- Per-slot records reset to `na` when a new session instance opens, so a stale record can never be carried forward by the next day's extend window.
- A wrapped expression must continue on a line indented by a number of spaces that is **not** a multiple of 4, or Pine reads it as a new statement. Multi-line ternaries in this file sit at 5 or 17 spaces. Where that got fragile the expression was converted to an `if`/`else` chain instead.
- `ta.*` must be evaluated unconditionally on every bar of its context, so the ATR is taken at the top of the detection function rather than inside the filter branch.

---

## Settings reference

### Session 1 / 2 / 3

| Input | Default (S1 / S2 / S3) | Notes |
|---|---|---|
| Enable | off / on / on | |
| Colour | blue / cyan / orange | Identifies the slot; carries no direction |
| Session Time | 0301-0400 / 0931-1200 / 1001-1100 | Detection window, New York, literal — the gap sits at the start minute |
| Extend FPFVG | mirrors session time | Hard ceiling on the right edge; nothing extends past it |
| Label text | `3-4 FPFVG` / `Open FPFVG` / `10-11 FPFVG` | Drawn on that slot's zone |
| History | off | Show prior days for this slot |
| Days | 2 | Counts days *this slot* produced a gap, so blank days are skipped |

### FPFVG Settings

| Input | Default | Notes |
|---|---|---|
| Min FVG Size | ATR multiple / 0.4 | Measured on zone height post-VI. Applied before the first-decision |
| Encompass Volume Imbalances | on | |
| Historical transparency | 88 | Swatch sets hue, this sets strength |

### C.E. (Consequent Encroachment)

Hardcoded, no inputs: the zone's midpoint line is `#424351`, dotted, width 1. It is a fixed part of the zone's presentation rather than something to restyle per chart.

On historical zones it inherits `Historical transparency` along with the box.

### Label Settings

Toggle, text colour, background, size, alignment, and an optional date stamp. The text itself is per session, in each session's own group.

Date formats: `Full` (Friday, August 21, 2026), `Long` (Fri, 21 Aug 2026), `Medium`, `ISO`, `Day/Month`, `Month/Day`, `Short`, `Weekday`. Rendered in New York time so the stamp matches the session day rather than the exchange day.

---

## What it does not do

No alerts. No entries, exits, targets or stops. No direction bias. No mitigation or inversion tracking. No multi-timeframe confluence. No dashboards.

It marks one level per session and gets out of the way.

---

## Install

Copy `First_Presented_FVG.pine` into the TradingView Pine Editor and add to chart. Open a 1m to 5m chart of an intraday instrument.

---

## License

Mozilla Public License 2.0. See [LICENSE](LICENSE).

File-level copyleft: use it commercially, wrap it in whatever you like, but modifications to these files stay open under MPL. This is the convention for published Pine and is what TradingView's own scripts carry.

---

## Notes

Built on ICT concepts. Timezone is hardcoded to `America/New_York` so daylight saving is handled without a selector.

Nothing here is financial advice, and past behaviour of any level predicts nothing.
