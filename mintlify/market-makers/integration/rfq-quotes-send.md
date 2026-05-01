---
title: "Sending quotes"
description: "Wire format and field reference for sending signed quotes over MakerStream, including quote ACK and error response handling."
updatedAt: "2026-05-01"
---

Wrap the quote in a `MakerStreamStreamingRequest` with `message_type = "quote"`. Any field covered by the EIP-712 v2 signature must match exactly, or settlement rejects it.

| Field | Type | Value |
|---|---|---|
| `message_type` | string | `"quote"` |
| `quote.chain_id` | string | Cosmos chain ID for indexer compatibility, for example `"injective-888"` on testnet |
| `quote.contract_address` | string | RFQ contract address for indexer compatibility |
| `quote.rfq_id` | uint64 | From request |
| `quote.market_id` | string | From request |
| `quote.taker_direction` | string | `"long"` / `"short"` |
| `quote.margin` | string | Your margin |
| `quote.quantity` | string | Your quantity |
| `quote.price` | string | Your price (tick-quantized, same string you signed) |
| `quote.expiry` | uint64 | Unix ms (recommend `now + 20s` for live quotes) |
| `quote.maker` | string | Your `inj1...` |
| `quote.maker_subaccount_nonce` | uint32 | Subaccount index (default `0`). Must match the value you signed. |
| `quote.taker` | string | Taker's `inj1...` (from request) |
| `quote.signature` | string | Hex, `0x`-prefixed |
| `quote.sign_mode` | string | Required. Use `"v2"`. |
| `quote.min_fill_quantity` | string | Optional. Must match the value you signed. Omit if not declaring a minimum. |

The v2 signature binds chain and contract through the EIP-712 domain (`evm_chain_id` and `verifying_contract_bech32`). The `chain_id` and `contract_address` fields above are still sent for indexer compatibility.

### Quote ACK

- Success: `{"message_type": "quote_ack", "quote_ack": {"rfq_id": 123, "status": "success"}}`
- Error: `{"message_type": "error", "error": {"code": "...", "message": "..."}}`
