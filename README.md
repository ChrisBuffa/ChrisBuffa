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
2. Use a **5-minute** chart of a futures contract such as NQ1!, MNQ1!, ES1!, GC1!, SI1! or CL1!. The profiles use 1-minute intrabar data for precision, and you need at least 20 sessions loaded for the monthly profile.
3. Paste the day's option levels into **EOD option levels**, one line per ticker:
   ```
   NQ: GEX10=27750, Call Resistance=27600, GEX6=27500, 1D Max=27372.57, GEX5=27250, Put Support 0DTE=27010, Call Resistance 0DTE=27000, Gamma Wall 0DTE=27000, GEX3=26950, HVL=26850, GEX2=26750, HVL 0DTE=26740, GEX4=26600, GEX1=26500, 1D Min=26317.43, GEX9=26200, GEX7=26100, Put Support=26000, GEX8=25900
   ES: Call Resistance=7550, GEX8=7450, ...
   GC: HVL=3935, ...
   SI: HVL=50, ...
   CL: Call Resistance=90, ...
   ```
   You can paste the whole block into every chart. Each chart picks out its own line: MNQ uses NQ, MES uses ES, MGC uses GC, SIL uses SI and MCL uses CL. A chart with no matching line gets no levels.

## What's drawn for checking
- **VWAP:** the session VWAP and its ±1σ, ±2σ and ±3σ bands are on by default. A setting can also show them on past sessions.
- **Profile levels:** POC is a solid line, VAH and VAL are dashed, and HVNs are dotted. Each of the 3D, 5D, W (7-session) and M (20-session) profiles has its own colour.
- **Option levels:** every pasted level (GEX 1–10, HVL, Call Resistance, Put Support, 1D Max/Min and the 0DTE levels) is drawn as a dashed line.
- **Labels:** each price gets one label listing every level at that price with its S-score points, e.g. `26500.00  GEX1 [100] · 5D POC [50]`.
- **Histograms:** the four profiles are drawn side by side to the right of the last bar. Buy volume is cyan and sell volume is red, with the POC, VA and HVN lines marked on each one.
- **Level check table** (bottom right) shows every profile's POC, VAH, VAL and HVN 1–3, plus VWAP and all six bands. Below them it shows the row size, the tolerance, whether 1-minute intrabar data was used, the number of sessions loaded and the number of option levels parsed.

### Profile precision
- **Row size:** rows are about one tick tall: NQ 1.0, ES 0.25, GC 0.1, CL 0.01.
- **Intrabar data:** volume comes from **1-minute intrabars** (`request.security_lower_tf`), with each minute spread over its own high–low range. This is more accurate than spreading a whole 5-minute bar evenly.
- **HVNs:** an HVN is a peak on the smoothed profile, at least about 1/30 of the profile's height away from the POC. It is then moved to the single busiest row near that peak, so the price is exact.
- **Recommended chart:** a **5-minute chart**. It loads 20+ sessions, which the monthly profile needs, while the 1-minute intrabars provide the precision.

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
- **Size** = floor(risk budget ÷ (stop distance × point value)). It uses Micro or Mini point values for NQ, ES, YM, RTY, GC, SI and CL. Other symbols use the chart's own point value.
- **BE milestone** follows the model's BE rules. It is the nearest VWAP or VWAP band, or opposing option level, at least 0.5R away.
- **Runners** are the next option, profile, VWAP and prior-day or prior-week high/low levels beyond BE, spaced at least 0.5R apart.

## Alerts
Create an alert on the indicator and choose **Any alert() function call**. You'll get one message per confirmed setup with its number, direction, S-score, zone and components. **Confirmed long T3** and **Confirmed short T3** are also available as plain alert conditions.

## Assumptions to review (under *Model settings*)
- **Which sessions get setups:** pasted option levels are treated as valid for the current session only, so setups are only built there. *All loaded history* applies today's levels to every session. That is useful for study, but it is not a true backtest.
- **Weekly profile:** W uses the last 7 sessions, which is the "7-day" tier. **Monthly** uses the last 20 sessions. Both can be changed.
- **Tolerance:** the default is 4% of the daily ATR (about 14 points on NQ). You can set a fixed number of points instead.
- **VWAP warm-up:** no setups form during the first 30 minutes of a session, because the bands are still collapsed then.
- **Unlisted option names:** names outside the grading chart (blind spots, for example) are drawn on the chart but never scored.
- **What the script cannot see:** order-book and absorption confirmation (Bookmap, L2) aren't available in Pine. Treat the alert as "price rejected the zone" and confirm with order flow yourself.
