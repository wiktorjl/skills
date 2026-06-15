---
name: stock-levels-finder
description: Identify major support and resistance levels for a stock by combining 2 years of daily price/volume history (Massive API) with options positioning (GEX, DEX, gamma, max pain), open interest, and dark pool prints (Unusual Whales API). Use whenever the user asks about support/resistance, key levels, magnet strikes, gamma walls, dealer positioning, dark pool levels, "where will this stock find support / get rejected", price targets, accumulation zones, pivot levels, or anything resembling a technical levels read on a specific ticker — even if they don't use the words "support" or "resistance" explicitly.
---

# Stock Levels Finder

Find the price levels that actually matter for a given ticker by fusing three independent signals:

1. **Price/volume history** (Massive API) — where price has repeatedly reacted over the last ~2 years
2. **Options positioning** (Unusual Whales) — where dealers are forced to hedge (GEX/DEX), where the most contracts sit (OI), and the strike that minimizes option-holder payout (max pain)
3. **Dark pool prints** (Unusual Whales) — where institutions transacted in size off-exchange

The thesis behind combining these: a level that shows up in *all three* (say, a price that's been tested 4x, has a gamma wall on it, and a large dark pool print clustered there) is materially more durable than a level that shows up in only one. The final output should rank levels by this confluence.

## When to invoke

Trigger on phrases like "support and resistance for $TICKER", "key levels on NVDA", "where's gamma flip", "dark pool levels for SPY", "major levels for the week", "what strikes matter", "where are the walls", "find me a buy zone", or any combination of ticker + (level/zone/target/pivot/magnet/wall/pin). If the user names a ticker and asks about anything price-structure-related, default to running this skill.

If the user hasn't given a ticker, ask for one before starting. If they haven't given a timeframe, default to swing-trading horizon (next 1–8 weeks) and tell them so — they can redirect.

## Required data — fetch in parallel

All of these should be fetched concurrently in a single message. They are independent.

### From Massive API (price/volume — `mcp__massive__*` tools)

Use `search_endpoints` first if you don't know the endpoint name (Massive exposes Polygon-style endpoints). What you need:

- **Daily aggregates / bars** for the ticker, covering the last 2 years (~504 trading days). Adjusted for splits.
- **Previous close and current quote** so you know where price is *right now* relative to historical levels.
- If available, **52-week high/low** and **weekly aggregates** for the same window.

Use `store_as` on the bars response so you can run multiple analyses against the same dataset without refetching. Then `query_data` to compute the derived levels (see "Compute" below).

### From Unusual Whales (options + dark pool)

Call these in parallel:

- `get_greek_exposure_by_strike` — GEX and DEX by strike (the single most important signal). Sum across expiries unless the user is asking about a specific event.
- `get_greek_exposure_by_expiry` — to know how dealer positioning rolls off over time (a wall expiring this Friday is different from one anchored 3 months out).
- `get_max_pain` — magnet level for monthly OPEX.
- `get_options_chain` or `get_chains_for_expiry` — for **open interest** distribution by strike. Top-OI strikes act as gravity points and pin candidates.
- `get_dark_pool_volume_price_group` — dark pool prints bucketed by price. This is the single best "institutional level" signal.
- `get_dark_pool_trades` (limit ~200, recent) — to spot recent large single prints worth flagging separately.
- `get_flow_alerts` for the ticker (recent) — unusual options activity reveals where smart money is positioning *now*; the strikes show up as forward-looking levels.
- `get_open_interest_changes` — strikes where OI is growing/shrinking fastest (forming or dissolving levels).

If any call errors, continue with the others and note the gap in the final report — don't abort.

## Compute — how to derive levels from raw data

See `references/methodology.md` for full formulas and edge cases. The short version per signal:

### Price-history levels

- **Swing highs/lows**: find local extrema in the daily bars where a high/low is greater/less than the surrounding N=5 bars on each side. Cluster swings within 1% of each other into a single level (the "zone").
- **Volume-by-price (poor man's volume profile)**: bucket the 2-year history into ~50 price bins, sum the daily volume traded inside each bin's range. The top 3–5 bins by volume are high-volume nodes (acceptance zones); the lowest-volume valleys between them are low-volume nodes (price moves *through* these fast).
- **Moving averages** as dynamic levels: 50-day, 100-day, 200-day SMA. The 200-DMA in particular is watched by institutions and frequently reacts.
- **Anchored levels**: 52-week high, 52-week low, all-time high (if in the 2-year window), prior year's high/low, YTD high/low.
- **Round numbers**: psychological levels at $X00 / $X50 within 10% of current price.

### Options-positioning levels

- **Largest positive GEX strikes** → resistance (dealers short gamma above and sell rallies / long gamma forces rallies to stall).
- **Largest negative GEX strikes** → instability / acceleration zones, not levels per se — flag as "low support / high volatility below here".
- **Zero gamma / gamma flip level**: the strike where cumulative GEX crosses zero. Above = stabilizing regime, below = destabilizing. Critical level to report.
- **Largest DEX strikes** → directional pressure zones.
- **Max pain** → magnet into monthly OPEX, especially the Tuesday-Wednesday before.
- **Top open-interest strikes** (calls and puts separately): call OI walls = resistance, put OI walls = support.
- **Recent flow strikes** (last 5–10 trading days): where new positioning is being built.

### Dark pool levels

- **Top price buckets by DP volume** in the price-group endpoint. These are where institutions accept inventory.
- **Recent large single prints** (>$10M notional or >1% of ADV): treat each as a discrete level worth flagging.

## Synthesize — confluence ranking

This is the most valuable step. Don't dump raw lists at the user.

1. Collect every candidate level from every signal into one list with `{price, source, strength, note}`.
2. Cluster nearby candidates: anything within **0.5% of each other** (for liquid large-caps) or **1%** (for smaller/more volatile names) becomes a single zone, with the cluster's midpoint as the headline price.
3. Score each cluster by:
   - **Number of independent sources** confirming it (price-action, GEX, OI, DEX, max pain, dark pool) — this is the dominant factor
   - **Strength within each source** (e.g., this is the #1 GEX strike, not the #7)
   - **Recency** (a level tested this month > a level last touched 18 months ago)
   - **Distance from current price** (levels very close to spot get priority for actionability)
4. Label each zone as **support** (below current price) or **resistance** (above), and tag it: `S1`, `S2`, `S3`, `R1`, `R2`, `R3` ordered by distance from spot.

## Report structure

Output exactly this structure. The user wants something they can act on, not a wall of text.

```markdown
# $TICKER — Support & Resistance Levels
*Spot: $X.XX • As of: YYYY-MM-DD • Window: 2y daily + current options/DP*

## Headline levels

| Tag | Price | Type | Confluence | Why it matters |
|-----|-------|------|------------|----------------|
| R3  | $...  | Resistance | 4/6 sources | 200-DMA + largest call OI + max pain + DP cluster |
| R2  | $...  | Resistance | 3/6 | ... |
| R1  | $...  | Resistance | ... | ... |
| --- | spot  | $X.XX      | ---        | --- |
| S1  | $...  | Support    | ... | ... |
| S2  | $...  | Support    | ... | ... |
| S3  | $...  | Support    | ... | ... |

## Gamma regime

- **Zero-gamma / flip level**: $...
- **Current regime**: above flip → stabilizing / below flip → reflexive
- **Largest GEX wall**: $... (size, expiry concentration)

## Notable observations

- Dark pool concentration at $... (X% of recent DP volume) — institutional level
- Unusual recent flow into the $... strike (expiry ...) — forward-looking
- Low-volume gap between $... and $... — price likely moves quickly through this range

## Caveats

- [Anything that was unavailable, e.g., "DEX-by-strike was empty for this name"]
- [Time-sensitivity, e.g., "GEX is pre-OPEX and may shift materially after Friday"]
```

Keep prose minimal. The table is the deliverable.

## Important judgment calls

- **Don't over-precision the levels.** Report to the nearest dime for stocks under $50, nearest dollar for $50–500, nearest $5 above that. False precision ($427.83 as a level) is noise.
- **A "level" is a zone, not a price.** When you cite $450 as resistance, it really means roughly $448–$452 in practice.
- **Recency matters but isn't everything.** A 2022 high that price has approached three times in 2025 without breaking is still alive. Don't dismiss old levels mechanically.
- **GEX/DEX values are not levels by themselves.** The *strikes* with the largest exposure are the levels. Don't paste raw exposure numbers at the user without translating to price.
- **Don't conflate strike price with stock price for ETFs that have a different reference** (e.g., index futures vs. cash). For typical US single-name equities and ETFs, strikes == stock prices.
- **Be explicit when a signal is missing.** If `get_greek_exposure_by_strike` returns empty for a small-cap, say so — don't quietly fall back to price-only and pretend the report is complete.
- **Don't include forecasts or trade recommendations** unless asked. The skill produces levels; the user decides what to do with them.

## Followups to offer

After delivering the report, briefly offer (one line each, not a long menu):

- A focused look at a specific level the user cares about
- A version weighted toward a different time horizon (intraday / next earnings / quarterly)
- Comparison to a related ticker (e.g., a sector ETF or competitor)

Don't run any of these unprompted.
