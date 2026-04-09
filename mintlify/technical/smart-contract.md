---
title: "Smart contract"
description: "Reference documentation for TrueCurrent's CosmWasm smart contract including AcceptQuote entrypoint, settlement validation, authz integration, and query methods for market maker registry."
updatedAt: "2026-04-08"
---

TrueCurrent's core logic is implemented as a CosmWasm smart contract deployed on Injective. This page documents the contract's main entrypoints, the settlement flow, and how to interact with it directly.

---

## Contract address

| Network | Address |
|---------|---------|
| Injective Testnet | `inj1vtswdey9c70n475q7q75wgmkfdw8xw4rcfeqa4` |
| Injective Mainnet | *(TBD – available after mainnet deployment)* |

---

## Primary entrypoint: AcceptQuote

The `AcceptQuote` execute message is the main settlement entrypoint. It's called by the taker when accepting a market maker's quote.

**Message structure:**

```json
{
  "accept_quote": {
    "rfq_id": 1708000700000,
    "market_id": "0xabc123...",
    "direction": "long",
    "margin": "200",
    "quantity": "100",
    "worst_price": "4.7000",
    "quotes": [
      {
        "maker": "inj1...",
        "margin": "200",
        "quantity": "100",
        "price": "4.4550",
        "expiry": { "ts": 1708000800000 },
        "signature": "Kg8z...base64..."
      }
    ],
    "unfilled_action": { "market": {} }
  }
}
```

**Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `rfq_id` | number (u64) | RFQ identifier – must be a JSON **number**, not a string |
| `market_id` | string | Injective perpetual market ID |
| `direction` | string | Taker's direction – lowercase `"long"` or `"short"` |
| `margin` | string | Taker's margin (USDC, decimal string) |
| `quantity` | string | Taker's requested quantity |
| `worst_price` | string | Taker's hard price limit – required |
| `quotes` | array | One or more maker quotes to consume atomically |
| `quotes[].maker` | string | Market maker's Injective address |
| `quotes[].margin` | string | Maker's margin commitment |
| `quotes[].quantity` | string | Quantity the maker is filling |
| `quotes[].price` | string | Maker's quoted price |
| `quotes[].expiry` | object | Wrapped enum – `{"ts": <unix_ms>}` or `{"h": <height>}` |
| `quotes[].signature` | string | Maker's secp256k1 signature, **base64-encoded** |
| `unfilled_action` | object \| null | What to do with any unfilled quantity |

**Multi-quote aggregation:** `quotes` is an array, and the contract processes every entry in submission order, filling from each until the taker's `quantity` is exhausted. A taker can submit 1 to `config.max_quotes` quotes from different makers in a single settlement and receive one aggregated position. See [Accepting quotes](/takers/accepting-quotes) for details.

**Encoding requirements:**

- **`rfq_id` must be a JSON number**, not a string. The contract field is `u64`.
- **Quote `expiry` must be wrapped** as `{"ts": <ms>}` (timestamp) or `{"h": <height>}` (block height). The indexer delivers it as a plain integer – callers must wrap it.
- **Quote `signature` must be base64**, not hex. The indexer delivers it as hex – callers must convert.

**`unfilled_action` options:**

- `{"market": {}}` – Route unfilled quantity to the Injective orderbook as an immediate-or-cancel market order (capped by `worst_price`)
- `{"limit": {"price": "<decimal>"}}` – Post the unfilled quantity as a limit order at the specified price
- `null` – No fallback; the settlement fills only what the quotes cover

---

## Onchain validation

When `AcceptQuote` is called, the contract first enforces a global check:

- **`quotes.len() <= config.max_quotes`** – the submitted quote list cannot exceed the configured maximum. Transaction reverts if exceeded.

Then, for each quote in the array in submission order, the contract performs:

1. **Maker is whitelisted** – `quote.maker` is in the approved maker registry
2. **Quote not expired** – `block_time_ms < quote.expiry.ts` (or `block_height < quote.expiry.h`)
3. **Nonce not replayed** – maker hasn't already used this `rfq_id` for this taker
4. **Signature valid** – reconstruct the canonical signed message and verify against `quote.signature` using the maker's address
5. **Price inside `worst_price`** – for longs, `quote.price <= worst_price`; for shorts, `quote.price >= worst_price`
6. **Maker has sufficient margin** – maker's available subaccount balance covers their margin commitment
7. **Fill respects maker `min_fill_quantity`** – if the maker declared a minimum fill and the contract would fill them for less, the quote is skipped

**Per-quote failures are non-fatal.** A quote that fails any check is skipped and recorded in the emitted `quote_results`. The loop continues to the next quote. The transaction as a whole succeeds if **at least one quote fills**, or if `unfilled_action` covers the full requested quantity on its own.

**Whole-transaction failures:**

- `quotes.len() > config.max_quotes`
- Zero fills and no `unfilled_action` fallback to absorb the quantity
- Taker lacks sufficient margin to cover the aggregate filled quantity
- Taker missing required `authz` grants

---

## Settlement via authz

On successful validation, the contract uses `MsgPrivilegedExecuteContract` (authorized via `authz`) to interact with Injective's exchange module:

1. Opens the taker's position (long or short, per request)
2. Opens the maker's opposing position (short or long)
3. Routes any unfilled quantity to the order book (if `unfilled_action: {"market": {}}`)

Both positions are settled atomically in the same block. There is no partial state where one side has a position and the other doesn't.

---

## Query entrypoints

The contract exposes four query methods for reading state. All are standard CosmWasm smart contract queries.

**List approved market makers** – returns the current whitelist with pagination:

```json
{ "list_makers": { "start_after": null, "limit": 100 } }
```

**Get global config** – returns `taker_fee_rate`, `maker_fee_rate`, `fee_recipient`, `max_quotes`, `max_evict`:

```json
{ "config": {} }
```

**Taker nonce state** – returns the set of `rfq_id` values this taker has already settled, used for replay protection:

```json
{ "taker_info": { "taker": "inj1taker..." } }
```

**Maker nonces** – returns the set of blind-quote nonces this maker has already had consumed, used for replay protection:

```json
{ "maker_nonces": { "maker": "inj1maker..." } }
```

---

## Contract source code

The TrueCurrent contract source is at [`InjectiveLabs/rfq`](https://github.com/InjectiveLabs/rfq) under `rfq-contract/contracts/rfq/`. It is built with CosmWasm and the Injective CosmWasm SDK.

A security audit is planned before mainnet deployment. Audit report will be linked here upon completion.
