---
title: "Funding rates"
description: "Understand perpetual funding rates on TrueCurrent including hourly payments between longs and shorts, rate calculations, funding factor, and strategic implications for position holding costs."
updatedAt: "2026-04-30"
---

Perpetual contracts have no expiry, so they use **funding rates** to keep the perpetual price anchored to the underlying spot price. Funding is a periodic payment exchanged between long and short positions.

---

## How funding works

Every hour, a funding payment is calculated and exchanged between all open positions:

- If the **funding rate is positive**: longs pay shorts. This happens when the perpetual trades at a premium to spot – the payment incentivizes traders to short (and thus push the perpetual price back down).
- If the **funding rate is negative**: shorts pay longs. This happens when the perpetual trades at a discount to spot.

Funding payments are automatically applied to your margin. You don't need to do anything.

---

## How the funding rate is calculated

The funding rate on TrueCurrent is derived from the difference between the perpetual mark price and the underlying index price:

```
Premium = (Mark price − Index price) / Index price

Funding rate = clamp(Premium × Funding factor, −Cap, +Cap)
```

**Premium sampling:** The premium is not taken at a single snapshot. Instead, mark and index prices are sampled once per minute throughout the settlement window. The funding rate applied at settlement is the time-weighted average of these per-minute samples. This smoothing prevents brief price dislocations from producing outsized funding payments.

**Funding factor:** Controls how aggressively the rate responds to premium. A funding factor of `1/24` maps a sustained 1% premium to approximately a 0.042% hourly rate.

**Clamping:** The resulting rate is bounded by a maximum cap in each direction. This prevents extreme payments in highly dislocated markets. Current parameters are governed by the Injective exchange module; see the [Injective exchange documentation](https://docs.injective.network) for the exact values per market.

On TrueCurrent, funding settles **hourly**. The rate displayed in the UI is the per-hour rate.

### Funding parameter reference

| Parameter | Description | Typical value |
|-----------|-------------|---------------|
| Settlement interval | How often funding is paid | 1 hour |
| Sampling method | Price samples per settlement window | Once per minute, time-weighted average |
| Funding factor | Premium-to-rate multiplier | ~1/24 (varies by market) |
| Rate cap | Maximum absolute hourly rate | Set per market via governance |

---

{/* TODO DR-144 add worked examples section */}

## Reading the funding rate display

In the market header, TrueCurrent shows:

- **Current funding rate** – the rate that will be applied at the next hourly settlement
- **Countdown to next funding** – time until the next payment

A positive rate means longs are currently paying shorts. A negative rate means shorts are paying longs.

Funding rates close to zero indicate the perpetual price is well-anchored to spot – a healthy market condition.

### Converting hourly rate to daily and annual

| Frequency | Conversion | Formula |
|-----------|------------|---------|
| Hourly (displayed in UI) | — | Rate as shown |
| Daily | × 24 | Hourly rate × 24 |
| Annual (indicative) | × 8,760 | Hourly rate × 8,760 |

**Example:** An hourly rate of +0.005% equals +0.12% per day and approximately +43.8% annualised. This is a moderate bull-market condition.

### Typical funding rate ranges by market condition

| Condition | Hourly rate range | Daily range | What it signals |
|-----------|-------------------|-------------|-----------------|
| Calm / sideways | ±0.000% – ±0.003% | ±0.00% – ±0.07% | Perp well-anchored; balanced longs and shorts |
| Trending (mild) | ±0.003% – ±0.010% | ±0.07% – ±0.24% | Market leaning one way; moderate premium or discount |
| Trending (strong) | ±0.010% – ±0.030% | ±0.24% – ±0.72% | Persistent directional pressure; noticeable holding cost |
| Highly volatile / squeezed | ±0.030%+ | ±0.72%+ | Extreme dislocation; holding cost can be significant |

These ranges are indicative. Rates during market dislocations or short squeezes can briefly exceed the volatile range before the clamp takes effect.

---

## Funding rate history

Historical funding rates for each market are accessible through:

- **TrueCurrent explorer** — per-market funding history with settlement timestamps
- **Indexer API** — the `/derivatives/markets/{marketId}/funding` endpoint returns historical funding rate snapshots and can be queried for any date range

Reviewing funding history before entering a long-duration position helps set realistic expectations for holding cost, particularly in markets with a track record of persistent positive or negative rates.

---

## Impact on your position

**If you're long and funding is positive:**
You pay a small amount per hour to short holders. This is a cost of holding a long position when the perpetual trades at a premium.

**If you're short and funding is negative:**
You pay a small amount per hour to long holders. This is a cost of holding a short when the perpetual trades at a discount.

Funding is automatically deducted from (or added to) your margin each hour. If your funding payments are large enough relative to your remaining margin, they could accelerate your approach to liquidation.

---

## Strategic implications

Funding rates carry information about market sentiment. A persistently high positive funding rate suggests the market is heavily long and traders are paying a significant premium to hold long positions – historically a sign of elevated speculative interest that sometimes precedes reversals.

If you're paying significant funding on a position, factor that into your holding cost. For long-duration positions, funding can become a meaningful drag.
