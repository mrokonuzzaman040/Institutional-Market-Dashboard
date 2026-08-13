# Institutional Market Dashboard

TradingView Pine v6. Built for FX majors, crosses, metals and indices.

| Version | File | Status |
|---|---|---|
| v7 | [src/IMD_v7.pine](src/IMD_v7.pine) | current — latency rework |
| v6.1 | [src/IMD_v6.pine](src/IMD_v6.pine) | stable fallback |
| v5 | [src/IMD_v5.pine](src/IMD_v5.pine) | minimal fallback |

---

## 0a. Settings map

Thirty collapsible groups in the Inputs tab, ordered by how often you touch them —
trading decisions at the top, engine tuning at the bottom. Dependent fields
**grey out automatically** when their parent switch is off (Pine's `active`
parameter), so you only ever see what's live.

| | Group | What lives there |
|---|---|---|
| **TRADE** | `01 ▸ Trade Call` | enable, fire-on trigger, entry type, thresholds, cooldown |
| | `02 ▸ Trade Call · Chart Card` | lines, shading, label size, extension |
| | `03 ▸ Entry Timing` | speed profile, signal timing, intrabar detection, manual pivots |
| | `04 ▸ Armed Setup` | zone types, max distance, min quality |
| | `05 ▸ Risk & Targets` | SL buffer, TP1/TP2 R multiples, min RR |
| | `06 ▸ Position Sizing` | balance, risk %, conversion rate, contract size |
| **FILTERS** | `07 ▸ Sessions` | filter mode, timezone, hard block |
| | `08 ▸ Session Windows` | killzones, full sessions, rollover, Friday |
| | `09 ▸ News Blackout` | enable, padding, manual event list |
| | `10 ▸ News Auto-Rules` | NFP, 08:30 NY, FOMC hour |
| | `11 ▸ NO TRADE Gate` | threshold, ADX, alignment, risk, range |
| | `12 ▸ Volatility Regime` | ATR percentile, adaptive scaling |
| **ENGINE** | `13 ▸ Multi-Timeframe` | repaint mode, which TFs |
| | `14 ▸ Structure Anchors` | primary + second anchor |
| | `15 ▸ Liquidity` | EQH/EQL tolerance, level cap, sweep window |
| | `16 ▸ Order Blocks` | displacement mode, body size, lookback |
| | `17 ▸ FVG · P/D · VWAP` | gap size, caps, VWAP anchor |
| | `18 ▸ Key Levels` | PDH/PDL, PWH/PWL, opens, Asian range |
| | `19 ▸ Currency Flow` | strength TF, lookback, noise floor, prefix |
| | `20 ▸ Order Flow` | source, proxy, intrabar TF, Premium flag |
| | `21 ▸ Order Flow · Thresholds` | delta %, divergence, absorption, vacuum |
| | `22 ▸ Decision Scoring` | A+/A thresholds, checklist %, deadzone |
| | `23 ▸ Indicator Lengths` | EMA / RSI / ADX / ATR periods |
| **DISPLAY** | `24 ▸ Panel` | position, text size, view mode |
| | `25 ▸ Panel Layout` | split mode, panel count, row budget, positions |
| | `26 ▸ Panel Sections` | per-section on/off **and** panel assignment on one line |
| | `27 ▸ Chart Drawings` | SMC zones, structure labels, level lines, NO TRADE shading |
| | `28 ▸ Theme` | dark/light, custom colours |
| **OUTPUT** | `29 ▸ Alerts` | JSON payload toggle |
| | `30 ▸ Performance Tracker` | target, BE modelling, cooldown, hold |

### Things that now grey themselves out

- Manual pivot fields unless `Speed Profile` is `Manual`
- Panel B/C position and row budget unless the layout is split; Panel C unless `Panels` is 3
- Section A/B/C assignment unless layout is `Manual`
- Killzone windows unless the session filter is on `Killzones` (and likewise Full / Custom)
- Rollover and Friday times unless their block is ticked
- Every Order Flow threshold unless the module is enabled
- Custom colours unless `Use Custom Colors` is on
- Adaptive multipliers, news auto-rules, tracker options, armed-setup fields, second anchor — all follow their parent

Section visibility and panel assignment now share one line in `26 ▸ Panel
Sections`, so a section's on/off switch and where it goes sit next to each other
instead of in separate groups.

---

## 0. Signal latency — read this first

v6 published an order block **6–12 bars after the move started**. That is not a
bug; it is five delays stacked on top of each other, and on a scalp timeframe it
is the whole trade.

| Source | v6 delay | v7 |
|---|---|---|
| Chart pivot `(5,5)` — right bars *are* the wait | 5 bars | `(4,1)` on Scalp → **1 bar** |
| Anchor pivot `(15,15)` on the anchor TF | 15 anchor bars | `(10,2)` → **2 anchor bars** |
| BOS decided inside the HTF context | up to 1 full HTF bar (4 h on a 240 anchor) | levels stay HTF, **break read chart-side** |
| OB waits for a pivot-confirmed break | + 1–3 bars | **displacement candle publishes it immediately** |
| Everything gated on `barstate.isconfirmed` | + 1 bar | optional Early tier |

Worst case on M5 with a 240 anchor: v6 needed 5 + 3 + 48 + 1 bars of chart time
before an OB appeared. v7 with the Scalp profile publishes the same block on the
close of the impulse candle.

### The honest trade-off

**Earlier means wronger.** Every bar of confirmation you remove buys speed with
accuracy — a 1-bar-right pivot marks swings that later turn out not to be swings.
Nothing here changes that; v7 just puts the dial in your hands and, importantly,
**measures the result** so the choice stops being a guess.

Two things also worth saying plainly:

- **Late entries are one possible cause of losses, not the only one.** Over-trading,
  size, spread on a 1-minute scalp, and a strategy that simply has no edge on your
  pair and session all look identical from inside a losing week. The new MFE / MAE
  row is there to tell these apart — see below.
- **No indicator makes trading precise.** This one scores confluence and manages
  timing. It does not know the future, and A+ is not a probability.

### Reading MFE / MAE

The tracker now records, for every closed trade, how far price ran **in your
favour** (MFE) and **against you** (MAE) before it resolved, both in R.

| Pattern | Meaning |
|---|---|
| MFE ≫ MAE | entries are early enough — the edge is real |
| MFE ≈ MAE | coin-flip entries; confluence is not adding value |
| **MAE > MFE** | **you are entering late** — price consistently moves against you first |

The panel prints an explicit `⚠ Entry timing` warning once 10+ trades show
MAE ≥ MFE. That is the number to optimise against, not the hit rate.

v5 fixed v4's blocking logic and added HTF-anchored structure, sessions and a
trade plan. v6 adds measurement, currency flow, key levels, news blackout,
volatility adaptation, sizing and machine-readable alerts.

---

## 1. Architecture

```
MTF TREND ─┐
CCY FLOW  ─┼─► BIAS ─► ANCHOR STRUCTURE ─► CHART ZONES ─► DECISION ─► PLAN
KEY LEVELS─┘   (x2 anchors)   (OB/FVG/sweeps/divergence)   ▲          ▼
                                                    NO TRADE    TRACKER
```

### Layer 1 — MTF trend index

`f_trendBundle()` runs once per requested timeframe, returns three numbers.

| Factor | Weight |
|---|---|
| EMA fast − EMA mid | 15 |
| EMA mid − EMA slow | 15 |
| close − EMA slow | 10 |
| EMA fast slope | 10 |
| RSI level | 15 |
| MACD histogram | 15 |
| DI+ − DI− | 10 |
| volume-weighted candle direction | 5 |
| ATR expansion × direction | 5 |

Each factor is squashed to 0–1 by `f_norm(x, scale)` before weighting, so `ix` is
a bounded 0–100 trend index. `mo` is momentum, `ax` is raw ADX. Pulled for
1m/5m/15m/30m/1H/4H/1D.

**No-repaint:** `lookahead_on` plus a `[1]` offset applied *inside* the HTF
context — the correct idiom, returns the last **closed** HTF bar.
`liveMode = true` switches to `lookahead_off` with no offset (micro-repaint).

Aggregates: `avgIdx`, `avgMom`, `avgAdx`, `alignPct`, `confidence`, `mktState`.

### Layer 2 — Currency flow (new in v6)

Seven USD majors are fetched (EURUSD, GBPUSD, USDJPY, USDCHF, USDCAD, AUDUSD,
NZDUSD) and converted to a percentage return over `strLen` bars on `strTF`. Each
currency's raw strength is its return against USD, and **USD is the zero
reference** — which makes any pair's implied return simply `rBase − rQuote`:

| Symbol | Resolution | Equals |
|---|---|---|
| EURUSD | rEUR − rUSD | `ret(EURUSD)` |
| USDJPY | rUSD − rJPY | `ret(USDJPY)` |
| EURJPY | rEUR − rJPY | ≈ `ret(EURJPY)` — no USD leg needed |
| XAUUSD | base unknown → `−USD` | weak dollar = gold tailwind |
| US30 | disabled under `Auto` | — |

That differential is scaled by the **spread** between the strongest and weakest of
the eight currencies, so ±100 means "this pair moved as much as the entire basket
spread". Displayed leg strengths are mean-demeaned versions of the same numbers.

`flowFloor` is the noise gate: when the whole basket's spread is under it (default
0.05 %), nothing meaningful is happening and `flowScore` is forced to 0 rather
than amplified. Without it, a flat basket divided by a near-zero spread pinned
every currency to ±100 and handed the filter maximum conviction on noise.

`flowAgree` needs `flowScore` past `flowMin` in the bias direction. A direct
conflict adds 12 to both the risk score and the NO TRADE score.

Set `strPfx` to `OANDA:` or `FX:` if the bare symbols don't resolve on your feed.
When `flowMode` is `Off` — or `Auto` on a non-FX symbol — the seven requests are
skipped entirely.

### Layer 3 — Anchor structure (bias)

`f_structBundle()` runs entirely inside the anchor timeframe and keeps its `var`
state there. Returns swing extremes, trend, BOS/CHoCH, EQH/EQL, raw pivots.

- `close > extHi` while already bullish → **BOS**; while bearish or undefined →
  **CHoCH** and the trend flips. Mirrored for lows.
- Equal highs/lows within `eqTol × ATR` → EQH/EQL, drawn thicker.
- **Two anchors** (v6). Primary drives structure and P/D; the second (default D)
  is a trend-only confirmation. `dualAgree` requires both non-zero and equal.

Chart-side, flags are edge-detected so each event fires once.

### Layer 4 — Chart entry zones

Faster structure on the chart TF via `intLen` pivots. The anchor decides
direction; this times the entry.

- **Order Blocks** — created on a chart BOS/CHoCH only when the break candle body
  ≥ `obDispAdj × ATR`. The block is the last opposite candle within `obLook`.
- **Fair Value Gaps** — 3-bar imbalance ≥ `fvgMinAdj × ATR`, removed at the 50 %
  mid fill or on close-through.
- **Liquidity** — anchor pivots, EQH/EQL, **plus PDH/PDL, PWH/PWL and Asian range
  H/L** (v6). Swept = wick through and close back; taken = close through.
- **Sweep → CHoCH** — sweep of sell-side liquidity then a bullish chart CHoCH
  within `seqBars` → `seqDir = 1`. Mirror for bearish. The core edge trigger.
- **Divergence** (v6) — pivot-confirmed RSI divergence, replacing v5's crude
  one-bar heuristic. Bearish = price HH with RSI LH.
- **Anchored VWAP** (v6) — cumulative from each new day or CHoCH, with ±1σ bands.
  Computed manually rather than with `ta.vwap()`, which raises a runtime error on
  volume-less feeds.
- **Premium/Discount** — where close sits inside the anchor range.

### Layer 5 — Key levels (new in v6)

| Level | Source |
|---|---|
| PDH / PDL | daily `high[1]` / `low[1]` |
| PWH / PWL | weekly `high[1]` / `low[1]` |
| Daily / Weekly open | HTF `open` (determined at HTF bar open — no lookahead) |
| Midnight open | first bar of 00:00 America/New_York |
| Asian range H/L | tracked live during `asiaSess`, frozen at session end |

Anything within `lvlProx × ATR` of price counts toward `levelConf`, worth up to 8
quality points, and all of them are pushed into the liquidity array so they can be
swept and used as TP targets.

### Layer 6 — Decision

**Quality score (0–100)** — confluence, *not* win probability:

| Component | Points |
|---|---|
| Trend index strongly with bias | 12 |
| Sweep→CHoCH sequence | 12 |
| Price inside aligned OB | 11 |
| Primary anchor agrees | 10 |
| Aligned FVG | 9 |
| Currency flow agrees | 8 |
| Key-level confluence | 8 |
| Momentum with bias | 8 |
| Correct Premium/Discount side | 7 |
| Second anchor agrees | 6 |
| Relative volume | 5 |
| TF alignment | 4 |

**Risk score** — exhaustion, low quality, poor alignment, wrong P/D side, flow
conflict, thin volume.

**Checklist** — 14 items, but the denominator shrinks when an input is
unavailable (no volume feed, no flow data, dual anchor off, no VWAP). Reported as
a percentage so thresholds stay comparable.

Session is deliberately *not* a checklist item. Out of session nothing can fire
anyway, so it was a constant +1 on every reading that mattered — it inflated
`checkPct` by ~7 % and made `minChkPct` looser than the number suggested. It stays
a 25-point NO TRADE penalty and a hard block.

---

## 2. NO TRADE — weighted

Penalties are summed and compared against `ntThrAdj` (the adaptive threshold).

| Condition | Penalty |
|---|---|
| News blackout | 30 |
| Outside session / rollover / late Friday | 25 |
| ATR percentile below `atrRankMin` | 20 |
| Range / Indecisive | 18 |
| ADX below `ntAdx` | 15 |
| TF alignment below `ntAlign` | 14 |
| Risk ≥ `ntRisk` | 12 |
| No anchor structure | 12 |
| Currency flow conflict | 12 |
| Bias in deadzone | 10 |
| Prolonged range | 10 |
| Anchors disagree | 10 |
| Weak impulse + high exhaustion | 10 |
| Contradictory TF split | 9 |
| Small indecisive candles | 7 |

**Hard blocks:** news (when `newsHard`), out-of-session (when `sessHard`), no
anchor structure (when `hardStruct`), no MTF data.

Panel shows `NO TRADE 62/45` plus the ranked reasons, and shades the chart
background when `shadeNT` is on.

### Volatility gate

`ta.percentrank(atr, 200)` — the ATR's percentile against its own history.
Instrument-agnostic, unlike v4's `ATR/price %` which was permanently true on FX
majors and blocked the indicator forever.

---

## 3. News blackout (new in v6)

Pine cannot read an economic calendar, so this is three mechanisms:

1. **Manual list** — `newsRaw`, comma-separated `YYYY-MM-DDTHH:MM` in **UTC**.
   Parsed once on the first bar into a timestamp array.
   Example: `2026-08-14T12:30,2026-08-20T18:00`
2. **NFP auto-block** — first Friday of the month, 08:30 America/New_York.
   Timezone-aware, so it follows US DST automatically.
3. **Optional broad blocks** — all 08:30 NY releases (CPI, PPI, claims, retail
   sales all land there) and the 14:00 NY FOMC hour.

`newsPad` sets the window either side, in minutes.

---

## 4. Volatility adaptation (new in v6)

`atrRank` scales a single multiplier from `adaptLo` (0th percentile) to `adaptHi`
(100th). It drives:

- `fvgMinAdj` — FVG minimum size
- `obDispAdj` — OB displacement requirement
- `slBufAdj` — stop buffer
- `ntThrAdj` — NO TRADE threshold (stricter in dead volatility)

Turn `adaptOn` off to use the raw inputs everywhere.

---

## 5. Sessions

| Mode | Windows (default, GMT) |
|---|---|
| Off | always open |
| Killzones | London 07:00–10:00, NY 12:00–15:00 |
| Full Sessions | London 07:00–16:00, NY 12:00–21:00 |
| Custom | one user window |

Asia (00:00–07:00) is opt-in. Rollover (20:50–22:10) and late Friday
(19:00–23:59 Fri) are blocked separately — both are spread-widening windows.

---

## 6. Trade plan and sizing

- **Stop** — beyond the chart-TF swing, floored at 0.5 ATR from entry, plus
  `slBufAdj × ATR`. Falls back to 1.5 ATR if no swing exists.
- **TP1 / TP2** — `rr1` and `rr2` multiples of stop distance.
- **Liq Target** — nearest unswept opposing level with its R. Below `minRR`,
  A/A+ signals are suppressed — no room to the next pool.
- **Size** — `lots = (balance × risk% / convRate) / stopDistance / contractSize`.

`contractSize` auto-detects: 100 000 forex, 100 gold, 5 000 silver, 1 otherwise
(ticker matching is case-insensitive and covers `GOLD` / `SILVER` naming as well
as `XAU` / `XAG`). Override it if your broker differs.

### `convRate` — the one field that will bite you

It is **account currency per one unit of the pair's quote currency**, which is
often the reciprocal of the rate you have in your head.

| Account | Pair | Quote | `convRate` |
|---|---|---|---|
| USD | EURUSD, GBPUSD, XAUUSD | USD | **1.0** |
| USD | USDJPY, EURJPY | JPY | **≈0.0067** — *not* 150 |
| USD | EURGBP | GBP | ≈1.27 (GBPUSD) |
| EUR | EURUSD | USD | ≈0.92 (1 / EURUSD) |

Entering 150 instead of 0.0067 sizes you ~22 000× wrong. The panel guards against
this: the **Risk / lot** row shows what one lot actually risks in both quote and
account currency, and a `⚠ Check` row appears when that figure is absurd relative
to your balance. On a USD account with a 30-pip stop on USDJPY, Risk / lot should
read roughly `201.00 (30000.00 JPY)` — if it says 4 500 000, the rate is inverted.

---

## 7. Performance tracker (new in v6)

Every A+/A signal is recorded as a virtual trade and followed forward:

- Target = TP1, TP2 or the liquidity level, per `statTgt`.
- Closed on TP hit, SL hit, or `statHold` bars elapsed (marked to market).
- **Evaluation starts on the bar after entry.** Testing on the entry bar itself
  compares the trade against that bar's own high/low, which include the wick that
  happened *before* the fill — it manufactured losses out of price action the
  trade was never exposed to.
- **`statBE`** (on by default) models the management advice the panel gives: once
  1R is reached the stop moves to entry, and a stop-out there scores **0R**. BE
  scratches count in the sample as non-wins, so they pull hit rate down while
  leaving total R untouched.
- If TP and SL both touch inside the same bar, the stop is assumed — bar data
  cannot tell you which came first.
- **`statCool`** blocks re-entry in the same direction for N bars after a close
  (default 20). Without it the same setup re-opens the bar after it closes, and a
  handful of episodes inflate into a large, highly correlated "sample".
- Only one open trade per direction at a time.

Panel shows sample size, hit rate, average R and total R, split by A+ / A, plus
the best-performing session once it has 5+ trades there. Sessions are bucketed
London / New York / **Lon-NY overlap** / Asia / off-session — the 12:00–16:00
overlap is its own bucket rather than being lumped into New York.

**This is not a backtest.** It ignores spread, commission, swap and slippage;
entries are at bar close; intrabar sequence is unknown. Treat it as a directional
sanity check on your thresholds, and ignore any reading under ~20 trades — the
panel flags this itself.

---

## 8. Alerts

Every signal is gated on `barstate.isconfirmed`, so nothing fires intrabar.

**JSON payload** via `alert()` (create a "Any alert() function call" alert):

```json
{"ind":"IMD v6","event":"A_PLUS","sym":"EURUSD","tf":"15","dir":"BUY",
 "entry":1.08432,"sl":1.08210,"tp1":1.08765,"tp2":1.09098,"lots":4.5,
 "quality":86,"checklist":80,"risk":22,"flow":41,"atrPct":63,
 "ntScore":18,"session":"Open","time":1786512300000}
```

Events: `A_PLUS`, `A`, `SWEEP_CHOCH`, `NO_TRADE_ON`, `NO_TRADE_OFF`.

The named `alertcondition` entries are also kept for simple pop-up alerts.

---

## 9. What changed

### v6.1 — audit fixes

1. **Tracker fabricated losses on the entry bar.** Open and evaluate ran in the
   same pass, so a trade was tested against its own entry bar's high/low —
   including the wick that preceded the fill. Evaluation now starts the bar after
   entry, and the close-out pass runs before the open pass.
2. **No re-entry cooldown** meant one setup was resampled every bar after it
   closed, inflating a handful of episodes into a large correlated sample. Added
   `statCool` (default 20 bars).
3. **`cnt == 0` hard-blocked Weekly and Monthly charts forever** — every listed TF
   is below a weekly chart, so `skipLower` excluded all of them. Now falls back to
   the highest available TF and flags it in the panel.
4. **`scaleF` exploded in quiet markets** — `100 / maxAbs` with a near-zero
   `maxAbs` pinned every currency to ±100. Added the `flowFloor` noise gate.
5. **`chkSess` was a free checklist point.** Removed from the checklist.
6. **Flow maths was inconsistent between majors and crosses** — `sUSD` as the
   negated mean double-counted the USD leg on USD pairs. USD is now the zero
   reference, so `rBase − rQuote` is correct for both.
7. **Seven currency requests fired even with flow disabled.** Now gated, matching
   `f_htf`. (The previous README claim that `Off` skipped them was wrong.)
8. **`convRate` had no error surface.** Added the Risk / lot row, an absurdity
   warning, and the JPY worked example above.
9. **Daily levels evicted structural liquidity** — FIFO flushed anchor pivots and
   EQH/EQL as PDH/PDL/Asia levels piled up. Eviction is now by relevance (swept
   first, then farthest from price), plus a duplicate-level guard.
10. **`line.new` with a negative bar index** on charts under 200 bars.
11. **"Move to BE" was displayed but never modelled.** `statBE` now moves the stop
    to entry at 1R and scores that exit 0R.
12. **Anchored VWAP reset on every CHoCH**, leaving it a few bars old and close to
    random. Default anchor is now Day; `vwapMode` restores the old behaviour.
13. **News list failed silently on whitespace.** Whitespace is stripped, and the
    panel reports parsed-vs-supplied when any entry is rejected.
14. **Low volatility was penalised three times** (nt5, a lower `ntThrAdj`, and a
    smaller `volMult`). The threshold no longer scales below `atrRankMin`.

Minor: London/NY overlap is its own session bucket · gold/silver detection is
case-insensitive and covers `GOLD` / `SILVER` · the management row shows both
directions when two trades are open.

### v6.0

Added: performance tracker · currency flow filter · key levels (PDH/PDL, PWH/PWL,
Asian range, D/W/midnight opens) · news blackout · volatility-adaptive thresholds
· position sizing · JSON alerts · trade management state · second structure anchor
· pivot-confirmed divergence · anchored VWAP · NO TRADE chart shading.

Replaced: `ta.vwap()` with a manual cumulative VWAP, because `ta.vwap()` throws a
runtime error on symbols with no volume data.

### v5 (bug-fix release over v4)

1. **ATR gate blocked all FX** — `ATR/price < 0.10 %` is permanently true on
   majors (EURUSD H1 ≈ 0.074 %). Replaced with ATR percentile rank.
2. **Integer division truncated** — `bullCnt / cnt` and `rangeCount / ntRange`
   were int/int, so `confidence` jumped 0 → 60 with nothing between and
   `rangePct` was only ever 0 or 1.
3. **NO TRADE was ten OR'd booleans** — practically always on, making the A+
   signal dead code.
4. **`showDraw = false` broke liquidity** — `array.push` sat inside the drawing
   guard, so turning drawings off left zero tracked levels.
5. **Bias flipped at exactly 50** while every other band used 55/45.
6. **OB direction inferred from colour equality**, which breaks under themes.
7. **Alerts fired intrabar.**
8. **Volume assumed present** — `na` cascaded through `relVol`, continuation and
   institutional strength on volume-less FX feeds.
9. **Dead computation** — chart-TF `ta.macd` / `ta.dmi` were never read; internal
   structure was computed and discarded.
10. **Arrays reallocated every bar** via `array.from`.
11. `swingTxt` shared one variable across highs and lows.
12. `qualityClass` made the "A Setup" tier unreachable.
13. Liquidity levels were deleted on close-through instead of marked taken.
14. FVG boxes drawn one bar too narrow.
15. `f_institutional` counted `na` early bars as bearish.
16. `mktState` checked ADX before the range test.
17. Liquidity cap raised and made configurable.

---

## 10. Suggested settings

| Style | Chart TF | Anchor 1 / 2 | Killzones | `ntThresh` | Flow |
|---|---|---|---|---|---|
| FX scalp | 1m–5m | 60 / 240 | on | 35 | on |
| FX intraday | 15m | 240 / D | on | 45 | on |
| FX swing | 1H–4H | D / W | off | 55 | on |
| FX crosses | 15m–1H | 240 / D | on | 45 | on (needs both legs) |
| XAUUSD intraday | 5m–15m | 60 / 240 | on | 40 | on (USD leg only) |
| Indices | 5m–15m | 60 / 240 | NY only | 45 | off |

Lower `ntThresh` = stricter. Raise it if the panel never opens.

**Performance:** 18 `request.security` calls (7 MTF + 2 anchors + 2 level TFs +
7 currency pairs), against Pine's limit of 40. Turn `showDraw` off for speed —
all panel logic stays intact. Set `flowMode` to `Off` to skip the seven currency
calls on instruments where it doesn't apply.

---

## 11. Limitations

- Quality score measures **confluence**, not probability. The tracker gives you a
  rough forward reading; it is not an expectancy guarantee.
- OB/FVG/sweeps are chart-timeframe; only bias structure is HTF-anchored.
- No spread or swap feed. Rollover and Friday blocks are time approximations.
- The news blackout only knows what you type in plus the NFP/08:30/14:00 rules.
  It has no live calendar.
- Volume on FX is broker tick volume, not traded volume — VWAP and all
  volume-derived components are weak evidence there.
- Currency flow needs the seven majors to resolve on your data feed. If they
  don't, set `strPfx`.
- `liveMode = true` repaints the current HTF candle by design.

---

## 11b. Armed setups — the anticipatory workflow

The biggest reason a confluence dashboard feels late is that it only speaks once
*everything* is true, and by then price has left. `⑥b Armed Setup` inverts that.

When bias is set, NO TRADE is clear and quality is at least `armMinQ`, v7 scans
for the nearest **aligned zone price has not reached yet** — the highest bullish
OB/FVG below price for a long, the lowest bearish zone above for a short — and
publishes it:

```
🎯 ARMED · LONG 1.08450 (inv 1.08390) · 32 ticks away
```

You get the `V7 · Setup Armed` alert **before** the tap, so you can rest a limit
order at the zone instead of chasing a market fill after the candle closes.
`V7 · Zone Tapped` fires when price actually reaches it.

| Input | Purpose |
|---|---|
| `Arm Setups Before Price Arrives` | master switch |
| `Max Distance to Zone (× ATR)` | ignore zones too far to be actionable (default 3) |
| `Zone Types` | OB + FVG / OB only / FVG only |
| `Minimum Quality to Arm` | don't arm on weak context (default 55) |

This is the single most useful change for scalping: it moves you from reacting to
a printed signal to waiting at a level you chose in advance.

---

## 11c. Speed profiles

`⑥ Speed & Latency` sets every latency-related parameter at once.

| Param | Scalp | Intraday | Swing |
|---|---|---|---|
| Anchor pivot (left, right) | 10, 2 | 15, 4 | 20, 8 |
| Chart pivot (left, right) | 4, 1 | 5, 2 | 8, 3 |
| OB lookback | 8 | 12 | 20 |
| Sweep→CHoCH window | 6 | 12 | 20 |
| Tracker max hold | 30 | 60 | 120 |

`Manual` uses the four pivot fields directly plus the values in their own groups.

**Right bars are the confirmation delay.** A `(10, 2)` pivot needs only 2 bars
after the extreme to confirm, versus 15 for v6's symmetric `(15, 15)`. That single
change removes most of the "3–4 candles late" complaint.

Other switches in that group:

- **`Signal Timing`** — `Confirmed + Early` adds an unconfirmed tier that
  evaluates on the forming bar, shown as `~ EARLY A+ BUY (unconfirmed)` in a faded
  colour and fired via the `EARLY` alert. It can flip before the bar closes and is
  **never fed to the performance tracker**, so your statistics stay honest.
- **`Detect Sweeps Intrabar`** — flags a liquidity sweep as the wick happens
  rather than at bar close.
- **`Detect Structure Breaks Intrabar`** — reads a break of an anchor level on the
  forming bar. The *levels* always come from settled HTF bars and never repaint;
  only the moment the break is recognised changes.

### Displacement order blocks

`Create OB on Displacement Candle` (on by default) is the direct fix for the
original complaint. Instead of waiting for a pivot-confirmed break, v7 marks a
displacement the moment a candle closes with a body ≥ `obDisp × ATR` **beyond the
prior `obSwing`-bar extreme**, and publishes the last opposite candle as the block
on that same close — zero extra bars.

A duplicate guard skips any block overlapping one already tracked, since
displacement often fires on consecutive candles.

Sweep→CHoCH sequencing now also accepts **sweep → displacement**, whichever
confirms first.

---

## 11d. Order flow — what is and isn't possible

### The hard limit

**Pine Script has no bid/ask aggressor feed.** TradingView does not store ticks
with buyer/seller side, so the following are *impossible* in any Pine indicator,
including this one:

- Footprint / cluster charts, delta by price level
- Stacked imbalances, absorption at a specific price
- DOM, order book, resting liquidity

Those need NinjaTrader, ATAS, Sierra Chart or Bookmap. Any TradingView script
claiming true footprint order flow is approximating.

### What v7 actually does

`request.security_lower_tf()` splits each chart bar into lower-timeframe
intrabars. Each intrabar counts as **buying** if it closes above its open,
**selling** if below, split evenly if unchanged. Summing gives bar delta. This is
the same method TradingView's own Volume Delta indicator uses.

Derived from that:

| Metric | Meaning |
|---|---|
| **Bar Delta %** | net buy/sell pressure as a share of bar volume, −100…+100 |
| **Session CVD** | cumulative delta, reset each day |
| **Delta divergence** | price makes a new N-bar extreme, CVD does not |
| **Absorption** | relVol ≥ 1.5 with range ≤ 0.6 ATR — heavy effort, no result |
| **Vacuum / thin** | relVol ≤ 0.7 with range ≥ 1.2 ATR — result with no effort |

### The forex fix that actually matters

**On FX pairs, chart "volume" is tick count, not contracts.** It counts price
changes, not money. Delta built on it is a weak proxy.

So when `Volume Source` is `Auto`, v7 pulls delta from the **CME/COMEX futures
contract instead**, which carries real exchange volume:

| Pair | Proxy | Note |
|---|---|---|
| EURUSD | `CME:6E1!` | direct |
| GBPUSD | `CME:6B1!` | direct |
| AUDUSD / NZDUSD | `CME:6A1!` / `CME:6N1!` | direct |
| USDJPY | `CME:6J1!` | **delta sign inverted** — 6J is JPY/USD |
| USDCHF / USDCAD | `CME:6S1!` / `CME:6C1!` | inverted |
| XAUUSD / XAGUSD | `COMEX:GC1!` / `COMEX:SI1!` | direct |
| Crosses (EURGBP…) | none | falls back to chart tick volume |

The panel tells you which is in use: `CME:6E1! (real vol)` vs
`chart (tick vol)`. Trust the readings far more in the first case.

### How it feeds the decision

Nothing is drawn on the chart — it all goes into scoring, as you asked:

- **Quality**: delta agreeing with bias is worth 8; absorption on the reversal
  side after a sweep adds 4; a vacuum move scores 0.
- **Risk**: delta conflict +14, vacuum +8.
- **NO TRADE**: `Delta Opposes Bias` +14, `Thin / Vacuum Move` +8.
- **Exhaustion**: delta divergence against your bias +16.
- **Checklist**: a `Δ` item appears when data is available.
- **Armed setups**: `Block Zone Tap When Delta Opposes` suppresses the tap
  trigger when price enters your zone while delta pushes the other way — the zone
  is failing, not offering an entry.

### Costs you should know about

1. **200,000 intrabar cap, shared across the whole script.** Finer intrabars mean
   less history. `Auto` picks a ratio of roughly 4–12 intrabars per chart bar
   (15S under M1, 1m up to M5, 5m up to H1, 15m up to H4) which keeps 16k–50k
   chart bars available. Forcing `15S` on an H1 chart would leave a few hundred
   bars and gut the performance tracker.
2. **Seconds-based intrabars are Premium-only** and raise a hard runtime error on
   lower plans — they do not silently degrade. So `Auto` **never** selects a
   seconds timeframe: it picks `1` up to M5, `5` up to H1, `15` up to H4, `60`
   above. `15S` is selectable manually but is ignored unless you also tick
   **"My plan allows seconds intrabars (Premium+)"**.

   When the intrabar timeframe is not strictly below the chart timeframe — a 1m
   chart with a 1m request, say — the request is skipped and delta falls back to a
   bar-level estimate (candle direction × volume). The panel's Source row says
   which is running: `CME:6E1! · real vol · 1 intrabars` versus
   `chart · tick vol · bar-level`. Bar-level is much coarser; on an M1 chart
   without Premium it is all that is available.
3. **Historical and realtime delta differ.** TradingView does not store ticks, so
   the same bar can show different delta before and after a chart refresh. This is
   inherent to the method, not a bug here.
4. It is an **approximation**. A bar closing up does not prove buyers were the
   aggressors.

Set `Enable Order Flow Module` off to remove the two lower-timeframe requests
entirely if you want maximum history for the tracker.

---

## 11e. Trade Call — the one alert that matters

Everything else in the script reports *state*. The Trade Call reports a
**decision**: whether to take a trade, where to enter, where the stop goes, both
targets and the size — in one message.

```
🟢 BUY EURUSD  15
Entry  1.08450  (limit @ zone)
SL     1.08380  (7.0 pips)
TP1    1.08555  (10.5 pips, 1.5R)
TP2    1.08660  (21.0 pips, 3.0R)
Size   0.45 lots  ·  risk 100.00
Why    sweep→reversal · zone tap · OB+ · discount · delta 34%
Score  Q84/100 · checklist 78% · risk 22
Session Open · real-vol delta
```

### Setting it up

The full message needs `alert()`, so in TradingView's alert dialog choose
**Condition: IMD v7 → "Any alert() function call"**. Leave the message box as
`{{alert_message}}` or empty — the script supplies the text. Works with the mobile
app, email and webhooks.

There is also a `V7 · TRADE CALL` entry in the condition list. That one is a plain
popup — TradingView cannot put dynamic prices into an `alertcondition` message, so
it only tells you a call fired. Use the `alert()` version for the levels.

### When it fires

| Input | Default | Effect |
|---|---|---|
| `Fire On` | Zone Tap or Impulse | the entry *event* required, not just a good score |
| `Entry Type` | Auto | limit at the armed zone when one exists, otherwise market |
| `Minimum Quality` | 70 | |
| `Minimum Checklist %` | 65 | |
| `Cooldown (bars)` | 15 | stops one setup alerting repeatedly |
| `Require Order Flow Not Opposing` | on | suppresses the call when delta pushes against the direction |
| `Draw Entry / SL / TP on Chart` | on | four lines plus a label at the levels |

`Fire On` is the important one:

- **Zone Tap** — only when price actually reaches an armed OB/FVG. Fewest, best calls.
- **Impulse** — on a displacement candle or a sweep-reversal. Catches momentum entries.
- **Zone Tap or Impulse** *(default)* — either.
- **Any Confluence** — score alone, no entry event. Noisiest; mostly for testing.

NO TRADE, session, news and the RR floor all still gate it — a call cannot fire
while the dashboard is blocked.

### Entry, stop and size are computed for the quoted entry

When Auto picks a **zone limit**, the stop is placed beyond that zone's
invalidation (plus the ATR buffer) and both TPs are measured from the *zone*
price, not from the current close. Lot size is recomputed against that stop
distance, so the number in the alert is the size for the trade being quoted.
Distances read in pips on FX and metals, ticks elsewhere.

### The chart card is the primary display

The call is drawn **on the chart**, not buried in the side panel. On fire you get:

- **Entry** line (solid, header colour), **SL** (dashed red), **TP1** (dotted
  green), **TP2** (dashed green)
- **Shaded zones** — risk tinted red from entry to stop, reward tinted green from
  entry to TP2, so the R:R is visible at a glance
- A **live card** anchored at the right edge of the levels:

```
🟢 BUY  EURUSD  15
● RUNNING
─────────────
Entry  1.08450
SL     1.08380   (7.0 pips)
TP1    1.08555   (1.5R)
TP2    1.08660   (3.0R)
Size   0.45 lots
─────────────
sweep→reversal · zone tap · OB+ · discount · delta 34%
```

**It tracks the trade live.** The lines and boxes extend forward with price and
the status line updates itself: `● RUNNING` → `✅ TP1 HIT · trail` → `✅ TP2 HIT`
or `⛔ SL HIT`, with the card recolouring to match. Once TP2 or SL resolves it, the
drawing freezes on the chart as a record and the next call replaces it.

The entry bar is never judged against its own high/low — that wick preceded the
fill, so hit detection starts the bar after entry.

Three things mark the call so direction is never ambiguous:

1. **A `BUY` / `SELL` marker on the entry bar itself** — plotted below the bar for
   longs, above for shorts. Always visible, never affected by chart margin.
2. **A price label on every level** — `SL 1.08380`, `TP1 1.08555`, `TP2 1.08660`,
   sitting on the line, ticking to `✓ TP1 …` as each is reached.
3. **The full card**, anchored a few bars right of the live bar and following it
   forward.

| Drawing input | Default |
|---|---|
| `Draw Entry / SL / TP on Chart` | on |
| `Shade Risk / Reward Zones` | on |
| `Card Offset from Price (bars)` | **3** |
| `Chart Label Size` | Small |
| `Price Label on Each Level` | on |
| `BUY / SELL Arrow at Entry Bar` | on |
| `Also Show Row in Side Panel` | off |

**Keep `Card Offset` small.** It sets how far right of the live bar the card sits.
TradingView's right margin is only a handful of bars, so a large offset parks the
card off-screen — the lines still draw, but the text is invisible.

The side-panel row is opt-in — turn it on only if you want a text summary
alongside the chart card.

### Honest caveat

The call is a *rules-based* trade idea from the confluence model, not a
prediction. It inherits every limitation above: no spread modelling, quality is
confluence rather than probability, and the outcome statistics come from the same
approximate forward tracker. Check MFE/MAE after 20+ calls before scaling size on
it.

---

## 12. Split layout

With everything enabled the dashboard runs ~75 rows, which overruns any screen.
`①b Split Layout` enforces a height budget instead of leaving it to chance.

| Input | Purpose |
|---|---|
| `Layout` | `Single` · **`Auto Balance`** (default) · `Manual` |
| `Panels` | 1–3 |
| `Max Rows per Panel` | height budget, default 26 |
| `Panel B / C Position` | any of the nine table anchors |
| `Repeat Status Line on Other Panels` | mirrors the NO TRADE / OPERABLE banner so each panel reads on its own |
| `Show Row Counts in Title` | prints `rows A:24 B:26 budget:26` for tuning |
| Nine `A`/`B`/`C` dropdowns | Manual mode only |

### Auto Balance

Each section's row cost is known ahead of render (Timeframes 8, Summary 7,
Currency Flow 3, Order Flow 6, Smart Money 14, Key Levels 6, Decision 11, Trade
Plan 13, Performance 8). The allocator fills Panel A until the *next* section
would breach `Max Rows`, then spills into B, then C. Sections are never split
mid-way, so a section is always whole on one panel.

Because the cost model reacts to your own toggles, turning off Smart Money or
switching to `Compact` re-balances automatically — you never have to re-assign
anything by hand.

**Tuning:** turn on `Show Row Counts in Title`, then lower `Max Rows` until the
tallest panel fits your screen. If two panels still overflow at a comfortable
budget, set `Panels` to 3. At the default 26-row budget the full dashboard needs
3 panels; at ~38 it fits in 2.

### Placement

- **Two columns:** A `Top Right`, B `Top Left`
- **Three:** A `Top Right`, B `Middle Left`, C `Bottom Left`
- **Stacked right edge:** A `Top Right`, B `Middle Right`, C `Bottom Right`

The status banner, reason list and title always stay on Panel A. The A+/A signal
band follows whichever panel holds Decision. `View Mode: Compact` works alongside
all of this and drops the optional rows entirely.

---

## 13. Install

1. TradingView → Pine Editor → paste [src/IMD_v6.pine](src/IMD_v6.pine).
2. Save, then **Add to chart**.
3. Set first, in this order: `Primary Anchor TF`, `Session Filter`, `Flow Filter`,
   then `Account Balance` / `Risk %` / `Quote → Account Rate`.
4. Let the tracker collect 20+ trades before you trust its numbers, then tune
   `qMinAplus` and `ntThresh` against them.
