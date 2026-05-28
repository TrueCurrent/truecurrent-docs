---
title: "RFQ explained"
description: "Learn how RFQ trading works on TrueCurrent, why makers quote each request, and how signed quotes differ from AMMs and order books."
updatedAt: "2026-05-27"
---

Request for Quote (RFQ) is a trading model where you request executable prices from liquidity providers before committing to a trade. On TrueCurrent, those prices are signed by makers, delivered through an indexer, and settled onchain only if they match your trade parameters.

## RFQ vs. order books vs. AMMs

| Model | How price is found | Execution profile | Best fit |
| --- | --- | --- | --- |
| RFQ | Makers compete to quote a specific request | Firm, signed quote for a short expiry window | Size-aware derivatives trades where execution quality matters |
| Order book | Takers cross resting bids and asks | Public depth, but large trades can walk the book | Continuous markets with visible passive liquidity |
| AMM | Pool balances and a pricing formula set the price | Always available pool liquidity, with formula-driven slippage | Passive spot liquidity and smaller flow |

TrueCurrent uses RFQ for onchain perpetuals because it lets makers price each request with current market data, inventory, and risk while keeping settlement transparent and self-custodial.

The important distinction is simple:

- **Price discovery happens offchain** through a short maker competition.
- **Settlement happens onchain** through the TrueCurrent RFQ contract and Injective exchange module.

---

## The five-message cycle

```mermaid
sequenceDiagram
    actor Taker
    participant Indexer
    participant Maker
    participant Chain

    Taker->>Indexer: 1. RFQ request
    Indexer->>Maker: 2. Broadcast request
    Maker->>Indexer: 3. Signed quote
    Indexer->>Taker: 4. Quote delivery
    Taker->>Chain: 5. AcceptQuote
```

As a trader, you receive quotes and settle the one you accept. As a maker, you do not submit a transaction per trade; you sign prices offchain and the contract verifies them if a taker accepts.

---

## RFQ vs. AMM

Automated Market Makers price trades from pool balances and a formula such as `x * y = k`. That is useful for passive spot liquidity, but it creates tradeoffs:

- Every trade moves the pool price.
- Large trades pay progressively worse execution.
- Public pending transactions can expose traders to MEV.
- The pool cannot react to external markets except through arbitrage.

TrueCurrent's RFQ model asks professional makers to quote each request using current market data, inventory, volatility, funding, and order size. The quote is firm for its short expiry window. If it settles, the contract enforces the signed price.

---

## RFQ vs. order books

An order book matches against passive resting liquidity. That gives traders visible depth, but it also creates tradeoffs:

- Large orders can walk through multiple price levels.
- Publicly visible resting liquidity may disappear before execution.
- Execution quality depends on the book state at the moment the order lands.
- Liquidity providers quote passively and wait to be lifted or hit.

RFQ uses active liquidity instead. Makers respond to your specific request, can quote one executable size-aware price, and compete for the trade before you settle.

TrueCurrent still settles through Injective's exchange module, but the public execution flow is RFQ-only: a trade fills from signed maker quotes, not from a different hidden price path.

---

## How makers price

When a maker receives your RFQ request, it typically considers:

1. Current mark price and external reference markets
2. Market volatility and quote expiry
3. Requested margin, quantity, and `worst_price`
4. Inventory skew and hedge cost
5. Funding carry
6. Available margin and per-market risk limits
7. Competition from other makers

The maker can decline by not quoting. If no maker returns an acceptable quote, no trade settles and you can request again with fresh market data.

---

## Quote expiry

Live quotes are intentionally short-lived. Makers must submit quotes with at least 1500 ms of validity, and many use longer expiries to increase the chance their quote can be selected and settled. Short expiries protect makers from stale-price exposure and keep spreads tighter for traders.

The contract checks expiry at settlement time. If a quote expires before the transaction lands, that quote is skipped. If every submitted quote is skipped or invalid, settlement fails with no fill.

---

## Why signed quotes matter

Signed quotes bind the maker to exact terms:

- Market
- RFQ ID
- Taker address and direction
- Maker address and subaccount nonce
- Margin and quantity
- Price
- Expiry
- Minimum fill quantity
- EIP-712 domain for the chain and RFQ contract

The indexer cannot change those terms. The taker cannot settle a worse price than their `worst_price`. The maker cannot be filled at a price it did not sign.

For the technical sequence, see [How RFQ works](/technical/how-rfq-works).
