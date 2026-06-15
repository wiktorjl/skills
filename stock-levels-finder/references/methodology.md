# Methodology — Level Detection

Detailed formulas, parameters, and edge cases for each signal used in `stock-levels-finder`. Read this when the SKILL.md summary is insufficient — typically when implementing the computation step or debugging a result that looks off.

## Table of contents

1. Swing high/low detection
2. Volume-by-price profile
3. Moving averages as levels
4. Anchored / structural levels
5. GEX & DEX interpretation
6. Zero-gamma flip computation
7. Open interest walls
8. Max pain interpretation
9. Dark pool clustering
10. Confluence scoring
11. Ticker-specific calibration

---

## 1. Swing high/low detection

A bar at index `i` is a swing high if `high[i] > high[i±k]` for all `k` in `1..N`. Use **N=5** as default (the bar is higher than the 5 bars on each side). For very volatile names (ATR > 4% of price), use **N=3** to avoid missing swings; for quiet large-caps, **N=7**.

After detection:
- Sort swings by price
- Cluster swings within **1%** of each other (use the median of the cluster as the level price)
- Record `touches = len(cluster)` — more touches = stronger level
- A level needs **≥2 touches** to be reported as a price-action level; a single isolated swing is just a swing, not a level

Watch out for: stock splits in the 2-year window. If using daily bars, ensure they're split-adjusted; otherwise old swings will be at the wrong absolute price. Most APIs return adjusted bars by default but it's worth confirming.

## 2. Volume-by-price profile

The cleanest way to compute this from daily bars (without intraday tick data):

1. Determine price range: `[min_low, max_high]` across the 2-year window
2. Create **50 equal-width bins** spanning that range
3. For each daily bar, distribute its volume across the bins it touched. Simplest acceptable approximation: assume volume is uniform across `[low, high]` of that day, so a bar that spans 3 bins gets `1/3 × volume` in each. More sophisticated: weight by the typical price (`(H+L+C)/3`) and assign all volume there — faster but loses some resolution.
4. Sort bins by total volume

Output:
- **HVN (high-volume nodes)**: top 3–5 bins. These are acceptance zones — price tends to chop here.
- **LVN (low-volume nodes)**: local minima in the volume distribution between HVNs. Price moves *quickly* through these. Useful to flag because a break of an HVN often leads to a fast move to the next HVN.
- **POC (point of control)**: the single highest-volume bin. Treat as a strong level.

If the stock has had a >50% directional move in the window, the profile will look bimodal — that's expected, not a bug. The HVNs from each "regime" are still meaningful.

## 3. Moving averages

Compute on **closes**, daily:

- **50-DMA**: short-term institutional benchmark, frequently reacts on first touch
- **100-DMA**: less watched but worth including
- **200-DMA**: heavily watched, often the line in the sand for trend definition
- **20-EMA**: short-term momentum, useful only if user asks for intraday/swing-tight levels

For each MA, report:
- Current value
- Whether spot is above or below
- Slope (rising / flat / falling) — a falling 200-DMA acting as resistance is qualitatively different from a rising one acting as support

A moving average is a level only if it's within ~5% of spot. A 200-DMA $80 away from price is technically a level but not actionable.

## 4. Anchored / structural levels

Always include:
- 52-week high, 52-week low
- Prior year's high, prior year's low (last calendar year)
- YTD high, YTD low
- All-time high if it occurred within the 2-year window
- Prior major swing high/low that hasn't been retested

Psychological / round numbers: include round figures (`$X00`, `$X50`) within ±10% of spot. These are weak on their own but reinforce other signals when they line up.

## 5. GEX & DEX interpretation

**GEX (gamma exposure)** by strike — what dealers' gamma position looks like at each strike.

- **Positive GEX strike**: dealers are long gamma here → they sell into rallies and buy dips → **stabilizing** → acts as resistance above spot, support below
- **Negative GEX strike**: dealers are short gamma → they buy rallies and sell dips → **destabilizing** → tends to accelerate moves rather than create a level

Rank strikes by **absolute** GEX magnitude. The top 3–5 positive-GEX strikes near spot are the most important "walls". Top negative-GEX strikes are flagged as volatility zones, not as levels.

**DEX (delta exposure)** by strike — directional pressure. Less commonly used than GEX for level identification, but:

- Heavy positive DEX above spot → dealers short delta → they buy stock if price rises → can act as a magnet pulling price up
- Heavy negative DEX below spot → dealers long delta → they sell stock if price falls → can accelerate downside

Treat DEX as supplementary. Report the largest DEX strikes alongside GEX but weight GEX more heavily in confluence scoring.

**Time decay note**: GEX/DEX from front-week options is highly transient. If most exposure is in expirations >1 month out, the levels are more durable. Mention concentration in the report when relevant.

## 6. Zero-gamma flip computation

If `get_greek_exposure_by_strike` returns cumulative GEX, the flip is just the strike where the cumulative curve crosses zero. If it returns per-strike GEX, you have to compute the cumulative yourself:

```
sort strikes ascending
running = 0
for strike, gex in strikes:
    prev_running = running
    running += gex
    if prev_running and running and sign(prev_running) != sign(running):
        flip = interpolate(strike-1, strike, prev_running, running)
        break
```

If multiple flips exist (rare but possible with skewed positioning), report all of them and the regime in each band.

When spot is above flip: market is in a stabilizing regime, expect mean-reversion, moves get sold/bought into the walls. When below: reflexive regime, moves accelerate, levels are weaker. This single number changes how the user should interpret all the other levels — always report it prominently.

## 7. Open interest walls

For each expiry within ~60 days, identify:
- Top 5 call-OI strikes
- Top 5 put-OI strikes

Aggregate across expiries by strike (sum OI). The all-expiry totals give you the most robust walls; the front-week totals tell you where this Friday's pin risk is.

**Call walls act as resistance** (concentration of short-gamma exposure for dealers, plus call sellers defending the strike). **Put walls act as support** (mirror logic).

A "wall" should be at least 2x the next-highest strike's OI to count. If OI is evenly spread, there are no walls.

If the user asks specifically about a near-term event (earnings, Fed, OPEX), filter OI to the relevant expiry.

## 8. Max pain interpretation

Max pain is the strike at which option holders (collectively) lose the most money at expiration. Theory: market makers have incentive to push price toward this strike into expiry, since they're typically net short the options.

In practice, max pain is a useful **monthly OPEX magnet** — the third Friday of each month, especially for high-OI names like SPY/QQQ/AAPL/NVDA. Less reliable for weeklies.

Report the next monthly max pain explicitly. If spot is within 2% of max pain and OPEX is within 5 trading days, flag this as a high-priority near-term level.

## 9. Dark pool clustering

`get_dark_pool_volume_price_group` already buckets DP volume by price level. Use the buckets directly — don't re-bucket.

Identify:
- **Top 3 price buckets by DP volume** in the recent window (default: last 30 days). These are institutional acceptance zones.
- **Any single bucket holding >15% of total DP volume** — that's an unusual concentration and a strong level.

Separately, scan `get_dark_pool_trades` (last ~200 trades) for individual prints with notional >$10M. Each such print is a discrete level at its execution price. List the top 3–5 individually if they're not already captured by the buckets.

Dark pool levels tend to be **inflection points, not bounces** — institutions accumulating at a level means they're absorbing supply, which doesn't necessarily mean price holds, but it does mean the level is "real" and being defended.

## 10. Confluence scoring

For each candidate level price `p`, count which of these independent sources confirm it (within the cluster tolerance, see SKILL.md):

1. Price-action (swing touches ≥2)
2. Volume profile HVN
3. Moving average (50/100/200-DMA, only if within 5% of spot)
4. Anchored level (52w high/low, prior year H/L, etc.)
5. GEX wall (top 3 absolute by strike)
6. Open interest wall (top 3 call or put across expiries)
7. Max pain (current monthly)
8. Dark pool bucket (top 3) or large single print

A level confirmed by ≥3 sources is **major**. ≥2 is **secondary**. 1 is **noted but minor**.

Within source, tiebreak by:
1. Number of sources (always primary)
2. Sum of within-source ranks (lower = better, e.g., #1 GEX + #1 OI beats #3 GEX + #3 OI)
3. Recency of last test (more recent = stronger)
4. Distance to spot (closer = more actionable)

Don't report more than 3 resistances and 3 supports in the headline table. Anything beyond R3/S3 dilutes the signal. Mention additional ones in "Notable observations" if they're worth flagging.

## 11. Ticker-specific calibration

- **SPY / QQQ / IWM / index ETFs**: GEX/DEX/max pain are *primary*, price-action is secondary. Dealer positioning dominates these names.
- **Mega-caps (AAPL, NVDA, TSLA, MSFT, GOOGL, META, AMZN)**: all signals matter roughly equally. Options are deeply liquid; dark pool is heavy.
- **Mid-caps and below**: options data thins out fast. Trust price-action and dark pool more; GEX/OI may be noisy or absent for non-front-month strikes.
- **Highly shorted / meme names (e.g., heavy borrow rate)**: options positioning can be distorted by hedging of short positions. Treat GEX with extra skepticism. Use `get_short_data_by_ticker` if you want to sanity-check.
- **ETFs tracking futures (e.g., USO, UNG)**: the underlying is a futures contract, not the cash market. Cross-check levels against the underlying future where possible. Avoid this skill for inverse/leveraged ETFs — the math doesn't apply cleanly.

If volume on the name is <500k shares/day average, warn the user that the analysis will be thin and the levels less reliable.

---

## Data quality checks before reporting

Before producing the final report, sanity-check:

- Are bars adjusted for splits? (Look for unexplained discontinuities)
- Did the options data return at least one strike with non-zero OI? If not, options portion is unusable.
- Is DP volume non-zero? Some smaller names will have minimal DP activity.
- Does the GEX flip price look sane (within ±20% of spot)? If it's wildly out of range, the cumulative-sum may have failed; recompute or skip.

If any check fails, mention it in the report's **Caveats** section rather than silently dropping data.
