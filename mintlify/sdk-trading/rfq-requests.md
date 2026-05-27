---
title: "RFQ requests"
description: "Field reference for RFQ request messages received by makers over MakerStream and how to interpret taker intent."
updatedAt: "2026-05-27"
---

MakerStream `request` messages contain everything a maker needs to price, size, sign, and return a quote.

```json
{
  "message_type": "request",
  "request": {
    "client_id": "7d7f8c28-1df6-40d3-bf29-0f9d2c2f3d10",
    "rfq_id": 1770848375348,
    "market_id": "0xdc70164d7120529c3cd84278c98df4151210c0447a65a2aab03459cf328de41e",
    "direction": "long",
    "margin": "100",
    "quantity": "10",
    "worst_price": "100",
    "request_address": "inj1abc...xyz",
    "expiry": 1770848675348
  }
}
```

---

## Request fields

| Field | Type | Meaning |
| --- | --- | --- |
| `client_id` | string | Taker-supplied UUID correlation ID. The indexer enforces UUID format. |
| `rfq_id` | uint64 | Indexer-assigned RFQ ID. Use this for quote signing and settlement correlation. |
| `market_id` | string | Injective derivative market ID. |
| `direction` | string | Taker direction: `"long"` or `"short"`. |
| `margin` | string | Taker's requested margin, human-scale decimal string. |
| `quantity` | string | Taker's requested quantity, human-scale decimal string. |
| `worst_price` | string | Max acceptable price for taker long, min acceptable price for taker short. |
| `request_address` | string | Taker `inj1...` address. |
| `expiry` | uint64 | Request expiry as Unix milliseconds. |

`rfq_id` is the settlement correlation key. Do not invent it locally and do not use `client_id` in its place.

---

## How to interpret direction

The direction is always from the taker's perspective.

| Taker direction | Taker wants | Maker fills |
| --- | --- | --- |
| `long` | Buy / go long | Short side |
| `short` | Sell / go short | Long side |

For a taker long, lower maker quote prices are more competitive. For a taker short, higher maker quote prices are more competitive.

---

## Pricing inputs

A production maker usually prices from:

- Current mark price and external reference venues
- Market tick size and quantity step
- Request `quantity` and `margin`
- Taker `worst_price`
- Inventory skew
- Funding carry
- Hedge cost
- Available USDC margin
- Quote expiry budget

`worst_price` is already human-scale, for example `"14.85"`. Do not treat it as a 1e18-scaled integer.

---

## Partial fills

Your quote does not have to match the taker's full requested margin or quantity. You may quote a smaller `maker_quantity` / `maker_margin` if you only want to fill part of the request.

If you set `min_fill_quantity`, the contract will not fill that quote below your minimum. Sign absent `min_fill_quantity` as `"0"` through the helper path; never use an empty string in the signed digest.

---

## Next step

After receiving a request:

1. Decide whether to quote.
2. Canonicalize all decimal strings.
3. Sign the quote with EIP-712 v2.
4. Send it with `sign_mode: "v2"` and `evm_chain_id`.

See [Building & signing quotes](/sdk-trading/signing-quotes) and [Sending quotes](/sdk-trading/sending-quotes).
