---
title: "Receiving RFQ requests"
description: "Structure and field reference for RFQ request messages received by makers over MakerStream, including field types, meanings, and how to interpret taker intent."
updatedAt: "2026-05-05"
---

```json
{
  "message_type": "request",
  "request": {
    "rfq_id": 1770848375348,
    "client_id": "7d7f8c28-1df6-40d3-bf29-0f9d2c2f3d10",
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

### Request fields

| Field | Type | Description |
|---|---|---|
| `rfq_id` | uint64 | Indexer-assigned unique request identifier. |
| `client_id` | string (UUID) | Taker-supplied UUID v4 correlation ID. The indexer enforces UUID format. Numeric timestamps are rejected with `message.client_id must be formatted as a uuid`. |
| `market_id` | string | Injective derivative market ID (hex, `0x…`). |
| `direction` | string | Taker's direction: `"long"` or `"short"`. |
| `margin` | string | Taker's margin (decimal string, USDC on the current testnet market). |
| `quantity` | string | Taker's quantity (decimal string). |
| `worst_price` | string | Max price the taker accepts (long) or min (short). Value is in human-readable scale (e.g. `"14.85"`), not 1e18-scaled. |
| `request_address` | string | Taker's `inj1...` address. |
| `expiry` | uint64 | Request expiration (Unix ms). |

Your job: price, build a quote, sign it, send it.
