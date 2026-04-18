---
title: "Sending quotes"
description: "Wire format and field reference for sending signed quotes over MakerStream, including quote ACK and error response handling."
updatedAt: "2026-04-18"
---

Wrap the signed payload in a `MakerStreamStreamingRequest` with `message_type = "quote"`. The quote fields mirror the signed payload — any field covered by the signature must match exactly, or the contract rejects it at settlement.

| Field | Type | Value |
|---|---|---|
| `message_type` | string | `"quote"` |
| `quote.chain_id` | string | Chain ID |
| `quote.contract_address` | string | RFQ contract |
| `quote.rfq_id` | uint64 | From request |
| `quote.market_id` | string | From request |
| `quote.taker_direction` | string | `"long"` / `"short"` |
| `quote.margin` | string | Your margin |
| `quote.quantity` | string | Your quantity |
| `quote.price` | string | Your price (tick-quantized, same string you signed) |
| `quote.expiry` | uint64 | Unix ms (recommend `now + 20s` for live quotes) |
| `quote.maker` | string | Your `inj1...` |
| `quote.maker_subaccount_nonce` | uint32 | Subaccount index (default `0`). **Must match the `ms` value you signed.** |
| `quote.taker` | string | Taker's `inj1...` (from request) |
| `quote.signature` | string | Hex, `0x`-prefixed |
| `quote.min_fill_quantity` | string | Optional. **Must match `mfq` in your signed payload.** Omit if not declaring a minimum. |
| `quote.nonce` | uint64 | Only for blind quotes ([§ Blind quotes](/market-makers/integration/rfq-quotes-blind)). Omit for taker-bound live quotes. |

### Quote ACK

- Success: `{"message_type": "quote_ack", "quote_ack": {"rfq_id": 123, "status": "success"}}`
- Error: `{"message_type": "error", "error": {"code": "...", "message": "..."}}`
