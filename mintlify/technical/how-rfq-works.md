# RFQ explained

Request for Quote (RFQ) is a trading model where a buyer or seller solicits price quotes from one or more liquidity providers before committing to a trade. Unlike automated pricing models, RFQ involves human or algorithmic market makers actively pricing each individual trade.

RFQ has been the backbone of institutional trading in traditional finance for decades – used in bond markets, FX, and derivatives. TrueCurrent brings this model onchain, combining the pricing quality of institutional markets with the transparency and self-custody of DeFi.

---

## RFQ vs. AMM

Automated Market Makers (AMMs) like Uniswap price trades using a mathematical formula – typically constant product (`x * y = k`). This works well for small trades, but has serious drawbacks for larger ones:

- **Price impact.** Every trade moves the pool's price. The larger the trade, the worse the execution price.
- **MEV exposure.** Because trades are predictable and public before confirmation, bots can front-run or sandwich your transaction.
- **Fixed pricing.** AMMs can't react to off-chain information; they only know their pool balances.

In TrueCurrent's RFQ model, prices are set by professional market makers who monitor real-time market data across centralized and decentralized venues. They can price each trade based on actual current conditions, not a formula. The result:

- **Zero slippage.** Your price is fixed at quote time. It doesn't move between when you see the quote and when it settles.
- **MEV resistance.** Signed quotes are atomic – there's nothing for a bot to extract from the price discovery process.
- **Better prices for larger trades.** Market makers can offer tighter spreads on larger sizes than any AMM pool.

---

## RFQ vs. order books

Traditional onchain order books (like the one Injective natively provides, and which Helix uses) match passive limit orders. This is more efficient than AMMs for liquid markets, but RFQ is still meaningfully different:

- **Active vs. passive liquidity.** Limit orders sit idle until matched. RFQ market makers actively respond to each trade request, so you always get a fresh, competitive price.
- **No queue.** On a busy order book, your market order competes with others. With RFQ, you have a dedicated quoting window.
- **Better for large sizes.** A large market order on an order book walks up the book, paying progressively worse prices. RFQ market makers can absorb larger sizes at a single price.

TrueCurrent's hybrid model captures the best of both: RFQ for primary price discovery, with the Injective order book as a fallback to maximize fill rates.

---

## How market makers price quotes

When a market maker receives your RFQ request, they consider several factors:

1. **Current mid-market price** from Injective's oracle and/or external reference venues
2. **Their existing inventory** – a maker who is already long INJ may offer a better price to reduce that position
3. **Market volatility** – wider spreads in volatile markets, tighter in calm ones
4. **Order size** – pricing adjusts for the quantity being requested
5. **Competition** – knowing they're competing against other makers incentivizes tighter quotes

This real-time, size-aware pricing is fundamentally superior to static formula pricing for most trading scenarios.

---

## Quote expiry and settlement timing

Every quote includes an expiry timestamp. Quotes are valid for a short window (typically 30 seconds), which prevents stale quotes from being used in fast-moving markets.

When you accept a quote, the onchain settlement must happen before the quote expires. The TrueCurrent contract verifies the expiry as part of settlement validation. If a quote expires before settlement confirms, the transaction will be rejected and you'll need to request a new quote.
