---
name: options-position-review
description: Review an existing options position (single leg or multi-leg spread) and recommend a concrete action — KEEP, CLOSE, or MODIFY (adjust legs, roll, or close-and-reopen a different structure). Always reviews the underlying STOCK first — volume, volume profile, moving averages (20/50/200), RSI, MACD, and relative strength vs SPY (via stock-data MCP), plus options context like IV, greeks, dealer positioning, and earnings risk (via Unusual Whales) — forms a bullish/bearish/neutral read matched to the position's expiry, then reconciles that read against the position's directional thesis. Use whenever the user shares an options position, spread, or trade and asks what to do with it — "should I keep / close / roll / adjust my [calls/puts/spread/condor/strangle]", "manage my position", "is my trade still good", "my [TICKER] spread", or any options legs + a request for a recommendation.
---

# Options Position Review

Decide what to do with an options position by answering one question in order: **"Where is the stock going over the life of this trade, and does this position still express that view?"**

The skill runs in two halves:

1. **Read the stock** (the objective layer) — build a bullish / bearish / neutral verdict from price/volume/momentum, calibrated to the *position's expiry horizon*, not some generic timeframe.
2. **Judge the position against that read** (the decision layer) — classify the structure, recover its embedded thesis, compare thesis vs. stock read, and emit one of three verdicts with a specific action:
   - **KEEP** — thesis intact, position healthy, no better structure available
   - **CLOSE** — thesis is broken or the risk/reward has decayed past the point of repair
   - **MODIFY** — thesis mostly intact but the expression is wrong: roll in time, adjust strikes, roll one side, or close this structure and reopen a better-fit one

The single most common failure this skill prevents: **holding a directional position after the stock has flipped on you** (a bullish call spread while the stock breaks down), and its mirror, **panic-closing a winner that's simply consolidating**. Tie every recommendation back to the stock read.

## When to invoke

Trigger whenever the user supplies an options position — any of: single calls/puts, vertical/credit/debit spreads, calendars, diagonals, iron condors, iron flies, butterflies, straddles, strangles, ratio spreads, covered calls, CSPs, collars, LEAPS — together with any ask to evaluate or manage it ("should I hold this?", "roll or close?", "is this still good?", "what do I do with my $TICKER spread?"). Also trigger on a bare position dump with no explicit question — the implied request is "tell me what to do."

Do **not** trigger this skill for: finding new trade ideas from scratch (no existing position), pure support/resistance requests (use `stock-levels-finder`), or general market commentary.

## Step 0 — Parse the position completely before fetching anything

Pin down every field. If the user's description is ambiguous on a field that changes the recommendation, ask **one** consolidated clarifying question rather than guessing. Required fields:

- **Underlying ticker** and **current/entry underlying price** (entry optional but useful)
- **Every leg**: long/short, call/put, strike, expiration, quantity
- **Net debit/credit paid** (entry cost basis) — needed for P&L and management targets. If missing, ask or proceed and flag that P&L-based logic is degraded.
- **Current position value / mark** if the user has it (otherwise pull leg marks from the chain)
- **The user's original thesis and timeframe**, if stated. If not, infer the thesis from the structure (see methodology) and ask whether the inferred view is correct.
- **Account/risk context** if volunteered (size relative to account, max loss tolerance, assignment concerns, whether they can take assignment / margin). Never invent this; if it's decision-relevant and absent, note the assumption.

From the legs, immediately classify: **structure type**, **directional bias** (bullish / bearish / neutral), **vega sign** (long or short volatility), **theta sign** (collecting or paying time decay), **defined or undefined risk**, and **days to expiration (DTE)** for each expiry. See `references/methodology.md` §1 for the structure taxonomy and how to read bias/greeks off the legs.

**Set the bias horizon from DTE.** This is the calibration that makes the stock read relevant:
- DTE ≤ 10 → weight the **daily / short-term** trend and intraday volume
- DTE 10–45 → weight the **daily trend with weekly confirmation**
- DTE > 45 (incl. LEAPS) → weight the **weekly / multi-month** trend; daily noise matters less

A position is only as "wrong" as its horizon says it is. A bearish daily wobble does not break a 120-DTE bullish LEAP.

## Step 1 — Read the stock (fetch concurrently)

Fetch all of the following in a single batch — they're independent. Primary source is the **`stock-data`** MCP (it has purpose-built tools for every indicator below); use **Unusual Whales** for the options-context layer; **Massive** as a fallback for raw price/quotes.

### Required technical signals (`mcp__stock-data__*`)

- **`get_latest_price`** (or `get_live_price` during RTH) — where price is *right now*; everything else is relative to this.
- **`get_volume_profile`** (`lookback_days` matched to horizon: ~30 short, ~90–120 swing, ~252 long) — POC, value area high/low, HVN/LVN. Tells you whether price sits in acceptance (chop) or is breaking out of a node.
- **`get_latest_technical_indicators`** — snapshot of MAs, RSI, MACD.
- **`get_technical_indicators`** (`start_date` ~6–12 months back) — the *series*, so you can read **20/50/200 SMA values, their slopes, and price's position relative to each**, plus RSI level/trend and MACD line/signal/histogram. Slope and stacking order (20>50>200 rising = strong uptrend) matter as much as the absolute values.
- **`calculate_relative_strength`** (`benchmark="SPY"`, `lookback_days` matched to horizon) — RS line trend, beta, and out/under-performance vs SPY. A stock can be green while bleeding relative strength — that's a tell the move is weak.
- **`get_momentum_indicators`** — ADX/DMI for **trend strength** (is the trend real or chop?), plus Stochastic/Williams %R for over-extension. ADX < 20 = no trend → neutral structures favored; ADX > 25 with rising DMI = trend has legs.
- **`get_multi_timeframe_alignment`** (`indicators=["sma","rsi","macd"]`, `timeframes` per horizon) — confirms whether daily/weekly/monthly agree. Disagreement across timeframes is itself a signal (often → favor neutral/defined-risk over directional).
- Optional but high-value: **`detect_volume_anomalies`** (unusual volume = conviction behind a move), **`get_advanced_volume_indicators`** (OBV/accumulation-distribution for stealth buying/selling).

### Options & event context (`mcp__unusual-whales__*`)

- **`get_options_chain`** for the relevant expiry — pull **current marks, IV, and greeks** for the user's actual strikes so you can value the position and see net delta/theta/vega *now*. Note **IV level and rank context**: a short-vol position entered when IV was high behaves very differently once IV has crushed.
- **`get_greek_exposure_by_strike`** — dealer GEX/DEX near the position's strikes; tells you whether dealer hedging is likely to pin, support, or accelerate price through your strikes.
- **`get_max_pain`** — magnet into monthly OPEX; relevant when expiry is near and strikes straddle max pain.
- **Earnings / event check** — **`get_earnings_calendar`** (stock-data) or `get_earnings_report` / `get_upcoming_earnings` (UW). **An earnings date inside the position's life is decisive**: it can validate (long premium into a vol expansion) or destroy (short premium through an unexpected move, or long premium suffering IV crush after). Always check; always surface it.

If a call errors, continue with the rest and note the gap. Don't abort the whole review because one endpoint failed.

## Step 2 — Form the stock verdict

Score the signals into a single, horizon-appropriate read. Use the scorecard in `references/methodology.md` §2. The short version: each signal votes **bullish / neutral / bearish**, you weight by the bias horizon, and you land on one of:

**Strong Bullish · Bullish · Neutral-leaning-Bullish · Neutral / Rangebound · Neutral-leaning-Bearish · Bearish · Strong Bearish**

Always state **conviction** (high/medium/low) and the **dominant evidence**. Call out conflicts explicitly — "price above all MAs but RS vs SPY rolling over and MACD crossing down" is a *low-conviction bullish*, and that low conviction changes the recommendation (favor defensive modifications over holding full size). Note any **divergence** (price up, momentum/volume not confirming) — these are early thesis-break warnings.

## Step 3 — Reconcile thesis vs. read and decide

Lay the position's directional/vol thesis next to the stock verdict. The decision matrix lives in `references/methodology.md` §3; the logic:

- **Aligned and healthy** (bullish position + bullish read, not yet at a management target, risk/reward still good) → **KEEP**. Say what would change your mind (the invalidation level / signal to watch).
- **Thesis broken** (position bias opposes the stock read at the position's horizon, *or* the structural invalidation level is breached) → **CLOSE**, or **CLOSE-AND-FLIP** if the read is now strong in the opposite direction and the user wants to stay engaged. Don't "hope" a broken directional trade back.
- **Thesis intact but expression is wrong** → **MODIFY**. Pick the *minimal effective* adjustment:
  - **Roll out (time)** — thesis right, running out of runway → same strikes, later expiry. Watch that you're not paying up into an earnings/IV event.
  - **Roll directionally (strikes)** — view shifted in magnitude but not sign → move strikes up/down to re-center the profit zone.
  - **Roll the untested side** — credit spreads / iron condors where one side is threatened → roll the safe side closer for more credit, or roll the tested side away.
  - **Convert the structure** — e.g., long call bleeding theta → spread it off into a vertical to cut cost/theta; a vertical that's now directional-wrong but the stock went neutral → convert to a calendar/condor to harvest the new regime; a tested credit spread → roll into a different expiry/width. Convert when the *current* structure is the wrong tool for the *current* read.
  - **Defensive add** — hedge a tested side (buy protection, add a wing to define risk) when you want to stay in but cap damage.
- **Management-target hit** (defined-risk winner near max profit, short-premium at ~50% of credit, 21-DTE gamma-risk zone on short options) → recommend taking it / rolling per the mechanical rules in §4, *independent* of the directional read. A winner at target is a winner; don't get greedy because the read is still favorable.

Whenever you recommend MODIFY-by-reopening or CLOSE-AND-FLIP, specify the **alternative structure concretely** — exact legs (long/short, strike, expiry), why that structure fits the *current* read better (directional fit, vega fit given IV, theta posture, defined risk), and the rough cost/credit and max-loss profile. Don't say "consider a put spread" — say "close the call spread; if you want to stay short, a 30-DTE 95/90 put debit spread re-expresses the now-bearish read with defined risk and positions below the 50-DMA that just broke."

## Report structure — concise by default

**Do all the analysis in Steps 0–3, but report only the decision and the few facts that drive it.** Default to the terse format below. Do **not** dump the full stock-read / position-read breakdown unless the user asks for it (e.g. "why?", "elaborate", "full read", "details", "show your work"). Lead with the action line — it's the answer; everything else supports it.

```text
Recommended action: CLOSE / ADJUST / KEEP

Key reasons:
1) [specific signal — e.g. "50-DMA crossed below 200-DMA (death cross)"]
2) [specific signal — e.g. "RS vs SPY −10% and falling; stock is a market laggard"]
3) [specific signal — e.g. "BE is +13% away with 68 DTE; needs a full trend reversal"]

What would change my opinion: If [price closes above/below $X, or named signal flips], then [new action].

Re-open position as follows: TICKER longK/shortK M/DD    ← ONLY if action = ADJUST

Levels:
Major resistance: $X, $Y, $Z
Major support: $X, $Y, $Z

Stats:
BE point: $X
% to BE: +/−X% from spot
Probability of reaching BE: ~X%
Probability of reaching short strike: ~X%
```

Rules for the concise format:
- **Action labels are CLOSE / ADJUST / KEEP** (ADJUST = the MODIFY logic in Step 3). Keep key reasons to **3–5 punchy bullets** — name the specific signal, not a paragraph.
- **"Re-open position as follows"** appears **only when the action is ADJUST.** Use the user's shorthand `TICKER longK/shortK M/DD` (add put/call and the rough net debit/credit when not obvious from context).
- **Levels** — 2–3 major resistance and 2–3 major support, drawn from the volume profile (POC, VAH/VAL, HVNs), prior swing highs/lows, round numbers, GEX walls, and max-pain. These are zones; round sensibly.
- **Stats** describe the position the user would *end up holding*: the **current** position for KEEP/CLOSE, the **re-opened** structure for ADJUST.
  - **BE point** — break-even at expiry (debit spread: long strike + net debit; credit spread: short strike ∓ net credit; single leg: strike ± premium).
  - **% to BE** — signed % move in the underlying from current spot to BE.
  - **Probability of reaching BE / short strike** — estimate from the chain: P(expire beyond a strike) ≈ |delta| of the option at that strike (interpolate from `get_options_chain`); for P(*touch* before expiry) use ≈ 2× that. Give a rough, model-implied number ("~28%"), not false precision. If the chain data is missing, say so instead of guessing.
- **Always surface an earnings (or other major event) date inside the trade's life** even in concise mode — fold it into a key reason or the "what would change my opinion" line.
- Skip the not-financial-advice disclaimer in concise mode (keep it only in the expanded format).

### Expanded format — only when the user asks for detail

If the user requests more (or the position is complex enough that a one-liner would mislead), lead with the same action line, then expand:

```markdown
# $TICKER — Position Review
*Structure: [e.g., 100/105 call debit spread] • Expiry: YYYY-MM-DD (NN DTE) • Spot: $X.XX • As of: YYYY-MM-DD*

## ⟶ Verdict: KEEP / CLOSE / ADJUST
**[One-sentence recommendation.]**

## Stock read — [Bullish/Bearish/Neutral], [conviction] (horizon: [daily/weekly/...])
- **Trend / MAs**: price vs 20/50/200, stacking + slopes
- **Momentum**: RSI [level/trend], MACD [line vs signal, histogram], ADX [trend strength]
- **Relative strength vs SPY**: [out/under-performing, RS trend]
- **Volume / profile**: [POC & value area vs spot, accumulation/distribution, anomalies]
- **Net read**: [the synthesized verdict + the one fact that matters most]
- **Conflicts / divergences**: [anything pulling the other way → why conviction is what it is]

## Position read
- **Embedded thesis**: [bullish/bearish/neutral + long/short vol], DTE, defined/undefined risk
- **Greeks now**: net delta / theta / vega (from the chain)
- **P&L**: [current vs entry; % of max profit/loss; distance to break-even(s)]
- **Where spot sits in the structure**: [ITM/ATM/OTM legs; inside/outside profit zone]
- **Event risk**: [earnings/ex-div inside the life of the trade — date + implication], IV context

## Why this verdict
[2–4 sentences connecting the stock read to the position. Name the specific mismatch or alignment.]

## The action (be specific)
- **If KEEP**: what to hold, the invalidation level/signal that would flip this to CLOSE, and any profit-take target.
- **If CLOSE**: close it; the reason; and (optional) the re-entry structure if they want to stay engaged.
- **If ADJUST**: the exact adjustment — legs to close, legs to open (strike/expiry/long-short), resulting net debit/credit, new max loss & break-evens, and the new thesis it expresses.

## Caveats
- [Missing data, stale options data into OPEX, no cost basis provided, assumptions made about risk tolerance, etc.]
- Not financial advice — sizing and whether this fits your risk are yours to judge.
```

## Important judgment calls

- **Match the bias timeframe to DTE — always.** The most common analytical error is judging a 90-DTE position by today's candle. Pull the timeframe that matches the trade's life.
- **A position's thesis is what the structure *pays off on*, not what the user hoped.** A short put spread is neutral-to-bullish and short vol — judge it on that, even if the user "feels bearish."
- **Mechanical management targets override the directional read.** 50% on a credit spread, ~21 DTE on short options, near-max on a defined-risk debit spread → take/roll it regardless of how good the chart looks. Greed on a winner is the tax.
- **Respect IV / vega.** Recommending a long-premium roll into elevated pre-earnings IV (that will crush) or a short-premium hold through earnings is a real, separate risk from direction. State the vol posture.
- **Undefined-risk positions (naked short options, short strangles) get a stricter bar.** When the read turns against an undefined-risk short, lean toward CLOSE/defend rather than "wait" — tail risk is asymmetric.
- **Don't over-trade.** Every MODIFY costs spread/commission and adds risk. If KEEP is genuinely best, say KEEP. The goal is the right action, which is sometimes "do nothing."
- **Don't over-precision strikes/levels.** Round sensibly; a "level" is a zone.
- **Be explicit about missing data.** No cost basis → P&L logic is degraded; thin options data → greeks/IV read is soft. Flag it in Caveats, don't paper over it.
- **This is analysis, not advice.** Give a clear recommendation (the user asked for one) but close with the not-advice caveat and note you can't see their full account/risk picture.

## Followups to offer (one line each, don't run unprompted)

- Model the P&L/break-evens of the proposed modification vs. holding.
- Re-run the read at a different horizon (e.g., if they're thinking of rolling further out).
- Pull a fresh `stock-levels-finder` read to place stops/targets/roll strikes on real levels.
