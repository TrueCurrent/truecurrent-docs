---
title: "Understanding quotes"
description: "--"
updatedAt: "2026-04-06"
---

When you request a trade on TrueCurrent, market makers respond with quotes. Understanding what a quote contains – and what it commits to – helps you trade more effectively.

---

## What a quote contains

Every quote from a market maker includes:

**Price.** The exact price at which the market maker will trade with you. This is the fill price you'll see in your trade history. It does not change between when the quote is returned to you and when you accept it (within the expiry window).

**Quantity.** The size the market maker is willing to fill at this price. In most cases this will match your requested quantity. If a maker can only partially fill your order, TrueCurrent selects the maker who offers the best combination of price and quantity.

**Expiry.** A timestamp indicating how long the quote is valid. Quotes typically expire 30 seconds after they are signed. If you don't accept before expiry, the quote becomes invalid and a new one must be requested.

**Maker address.** The Injective wallet address of the market maker offering this quote. This is visible onchain after settlement.

**Signature.** A cryptographic signature from the market maker's private key covering all the above fields. The smart contract verifies this signature as part of settlement, ensuring the maker cannot repudiate or alter their quoted terms.

---

## How the best quote is selected

TrueCurrent evaluates all quotes received during the 2-second window and presents the best one:

- For **long** positions: the quote with the **lowest price** (you're buying, so lower is better)
- For **short** positions: the quote with the **highest price** (you're selling, so higher is better)

If multiple quotes tie on price, the one with the earlier expiry timestamp (i.e., the quote that arrived first) is selected.

---

## Quote validity and expiry

Once you see a quote on screen, a countdown timer shows how long you have to accept it. When the timer reaches zero, the quote is expired and can no longer be settled onchain.

This expiry mechanism protects market makers from being locked into stale prices in fast-moving markets. If a quote expires before you accept, simply request a new one.

**Why quotes expire quickly:** Perpetuals markets can move significantly in seconds. A market maker quoting a price for INJ/USDT needs to know that if the price moves sharply before settlement, they won't be stuck trading at a price that's now well off-market. Short expiry windows protect market makers and, in turn, keep the overall RFQ system functioning – without them, makers would widen spreads to account for the risk of stale quotes.

---

## What happens if no quotes are received?

If no market maker responds within the 2-second window, TrueCurrent automatically routes your order to Injective's onchain order book. In this case:

- Your order executes as a market order against the current order book
- The price guarantee of the RFQ quote does not apply – you may experience some slippage
- Your worst price setting still protects you from extreme slippage

If the order book also cannot fill your order (insufficient liquidity at or better than your worst price), the order is cancelled and your margin is returned.

---

## Reading the quote in the UI

When a quote appears on the trading interface, you'll see:

- **Fill price** – the exact execution price
- **Price vs. mark** – how the quote price compares to the current mark price (positive = better than mark)
- **Estimated fee** – if any protocol fee is included in the spread
- **Quote expiry** – countdown in seconds
- **Market maker** – shown as a shortened address (full address visible onchain)

Review these before accepting. In most cases, the quoted price will be at or very close to the current mark price.
