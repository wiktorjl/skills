# Methodology — Options Position Review

Detailed reference for `options-position-review`. Read this when the SKILL.md summary is insufficient — typically when classifying an unusual structure, scoring a conflicted stock read, or choosing the precise roll/adjustment.

## Table of contents

1. Structure taxonomy — reading thesis & greeks off the legs
2. Stock bias scorecard
3. Decision matrix — thesis vs. read → action
4. Mechanical management rules (targets that override direction)
5. Roll & adjustment mechanics
6. Alternative-structure selection (close-and-reopen)
7. IV / vega and event-risk handling
8. Worked examples
9. Data-quality checks

---

## 1. Structure taxonomy — reading thesis & greeks off the legs

Classify every position on five axes before judging it: **structure type · directional bias · vega sign · theta sign · risk definition**. The user's *feeling* about the trade is irrelevant; the payoff diagram is the thesis.

| Structure | Legs | Bias | Vega | Theta | Risk |
|---|---|---|---|---|---|
| Long call | +1 C | Bullish | Long | Pays (−) | Defined (debit) |
| Long put | +1 P | Bearish | Long | Pays (−) | Defined (debit) |
| Call debit spread (bull call) | +C lower / −C higher | Bullish | ~Neutral/slight long | Mild pay | Defined |
| Put debit spread (bear put) | +P higher / −P lower | Bearish | ~Neutral | Mild pay | Defined |
| Call credit spread (bear call) | −C lower / +C higher | Neutral-bearish | Short | Collects (+) | Defined |
| Put credit spread (bull put) | −P higher / +P lower | Neutral-bullish | Short | Collects (+) | Defined |
| Iron condor | bull put + bear call | Neutral (range) | Short | Collects | Defined |
| Iron fly | ATM short straddle + wings | Neutral (pin) | Short | Collects | Defined |
| Calendar | −near / +far, same strike | Neutral→slightly directional | **Long** (far > near) | Collects (near decays faster) | Defined-ish |
| Diagonal | −near OTM / +far | Directional + long vol | Long | Mixed | Defined-ish |
| Long straddle/strangle | +C +P | Neutral, **long vol/move** | Long | Pays hard | Defined (debit) |
| Short straddle/strangle | −C −P | Neutral, **short vol/range** | Short | Collects | **Undefined** |
| Ratio / backspread | unequal qty | Skewed directional | Varies | Varies | Often **undefined** on one side |
| Covered call | +100 shares / −C | Neutral-bullish, capped | Short (on the call) | Collects | Stock risk below |
| Cash-secured put | −P | Neutral-bullish | Short | Collects | Assignment risk |
| Collar | +shares / −C / +P | Bullish, hedged | ~Neutral | ~Flat | Defined band |

**Reading bias quickly:** net long call delta = bullish; net long put delta = bearish; deltas that cancel = neutral/vol play. **Reading vega:** net long options (more long than short, or longer-dated longs) = long vega (helped by rising IV); net short = short vega (helped by falling IV). **Reading theta:** net short premium = collecting theta (time is your friend); net long premium = paying theta (time is your enemy — you need the move *soon*).

**DTE per expiry** sets urgency: long-premium trades are racing theta (need the move now); short-premium trades *want* time to pass but face gamma risk as expiry nears (see §4). Multi-expiry structures (calendars/diagonals) — note each leg's DTE separately.

**Infer the thesis** from the above, then sanity-check it against what the user said. If they say "I'm bearish" but hold a bull put spread, surface the contradiction — the *position* is neutral-to-bullish; that mismatch may itself be the problem.

## 2. Stock bias scorecard

Each signal casts a vote: **+1 bullish / 0 neutral / −1 bearish**. Weight by horizon (see below), sum, and map to the verdict band. This is a guide, not a black box — a single decisive fact (e.g., a clean breakdown through the 200-DMA on huge volume) can override a marginal aggregate.

| Signal (source tool) | Bullish (+1) | Neutral (0) | Bearish (−1) |
|---|---|---|---|
| **Price vs MAs** (`get_technical_indicators`) | Price > 20 > 50 > 200, all rising | Mixed / flat / tangled | Price < 20 < 50 < 200, all falling |
| **Key-MA test** | Holding above rising 50/200 | Chopping around it | Lost & backtesting from below; death cross |
| **RSI** | 50–70 & rising (>70 strong but watch exhaustion) | ~45–55 flat | <50 & falling (bear divergence = early warning) |
| **MACD** | Line > signal & > 0, histogram expanding up | Around zero / flattening | Line < signal & < 0, histogram expanding down |
| **ADX/DMI** (`get_momentum_indicators`) | ADX>25, +DI>−DI | ADX<20 (no trend) | ADX>25, −DI>+DI |
| **Relative strength vs SPY** (`calculate_relative_strength`) | RS line rising, outperforming | Tracking SPY | RS line falling, lagging |
| **Volume / profile** (`get_volume_profile`, `detect_volume_anomalies`, `get_advanced_volume_indicators`) | Price above POC/VAH, up-moves on rising vol, OBV rising (accumulation) | Inside value area, no anomaly | Below POC/VAL, down-moves on rising vol, OBV falling (distribution) |

**Horizon weighting (from DTE):**
- **Short (DTE ≤ 10):** weight daily price-vs-20/50-MA, RSI, MACD, intraday volume. The 200-DMA and multi-month RS matter less.
- **Swing (10–45):** balanced; require **multi-timeframe agreement** (daily + weekly via `get_multi_timeframe_alignment`) for high conviction.
- **Long (>45 / LEAPS):** weight weekly/monthly trend, 50/200-DMA, longer-lookback RS. Daily noise is mostly irrelevant.

**Verdict bands** (after weighting): clearly positive → **Bullish** / strongly positive with confirmation → **Strong Bullish**; clearly negative → **Bearish** / **Strong Bearish**; near zero or internally contradictory → **Neutral / Rangebound**; small tilt → **Neutral-leaning-X**.

**Conviction** is separate from direction and is set by **agreement**: all signals + all timeframes pointing the same way = high; the verdict resting on one or two signals while others conflict = low. **Always report divergences** (price making highs while RSI/OBV/RS fail to confirm) — they're the earliest thesis-break signal and they cap conviction.

## 3. Decision matrix — thesis vs. read → action

Rows = position thesis (at the position's horizon). Columns = stock verdict. Cell = default action. Always modulate by §4 (management targets can force a take), DTE, P&L, and conviction.

| Position thesis ↓ / Stock read → | Bullish | Neutral | Bearish |
|---|---|---|---|
| **Bullish** (long call, bull call/put spread) | **KEEP** (watch target & invalidation) | **MODIFY** — tighten target / spread off theta / consider neutralizing | **CLOSE** (or CLOSE-AND-FLIP bearish) |
| **Neutral / short-vol** (condor, fly, short strangle, calendar) | **MODIFY** — roll tested call side / re-center upward | **KEEP** (ideal — let theta work) | **MODIFY** — roll tested put side / re-center down |
| **Bearish** (long put, bear put/call spread) | **CLOSE** (or CLOSE-AND-FLIP bullish) | **MODIFY** — tighten / spread off | **KEEP** (watch target & invalidation) |
| **Long-vol / long-move** (straddle/strangle) | partial fit — **MODIFY** toward the trending side (lean directional) | **CLOSE/REDUCE** — range kills long premium via theta | partial fit — **MODIFY** toward downside |

Overriding modifiers, applied after the matrix:
- **Conviction low** → prefer the *defensive* version of the cell (reduce size, spread off, define risk) over a full KEEP or a full flip.
- **Management target hit (§4)** → take/roll the winner regardless of cell.
- **Structural invalidation breached** (price through the level/MA the thesis depended on) → upgrade toward CLOSE even if the indicator aggregate is only mildly against you.
- **Undefined risk + read turning against** → bias to CLOSE/defend, never "wait and see."
- **Event inside the trade (§7)** → may force an action purely on vol/event grounds.

## 4. Mechanical management rules (targets that override direction)

These are well-worn defaults (tastytrade-style mechanics). Treat as defaults the user can override, but *state* them — a winner at target is a take.

- **Short-premium defined risk (credit spreads, iron condors/flies):** take profit at **~50% of max credit**. Roll or close the tested side; don't hold for the last few pennies against open risk.
- **Long-premium defined risk (debit spreads):** manage at **~50–75% of max value**; near-max-profit with little extrinsic left = close (no reason to hold full risk for the last sliver).
- **21-DTE rule on short options:** gamma/assignment risk accelerates inside ~21 DTE. Close or roll short-dated short-premium positions out — don't ride them to expiry hoping. (Long-premium directional trades have the *opposite* clock — see below.)
- **Long premium & theta:** a long single option / straddle that hasn't worked with <2–3 weeks left and IV not expanding is usually a bleed-out — cut or spread it off; don't pay theta to "hope."
- **Tested short strike going ITM:** decide *before* expiry week — roll out/down (puts) or out/up (calls) for a credit if the longer thesis holds, else close. Avoid pin/assignment surprises into OPEX.
- **Profit-take asymmetry:** there's no shame in closing a directional winner early when the read's conviction drops, even below a "target." Capital preserved > thesis ego.

## 5. Roll & adjustment mechanics

Pick the **minimal effective** adjustment — each touch costs spread/fees and changes the risk graph.

- **Roll out (calendar roll):** close near expiry, open same strikes later. Use when the *direction is right but you need time*. Verify net cost; for credit positions, roll **for a net credit** when possible (never pay to extend a losing short-premium trade without a reason). Check you're not rolling *into* an earnings/IV event.
- **Roll up / down (strike roll):** move strikes to re-center the profit zone when magnitude shifted. Rolling a long debit spread up locks some gain and resets break-even higher; rolling a short spread away from price buys safety, usually for less credit.
- **Roll the untested side (condors/strangles):** when one side is threatened and the other is safe, roll the *safe* side closer to collect more credit (widens the cushion / lowers break-even on the tested side). Only do this if you still believe in the range.
- **Spread it off (convert a long single into a vertical):** sell a further option against a long call/put to cut cost and theta when you still want direction but momentum stalled. Caps upside; buys staying power.
- **Define risk (add a wing):** turn a naked short or a tested side into a defined-risk spread by buying protection. Use when the read turned but you want to stay in with a hard floor.
- **Reduce / leg out:** close part of the position to take risk down when conviction drops but isn't zero.
- **Convert the regime:** when the stock's *character* changed (trending → ranging or vice versa), the structure may be the wrong tool. Trend→range: a directional vertical → calendar/condor to harvest theta. Range→trend (breakout): a short condor → close the threatened side and let the other run, or flip to a directional debit spread.

For every roll, recompute and report: **new net debit/credit, new break-even(s), new max loss/profit, new DTE.** A roll that "feels" safer but worsens the risk graph is not an improvement.

## 6. Alternative-structure selection (close-and-reopen)

When MODIFY means "this structure is the wrong tool — close it and open a better one," choose the replacement by matching four dials to the *current* read:

1. **Direction** — bullish/bearish/neutral from §2.
2. **Conviction → definition & width** — high conviction → tighter/more directional (long option or narrow debit spread); low → wider, more forgiving, or defined-risk neutral.
3. **IV posture → vega sign** — IV high/rich (high IV rank) → prefer **short premium** (credit spreads, condors) to sell expensive vol; IV low/cheap → prefer **long premium** (debit spreads, calendars) to buy cheap vol. Don't buy expensive premium or sell cheap premium without a reason.
4. **Event timing → theta/expiry** — avoid being short premium through binary events unless that's the explicit trade; pick expiry so the thesis has runway but you're not paying for dead time. Place short strikes beyond expected move / key levels; place long strikes where the read says price is going.

Then state the concrete replacement: exact legs, net debit/credit, max loss, break-evens, and the one sentence on *why this structure fits the current read better than the old one*. Tie strikes to real levels — pull `stock-levels-finder` if precise levels matter.

## 7. IV / vega and event-risk handling

- **Always locate earnings (and ex-div for covered calls/short puts) within the trade's life.** An earnings date inside the position is decisive: long premium can win on the vol expansion but lose to **post-event IV crush**; short premium faces an undefined-by-vol gap move. Surface the date and the implication every time.
- **IV crush:** a long option/straddle held *through* earnings often loses even on a correct direction because IV collapses. If the read is directional and earnings is imminent, a **debit spread** (vega-reduced) or post-earnings entry may beat a naked long.
- **IV rank context:** entering short premium is attractive when IV is historically high (mean-reversion of vol in your favor); long premium is attractive when IV is low. If the position was opened in the opposite regime and IV has since normalized, that alone can justify a take/adjust even with direction unchanged.
- **Pin / OPEX:** near monthly OPEX, `get_max_pain` + dealer GEX (`get_greek_exposure_by_strike`) can pin price to high-OI strikes — relevant when your short strikes sit near max pain or a gamma wall.

## 8. Worked examples

**A — KEEP.** User holds a 30-DTE NVDA 120/130 call debit spread, entered for $4.00, now $6.50. Read: price > 20>50>200 all rising, RSI 62, MACD positive & expanding, RS vs SPY rising, price above POC. Bullish, high conviction, horizon matches. Position is ~65% of max value. Verdict: **KEEP**, but note it's near the §4 take-zone — set a target to close at ~80% / $9, and invalidation = daily close back below the 50-DMA. *(If it were 90% of max with 5 DTE, it flips to "take it.")*

**B — CLOSE / FLIP.** User holds 25-DTE bullish AAPL 200 calls. Read flipped: lost the 50-DMA on heavy volume, MACD crossed down, RSI 41 falling, RS vs SPY rolling over, price broke below value area. Bearish, medium-high conviction at this horizon. Long premium + theta + wrong direction = no repair. Verdict: **CLOSE**. If they want to stay engaged: a 30-DTE 195/185 put debit spread re-expresses the bearish read with defined risk, vega-reduced ahead of any IV pop.

**C — MODIFY (roll untested side).** User holds a 14-DTE SPY iron condor; SPY is grinding up toward the short call side, but the read is now neutral-to-mildly-bullish, ADX low (no strong trend), price chopping in value. Thesis (neutral/range) mostly intact, just off-center. Verdict: **MODIFY** — roll the *put* side up for more credit to re-center, or roll the tested call side up-and-out; if it's already near 50% on the puts, take that side. Don't close the whole thing.

**D — MODIFY (convert regime).** User holds a 45-DTE bull call spread; stock went sideways in a tight range, ADX collapsed to 14, RS flat. Direction didn't break but the *trend* did — theta is now the enemy of a long-premium directional. Verdict: **MODIFY** — convert: close the call spread, open a calendar at the range's mid (or a small iron condor around the value area) to harvest the new low-vol, rangebound regime; or, if conviction in eventual upside remains, roll the spread out to give the thesis time while reducing per-day theta.

## 9. Data-quality checks before recommending

- **Cost basis present?** Without it, P&L %, break-evens, and management-target logic are degraded — say so and lean on the directional/structural read.
- **Options data fresh & populated?** If the chain returns stale/empty greeks or zero OI on the user's strikes, the greeks/IV read is soft — flag it; don't fabricate a precise net delta.
- **Indicator series long enough?** A 200-DMA needs ≥200 sessions of history; if the name is newly listed, note the 200-DMA is unavailable and weight 20/50 + price action.
- **Split/dividend adjustments** on the bars (unexplained gaps = unadjusted data → MA/levels wrong).
- **Earnings/ex-div actually checked?** Never ship a recommendation on a multi-week option without confirming whether an event lands inside the trade.
- **Horizon actually matched to DTE?** Re-read §2 weighting before finalizing — judging a long-dated position on a daily candle is the single most common error.

If any check fails, put it in the report's **Caveats** rather than silently degrading the recommendation.
