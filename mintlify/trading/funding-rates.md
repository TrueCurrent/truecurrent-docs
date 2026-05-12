---
title: "Funding rates"
description: "Learn how funding rates work on TrueCurrent perpetuals, when funding payments are exchanged between longs and shorts, and how funding affects position margin."
updatedAt: "2026-05-05"
---

Perpetual contracts have no expiry, so they use **funding rates** to keep the perpetual price anchored to the underlying spot price. Funding is a periodic payment exchanged between long and short positions.

---

## How funding works

Once an hour, a funding payment is calculated and exchanged between all open positions:

- **Positive rate:** longs pay shorts. The perpetual is trading at a premium to spot, and the payment incentivizes traders to short, pushing the perpetual price back toward spot.
- **Negative rate:** shorts pay longs. The perpetual is at a discount; longs are subsidized to absorb the selling pressure.

The payment is automatically applied to your margin each settlement. You don't need to do anything.

---

## How the rate is calculated

TrueCurrent inherits Injective's exchange-module funding mechanism. The rate is derived from the time-weighted average of the per-block premium between trade VWAP and the oracle price, normalized to an hourly cadence and clamped per market.

The full formula, the sampling cadence, the funding-cap parameter, and the staleness circuit breakers are all defined and maintained by Injective. Read the canonical specification here:

- [Injective docs — Margin & funding rates](https://docs.injective.network/defi/trading/margin-funding-rates#funding-rates)

---

## What you see in the UI

In the market header, TrueCurrent shows:

- **Current funding rate** – the rate that will be applied at the next hourly settlement
- **Countdown to next funding** – time until that settlement

A positive rate means longs are currently paying shorts. A negative rate means shorts are paying longs. Funding rates close to zero indicate the perpetual is well-anchored to spot, which is the healthy state.

### Converting the hourly rate

| Frequency | Formula |
| --- | --- |
| Hourly (displayed in UI) | rate as shown |
| Daily | hourly rate × 24 |
| Annualised (indicative) | hourly rate × 8,760 |

**Worked example:** an hourly rate of \+0.005% equals \+0.12% per day and roughly \+43.8% annualised – a moderate bull-market condition.

### Typical ranges by market condition

| Condition | Hourly rate | Daily rate | What it signals |
| --- | --- | --- | --- |
| Calm / sideways | ±0.000% – ±0.003% | ±0.00% – ±0.07% | Perp well-anchored; balanced longs and shorts |
| Trending (mild) | ±0.003% – ±0.010% | ±0.07% – ±0.24% | Market leaning one way; moderate premium or discount |
| Trending (strong) | ±0.010% – ±0.030% | ±0.24% – ±0.72% | Persistent directional pressure; noticeable holding cost |
| Highly volatile / squeezed | ±0.030%\+ | ±0.72%\+ | Extreme dislocation; holding cost can be significant |

These ranges are indicative. Rates during dislocations or squeezes can briefly exceed the volatile range before the per-market cap takes effect.

---

## Funding rate history

Historical rates are accessible through:

- **Injective explorer** – per-market funding history with settlement timestamps
- **Indexer API** – the `/derivatives/markets/{marketId}/funding` endpoint returns historical funding-rate snapshots over any date range

Reviewing funding history before entering a long-duration position helps set realistic expectations for holding cost, particularly in markets with a track record of persistent positive or negative rates.

---

## Impact on your position

**If you're long and funding is positive:** you pay a small amount per hour to short holders. This is a cost of holding a long when the perpetual trades at a premium.

**If you're short and funding is negative:** you pay a small amount per hour to long holders. This is a cost of holding a short when the perpetual trades at a discount.

Funding is automatically deducted from (or added to) your margin each hour. If your funding payments are large enough relative to your remaining margin, they can accelerate your approach to liquidation — see [Liquidation](/trading/liquidation).

---

## Strategic implications

Funding rates carry information about market sentiment. A persistently high positive rate suggests the market is heavily long and traders are paying a meaningful premium to hold long positions — historically a sign of elevated speculative interest that sometimes precedes reversals.

If you are paying significant funding on a position, factor that into your holding cost. For long-duration positions, funding can become a meaningful drag.