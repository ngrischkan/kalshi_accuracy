# Are Kalshi Sports Markets Well Calibrated?

<!-- WRITE THIS: 1–2 sentences stating your main finding with a number.
Facts to use:
- 6,155 NFL, NBA, MLB and NHL games
- prices 24 hours before close were well calibrated in all four leagues
- no price range missed by more than random chance would explain
- Brier score 0.233 overall vs 0.25 for guessing 50% (7% better)
-->
Kalshi's sports prices are well calibrated. Using 6,155 NFL, NBA, MLB and NHL games, I found that prices 24 hours before close were well calibrated in all four leagues. In every price range, the gap between Kalshi's price and the actual win rate was small enough to be explained by random chance. There was an overall Brier score of 0.233 vs 0.25 for guessing 50%, a 7% improvement.

![Calibration curve](charts/calibration.png)

## The question

When a Kalshi market says a team has a 70% chance to win, does that team actually win about 70% of the time? This project measures that for every settled NFL, NBA, MLB and NHL game-winner market I could pull from Kalshi's public API.

## Data

- **Source:** Kalshi's public trade API (no account needed): `/markets`, `/historical/markets`, and the market candlestick endpoints.
- **Series:** `KXNFLGAME`, `KXNBAGAME`, `KXMLBGAME`, `KXNHLGAME`.
- **Raw pull:** 16,890 settled markets (each game has two mirror-image markets, one per team).
- **Cleaning steps:**
  1. Dropped 202 duplicate tickers that appeared in both the live and historical endpoints.
  2. Kept only markets that resolved `yes` or `no` (dropped 44 `scalar` settlements, e.g. postponed games).
  3. Dropped 13 markets that never traded.
  4. Kept **one market per game**, chosen at random (`random_state=42`), so each game counts once and prices span the full 0–100% range.
  5. Kept only markets listed at least 24 hours before close (see note below).
  6. Dropped 11 markets with no price available near the 24-hour mark.
- **Final dataset:** 6,155 games — MLB 2,534 · NHL 1,690 · NBA 1,479 · NFL 452.

**Note on MLB:** Kalshi listed 2025 regular-season MLB games only 5–18 hours before close, then switched to listing them ~3 days ahead for the 2026 season. Because I score prices 24 hours out, the MLB sample is mostly the 2026 season plus the 2025 playoffs.

![MLB listing change](charts/mlb_listing_change.png)

## Method

- **Price used:** the market price **24 hours before close**, taken from hourly candlesticks (the candle closest to the 24-hour mark; if no trade happened that hour, the bid/ask midpoint). The final pre-close price is useless for this — by then the outcome is known (it scores a meaningless Brier of 0.0008).
- **Brier score:** the mean squared difference between price and outcome:

  $$\text{Brier} = \frac{1}{N}\sum_{i=1}^{N}(p_i - o_i)^2$$

  Lower is better; always guessing 50% scores 0.25.
- **Calibration curve:** prices grouped into 10% bins; each bin's average price is plotted against the fraction of those teams that won. Error bars are ±2 standard errors, $\sqrt{\bar p(1-\bar p)/n}$. Bins with fewer than 20 games are hidden in the per-league charts.
- **API etiquette:** requests are paced and retried with exponential backoff on HTTP 429 (rate limit).

## Results

| League | Games | Brier score | Improvement over 50% guess | Price spread (std) |
|---|---|---|---|---|
| NBA | 1,479 | 0.205 | 18% | 0.22 |
| NFL | 452 | 0.225 | 10% | 0.18 |
| NHL | 1,690 | 0.243 | 3% | 0.10 |
| MLB | 2,534 | 0.244 | 2% | 0.08 |
| **All** | **6,155** | **0.233** | **7%** | |

![Calibration by league](charts/calibration_by_league.png)

<!-- WRITE THIS: explain what the table + league chart show, in your own words.
Points to make:
- all four leagues sit on the diagonal (calibrated)
- the Brier differences track the price spread: NBA/NFL have lopsided matchups, MLB/NHL rarely go past 75%
- so leagues differ in how PREDICTABLE they are, not in how HONEST the prices are
-->

## A trap I found: trading volume

<!-- WRITE THIS: tell the story of the volume analysis.
Facts:
- tried testing whether heavily traded games are better priced, using volume quartiles
- High-volume group looked badly miscalibrated (calibration error 0.080 vs ~0.02–0.04)
- but volume_fp includes IN-GAME trading, and upsets/comebacks attract the most in-game trading
- so "high volume" partly means "surprising outcome" — look-ahead bias
- evidence: in the High group, 15% underdogs won 29%, 85% favorites won only 64%
- league mix can't explain it (High group is mostly NBA, which is well calibrated on its own)
- conclusion: total volume can't be used for this; a fair test needs volume traded BEFORE the 24h mark
-->

## Limitations

<!-- WRITE THIS: 3–5 bullets. Ideas:
- only four sports, game-winner markets only
- NFL sample is small (452 games) → wider error bars
- MLB sample is mostly one season (listing-time change)
- a single horizon (24h); results could differ closer to game time
- extreme price bins have few games
-->

## What I'd do next

<!-- WRITE THIS: 2–3 bullets. Ideas:
- redo the volume test using only pre-24h volume from candlesticks
- compare calibration at 7 days, 24 hours and 1 hour before close
- compare Kalshi prices to sportsbook closing lines
-->

## How to run

1. Python 3.14 with `requests`, `pandas`, `numpy`, `matplotlib`, `jupyter`.
2. Open `kalshi_calibration.ipynb`. With `DOWNLOAD = False` and `FETCH_PRICES = False` (the defaults), it loads `markets.csv` / `clean_markets.csv` and runs the analysis in under a minute.
3. Set either flag to `True` to re-pull data from Kalshi (the price fetch takes ~30–60 minutes).

## Files

- `kalshi_calibration.ipynb` — the full analysis
- `markets.csv` — raw settled markets
- `clean_markets.csv` — cleaned dataset with 24h prices
- `charts/` — figures used above