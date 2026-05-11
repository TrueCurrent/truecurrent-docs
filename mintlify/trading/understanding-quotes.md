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

**Quantity.** The size the maker is willing to fill at this price. In most cases this will match your requested quantity. If a maker can only partially fill your order, TrueCurrent selects the maker who offers the best combination of price and quantity. TrueCurrent also supports partial fills from one or more makers.

**Expiry.** A timestamp indicating how long the quote is valid. Live quotes expire quickly, typically after about 2 seconds. If the selected quote expires before settlement, the trade is rejected and a fresh quote must be requested. Short expiry windows protect makers from stale prices in fast-moving markets. Without them, makers would need to quote wider spreads to compensate for the risk of being held to an old price.

**Maker address.** The Injective wallet address of the maker offering the quote. This is visible onchain after settlement.

**Signature.** A cryptographic signature from the maker's private key covering all the above fields. The smart contract verifies this signature as part of settlement, ensuring the maker cannot repudiate or alter their quoted terms.

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