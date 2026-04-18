---
title: "Receiving RFQ requests"
description: "Structure and field reference for RFQ request messages received by market makers over MakerStream, including field types, meanings, and how to interpret taker intent."
updatedAt: "2026-04-18"
---

```json
{
  "message_type": "request",
  "request": {
    "rfq_id": 1770848375348,
    "market_id": "0x17ef48032cb24375ba7c2e39f384e56433bcab20cbee9a7357e4cba2eb00abe6",
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
| `rfq_id` | uint64 | Unique request identifier (timestamp-based nonce). |
| `market_id` | string | Injective derivative market ID (hex, `0x…`). |
| `direction` | string | Taker's direction: `"long"` or `"short"`. |
| `margin` | string | Taker's margin (decimal string, USDT on testnet). |
| `quantity` | string | Taker's quantity (decimal string). |
| `worst_price` | string | Max price the taker accepts (long) or min (short). |
| `request_address` | string | Taker's `inj1...` address. |
| `expiry` | uint64 | Request expiration (Unix ms). |

Your job: price, build a quote, sign it, send it.
