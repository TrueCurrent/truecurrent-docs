---
title: "Understanding quotes"
description: "Understand what TrueCurrent RFQ quotes contain, how maker-signed quotes expire, and how the best executable quote is selected when you click trade."
updatedAt: "2026-05-01"
---

When you submit a trade on TrueCurrent, makers respond with signed RFQ quotes. TrueCurrent selects the best executable quote automatically, so understanding quote fields helps you understand what was filled and why a trade may not execute.

---

## What a quote contains

Every quote from a liquidity provider includes:

**Price.** The exact price the maker is willing to trade at. This becomes your fill price if the quote is selected and settled before expiry.

**Quantity.** The amount available at this price, usually matching your request. If only a partial fill is available, TrueCurrent chooses the best price/quantity offer and may fill across one or more makers.

**Expiry.** How long the quote remains valid, usually about 2 seconds. If it expires before settlement, the trade is rejected and you’ll need a fresh quote; short expiries help makers avoid stale prices and keep spreads tighter.

**Maker address.** The Injective wallet address of the maker offering the quote. This is visible onchain after settlement.

**Signature.** A cryptographic signature from the maker's private key covering all the above fields. The smart contract verifies this signature as part of settlement, ensuring the maker cannot deny or change their quoted terms.

---

## How the best quote is selected

TrueCurrent evaluates all quotes received during the 2-second collection window and selects the best one automatically:

- For **long** positions: the quote with the **lowest price** (you're buying, so lower is better)
- For **short** positions: the quote with the **highest price** (you're selling, so higher is better)

If multiple quotes tie on price, the one with the earlier expiry timestamp (i.e. the quote that arrived first) is selected.

---

## What happens if no quotes are received?

If no maker responds within the collection window, the request is cancelled and no margin is debited from your wallet – nothing settles onchain, and you pay nothing. Prices update continuously, so you can submit a new request immediately.

This is uncommon on liquid markets, where multiple makers compete on every request. It mostly happens during extreme volatility, in deep illiquid markets, or when your `worst_price` is tight enough that no maker is willing to fill at it.