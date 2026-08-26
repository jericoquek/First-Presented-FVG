[README.md](https://github.com/user-attachments/files/31471306/README.md)
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

### Session membership

Decided by the **confirming (third) candle**. A gap belongs to whichever window its third candle closes inside.

### The opening-minute rule

The first bar of each session instance can never confirm an FPFVG. A `0930` window begins detection at `09:31`; `0300` begins at `03:01`; `1000` begins at `10:01`.

Enter the **true session open**. The offset is applied in code, not typed into the session string. Entering `0931` yields `09:32`, because the code adds its own minute on top.

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

Gap up and gap down are both simply *the first gap*. There is no polarity in the label and no bull/bear colour pair. Colour identifies the **session slot**, nothing else.

### Size filter

`Min FVG Size %` is applied **before** the first-decision, so the FPFVG is the first gap that *clears the filter* — not the first gap, subsequently discarded for being too small.

---

## Timeframe handling

Detection always runs on the **1-minute series** via `request.security`, whatever chart you are on.

| Chart | Behaviour |
|---|---|
| 1m – 5m | Draws the 1m zones |
| 6m and above | Draws nothing, shows a notice |
| Seconds, daily, weekly, non-minute | Blocked |

The gate is intentional. The first presented gap is a 1-minute reference; on a 5-minute chart the "first gap of the session" is a different gap, so the level would be silently wrong rather than absent.

**Known cosmetic limitation:** zone left edges use `xloc.bar_time` and therefore snap to the chart's bar grid. A gap formed at 09:33 anchors to the 09:30 bar on a 5m chart. Price levels remain exact; only the horizontal anchor rounds.

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

### Overlapping windows

Session 3 (`10:00–11:00`) sits inside Session 2 (`09:30–12:00`). If a single gap is the first for both, it registers **once** — the lowest-numbered claiming slot owns it — and every claimant is latched so none goes hunting for a later gap as its own "first". One box, not two stacked at doubled opacity.

### Right-edge resolution

Two modes:

- **Latest FPFVG edge** (default) — every prior zone runs forward to the newest zone's right edge. Old levels keep projecting into the present.
- **Next FPFVG** — each zone stops where the following one forms, giving a continuous non-overlapping chain.

Truncation is tested only against **drawn** zones, so disabling a slot never leaves a phantom cutting a neighbour short.

### Drawing

Zones are static references with absolute time coordinates, so the whole drawing set is torn down and rebuilt on the last bar rather than maintained per bar. The draw set is sorted by formation time before truncation is computed, because slots are ingested in slot order rather than clock order.

### Pine gotchas guarded against

- `for k = 0 to m - 1` counts **downwards** when `m` is 0, indexing position −1. Every such loop is guarded by a non-empty check.
- Per-slot records reset to `na` when a new session instance opens, so a stale record can never be carried forward by the next day's extend window.

---

## Settings reference

### Session 1 / 2 / 3

| Input | Default (S1 / S2 / S3) | Notes |
|---|---|---|
| Enable | off / on / on | |
| Colour | blue / cyan / orange | Identifies the slot; carries no direction |
| Session Time | 0300-0400 / 0930-1200 / 1000-1100 | Detection window, New York, opening minute excluded |
| Extend FPFVG | mirrors session time | Governs the right edge; start only matters if it precedes the session |
| History | off | Show prior days for this slot |
| Days | 2 | Counts days *this slot* produced a gap, so blank days are skipped |

### FPFVG Settings

| Input | Default | Notes |
|---|---|---|
| Min FVG Size % | 0 | Percentage of price. Applied before the first-decision |
| Encompass Volume Imbalances | on | |
| Historical transparency | 88 | Swatch sets hue, this sets strength |
| Historical right edge | Latest FPFVG edge | Or *Next FPFVG* |
| Extend latest FPFVG to current bar | on | |

### C.E. Settings

Consequent encroachment line at the zone midpoint — toggle, colour, style (default dotted), width.

### Label Settings

Toggle, text colour, background, size, alignment, label text (`FPFVG`), and an optional date stamp.

Date formats: `Full` (Friday, August 21, 2026), `Long` (Fri, 21 Aug 2026), `Medium`, `ISO`, `Day/Month`, `Month/Day`, `Short`, `Weekday`. Rendered in New York time so the stamp matches the session day rather than the exchange day.

**Run Left alignment with Latest FPFVG edge.** Every zone shares a right edge in that mode, so right-aligned labels overprint on the same spot.

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
