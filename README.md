# Flow Model V3: TradingView indicator

`flow_model_v3.pine` is a Pine Script v6 overlay that finds **Tier-3 (T3) reversal setups**. A T3 setup is a spot where three rejection sources sit within one tolerance band:

| Confluence | Sources | Points |
|---|---|---|
| Volume profile | POC / VAH / VAL / HVN of the 3-day · 5-day · 7-day · monthly composite | 25 · 50 · 90 · 100 |
| VWAP band | Current-session VWAP ±1σ · ±2σ · ±3σ | 50 · 90 · 100 |
| Option level | HVL, GEX 7–10 → 25 · GEX 4–6, HVL 0DTE → 50 · GEX 2–3 → 90 · GEX 1, Call Resistance, Put Support, 1D Max/Min, Call Resistance 0DTE, Put Support 0DTE, Gamma Wall 0DTE → 100 | |

**S-score** = round((profile + band + option) / 3). A used setup loses 10 points. A cluster (overlapping setups in the same direction) shows the highest score among its setups. The score measures how strong the confluence is. It is not a win probability.

## Install
1. In TradingView open **Pine Editor**, paste the whole file, and click **Add to chart**.
2. Use a 1–5 minute chart of a futures contract such as NQ1!, MNQ1!, ES1! or MES1!. The profiles are built from the chart's own bars, so you need at least 20 sessions of history loaded.
3. Paste the day's option levels into **EOD option levels**, for example:
   ```
   $NQ1!: Call Resistance, 21600, Put Support, 21000, HVL, 21300, 1D Min, 21080, 1D Max, 21550, Call Resistance 0DTE, 21450, Put Support 0DTE, 21150, HVL 0DTE, 21280, Gamma Wall 0DTE, 21400, GEX 1, 21500, GEX 2, 21350, GEX 3, 21200
   ```
   You can paste several tickers on separate lines. The indicator uses the line that matches the chart (MNQ uses the NQ line). `Name 21500` and `Name: 21500` also work.

## How a setup works
- **Direction:** a lower-band confluence below price is a **long** setup. An upper-band confluence above price is a **short** setup.
- **Box parts:**
  - **Outer box:** the full span of the three levels.
  - **Inner box:** the two levels that sit closest together.
  - **Center box:** the average of the three levels.
  - **Edge line (yellow):** the invalidation side of the box. The stop goes here plus a buffer.
- **Lifecycle** (evaluated at bar close only, so nothing repaints):
  - **Waiting:** the setup has formed and price has not reached it.
  - **Touched:** price enters the box. The touched part is shaded.
  - **Confirmed / used:** a candle closes back out on the entry side (a rejection). This fires the alert, applies −10, and switches the setup to the T3-U color.
  - **Invalidated:** a candle closes beyond the edge.
- **Session end:** setups expire. Numbers restart at #1 each session.

## Risk overlay
Set **Setup number** to the `#` shown on a label. The indicator then draws entry, stop, BE milestone and runners, and fills the risk box:
- **Size** = floor(risk budget ÷ (stop distance × point value)). It uses Micro or Mini point values for NQ, ES, YM, RTY, GC and CL. Other symbols use the chart's own point value.
- **BE milestone** follows the model's BE rules. It is the nearest VWAP or VWAP band, or opposing option level, at least 0.5R away.
- **Runners** are the next option, profile, VWAP and prior-day or prior-week high/low levels beyond BE, spaced at least 0.5R apart.

## Alerts
Create an alert on the indicator and choose **Any alert() function call**. You'll get one message per confirmed setup with its number, direction, S-score, zone and components. **Confirmed long T3** and **Confirmed short T3** are also available as plain alert conditions.

## Assumptions to review (under *Model settings*)
- **Which sessions get setups:** pasted option levels are treated as valid for the current session only, so setups are only built there. *All loaded history* applies today's levels to every session. That is useful for study, but it is not a true backtest.
- **7-day profile:** this uses the last 7 sessions. **Monthly** uses the last 20 sessions. Both can be changed.
- **Tolerance:** the default is 4% of the daily ATR (about 14 points on NQ). You can set a fixed number of points instead.
- **VWAP warm-up:** no setups form during the first 30 minutes of a session, because the bands are still collapsed then.
- **Unlisted option names:** names outside the grading chart (blind spots, for example) are drawn on the chart but never scored.
- **What the script cannot see:** order-book and absorption confirmation (Bookmap, L2) aren't available in Pine. Treat the alert as "price rejected the zone" and confirm with order flow yourself.
