---
title: "Sending quotes"
description: "Wire format and field reference for sending signed quotes over MakerStream, including quote ACK and error response handling."
updatedAt: "2026-05-05"
---

Wrap the quote in a `MakerStreamStreamingRequest` with `message_type = "quote"`.
Any field covered by the EIP-712 v2 signature must match exactly, or settlement rejects it.

<Warning>
`quote.chain_id` / `quote.chainId` is the Cosmos chain ID, not the EIP-712 numeric chain ID. Use `"injective-888"` on testnet or `"injective-1"` on mainnet. Put `1439` / `1776` only in `quote.evm_chain_id` / `quote.evmChainId` and the EIP-712 domain.
</Warning>

| Field | Type | Value |
|---|---|---|
| `message_type` | string | `"quote"` |
| `quote.chain_id` | string | Cosmos chain ID for indexer compatibility, for example `"injective-888"` on testnet. Never send `1439` or `1776` here. |
| `quote.contract_address` | string | RFQ contract address for indexer compatibility |
| `quote.rfq_id` | uint64 | From request |
| `quote.market_id` | string | From request |
| `quote.taker_direction` | string | `"long"` / `"short"` |
| `quote.margin` | string | Your margin |
| `quote.quantity` | string | Your quantity |
| `quote.price` | string | Your price (tick-quantized, same string you signed) |
| `quote.expiry` | uint64 | Unix ms. Must be at least `now + 1500ms`; `now + 2000ms` or longer improves match odds but increases stale-price exposure. |
| `quote.maker` | string | Your `inj1...` |
| `quote.maker_subaccount_nonce` | uint32 | Subaccount index (default `0`). Must match the value you signed. |
| `quote.taker` | string | Taker's `inj1...` (from request) |
| `quote.signature` | string | Hex, `0x`-prefixed |
| `quote.sign_mode` | string | Required. Use `"v2"`. |
| `quote.evm_chain_id` | uint64 | `sign_mode="v2"` is enforced. EVM chain ID matching the EIP-712 domain (`1439` for testnet, `1776` for mainnet). |
| `quote.min_fill_quantity` | string | Optional. Must match the value you signed. Use `"0"` when absent — never the empty string. |

The v2 signature binds chain and contract through the EIP-712 domain (`evm_chain_id` and `verifying_contract_bech32`). The `chain_id` and `contract_address` fields above are still sent for indexer compatibility.

### Quote ACK

- Success: `{"message_type": "quote_ack", "quote_ack": {"rfq_id": 123, "status": "success"}}`
- Error: `{"message_type": "error", "error": {"code": "...", "message": "..."}}`
