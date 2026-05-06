---
title: "RFQ explained"
description: "Learn how RFQ trading works on TrueCurrent, how liquidity providers price quotes, and how RFQ differs from AMMs and order books."
updatedAt: "2026-05-05"
---

Request for Quote (RFQ) is a trading model where you request executable prices from multiple liquidity providers before committing to a trade. On TrueCurrent, RFQ brings competitive pricing onchain: quotes are signed, time-limited, and settled only if they match your trade parameters.

RFQ has been used in institutional markets like bonds, FX, and derivatives for decades. TrueCurrent adapts that model for onchain perpetuals, combining professional liquidity with transparent, self-custodial settlement.

---

## RFQ vs. AMM

Automated Market Makers (AMMs) like Uniswap price trades using a mathematical formula – typically constant product (`x * y = k`). This works well for small trades, but has serious drawbacks for larger ones:

- **Price impact.** Every trade moves the pool's price. The larger the trade, the worse the execution price.
- **MEV exposure.** Because trades are predictable and public before confirmation, bots can front-run or sandwich your transaction.
- **Formula-based pricing.** AMMs can't react to off-chain information; they only know their pool balances.

In TrueCurrent's RFQ model, prices are set by professional liquidity providers who monitor real-time market data across centralized and decentralized venues. They can price each trade based on actual current conditions, not a formula. The result:

- **Fixed-price execution.** Your price is fixed at quote time. It doesn't move between when you see the quote and when it settles.
- **MEV resistance.** Signed quotes are atomic – there's nothing for a bot to extract from the price discovery process.
- **Better prices for larger trades.** Market makers can offer tighter spreads on larger sizes than any AMM pool.

---

## RFQ vs. order books

Traditional onchain order books (like the one Injective natively provides) match passive limit orders. This is more efficient than AMMs for liquid markets, but RFQ is still meaningfully different:

- **Active vs. passive liquidity.** Limit orders sit idle until matched. RFQ liquidity providers actively respond to each trade request, so you always get a fresh, competitive price.
- **Dedicated quote response.** On a busy order book, your market order competes with others. With RFQ, you have a dedicated quoting window.
- **Better for large sizes.** A large market order on an order book walks up the book, paying progressively worse prices. RFQ market makers can absorb larger sizes at a single price.

---

## How liquidity providers price quotes

When a liquidity provider receives your RFQ request, they consider several factors:

1. **Current market price** from various reference venues
2. **Their existing inventory** – a maker who is already long BTC may offer a better price to reduce that position
3. **Market volatility** – wider spreads in volatile markets, tighter in calm ones
4. **Order size** – pricing adjusts for the quantity being requested
5. **Competition** – knowing they're competing against other makers incentivizes tighter quotes

This real-time, size-aware pricing is fundamentally superior to static formula pricing for most trading scenarios.

---

## Quote expiry and settlement timing

Every quote includes an expiry timestamp. For live quotes, expiry is typically 2 seconds: Short enough to prevent stale prices from lingering, but long enough for settlement to confirm.

When TrueCurrent accepts a quote within your specified parameters, the onchain settlement must happen before the quote expires. The TrueCurrent contract verifies the expiry as part of settlement validation. If a quote expires before settlement confirms, the transaction will be rejected and you'll need to request a new quote.
