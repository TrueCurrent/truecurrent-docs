---
title: "Smart contract"
description: "Reference documentation for TrueCurrent's CosmWasm smart contract including AcceptQuote entrypoint, settlement validation, authz integration, and query methods for market maker registry."
updatedAt: "2026-04-06"
---

TrueCurrent's core logic is implemented as a CosmWasm smart contract deployed on Injective. This page documents the contract's main entrypoints, the settlement flow, and how to interact with it directly.

---

## Contract address

| Network | Address |
|---------|---------|
| Injective Testnet | *(TBD – available after testnet deployment)* |
| Injective Mainnet | *(TBD – available after mainnet deployment)* |

---

## Primary entrypoint: AcceptQuote

The `AcceptQuote` execute message is the main settlement entrypoint. It's called by the taker when accepting a market maker's quote.

**Message structure:**

```json
{
  "accept_quote": {
    "quotes": [
      {
        "maker": "inj1...",
        "margin": "200",
        "quantity": "100",
        "price": "4.4550",
        "expiry": 1708000800000,
        "signature": "0xabc123..."
      }
    ],
    "rfq_id": "1708000700000",
    "market_id": "0xabc123...",
    "direction": "long",
    "margin": "200",
    "quantity": "100",
    "worst_price": "4.7000",
    "unfilled_action": { "market": {} }
  }
}
```

**Fields:**

| Field | Description |
|-------|-------------|
| `quotes` | Array of one or more maker quotes. Currently one quote per settlement. |
| `quotes[].maker` | Market maker's Injective address |
| `quotes[].margin` | Maker's margin commitment |
| `quotes[].quantity` | Quantity the maker is filling |
| `quotes[].price` | Maker's quoted price |
| `quotes[].expiry` | Quote expiry (Unix milliseconds) |
| `quotes[].signature` | Maker's cryptographic signature |
| `rfq_id` | String representation of the RFQ identifier |
| `market_id` | Injective perpetual market ID |
| `direction` | Taker's direction (`long` or `short`) |
| `margin` | Taker's margin (USDT) |
| `quantity` | Taker's requested quantity |
| `worst_price` | Taker's price limit |
| `unfilled_action` | What to do with any unfilled quantity |

**`unfilled_action` options:**

- `{"market": {}}` – Route unfilled quantity to the Injective order book as a market order
- `{"cancel": {}}` – Cancel any unfilled quantity *(confirm if supported)*

---

## Onchain validation

When `AcceptQuote` is called, the contract performs these checks in order:

1. **Quote not expired** – `block_time < quote.expiry`
2. **Signature valid** – Reconstruct the signed message from parameters and verify against `quotes[].signature` using the maker's address
3. **Worst price respected** – For longs: `quote.price <= worst_price`; for shorts: `quote.price >= worst_price`
4. **Maker is whitelisted** – `quotes[].maker` is in the approved maker registry
5. **Sufficient margin** – Both taker and maker subaccounts have adequate margin

If any check fails, the transaction reverts with an appropriate error message.

---

## Settlement via authz

On successful validation, the contract uses `MsgPrivilegedExecuteContract` (authorized via `authz`) to interact with Injective's exchange module:

1. Opens the taker's position (long or short, per request)
2. Opens the maker's opposing position (short or long)
3. Routes any unfilled quantity to the order book (if `unfilled_action: {"market": {}}`)

Both positions are settled atomically in the same block. There is no partial state where one side has a position and the other doesn't.

---

## Query entrypoints

The contract exposes several query methods for reading state:

**List approved market makers:**
```json
{ "list_makers": {} }
```

**Check if an address is an approved maker:**
```json
{ "is_maker": { "address": "inj1..." } }
```

*(Additional queries TBD as contract interface is finalized)*

---

## Contract source code

The TrueCurrent contract source code will be published at *(GitHub link TBD)* before mainnet launch. The contract is built with CosmWasm and the Injective CosmWasm SDK.

A security audit is planned before mainnet deployment. Audit report will be linked here upon completion.
