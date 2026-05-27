---
title: "Sending quotes"
description: "Wire format and lifecycle semantics for sending signed RFQ quotes over MakerStream, including ACKs, quote updates, and settlement updates."
updatedAt: "2026-05-27"
---

After signing, send the quote over MakerStream as a `MakerStreamStreamingRequest` with `message_type = "quote"`.

Any field covered by the EIP-712 v2 signature must match exactly, or settlement will reject the quote even if the indexer accepts the payload.

---

## Quote payload

```json
{
  "message_type": "quote",
  "quote": {
    "chain_id": "injective-888",
    "contract_address": "inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk",
    "rfq_id": 1770848375348,
    "market_id": "0xdc70164d7120529c3cd84278c98df4151210c0447a65a2aab03459cf328de41e",
    "taker_direction": "long",
    "margin": "100",
    "quantity": "10",
    "price": "14.85",
    "expiry": 1770848395000,
    "maker": "inj1maker...",
    "maker_subaccount_nonce": 0,
    "taker": "inj1taker...",
    "signature": "0x...",
    "sign_mode": "v2",
    "evm_chain_id": 1439,
    "min_fill_quantity": "0"
  }
}
```

---

## Field reference

| Field | Type | Value |
| --- | --- | --- |
| `message_type` | string | `"quote"` |
| `quote.chain_id` | string | Cosmos chain ID for indexer compatibility, for example `"injective-888"` on testnet |
| `quote.contract_address` | string | RFQ contract bech32 address |
| `quote.rfq_id` | uint64 | Request `rfq_id` |
| `quote.market_id` | string | Request market ID |
| `quote.taker_direction` | string | Request direction, `"long"` or `"short"` |
| `quote.margin` | string | Maker margin string, same value signed as `makerMargin` |
| `quote.quantity` | string | Maker quantity string, same value signed as `makerQuantity` |
| `quote.price` | string | Tick-quantized price string, exactly as signed |
| `quote.expiry` | uint64 | Unix ms expiry for live quotes |
| `quote.maker` | string | Maker `inj1...` address |
| `quote.maker_subaccount_nonce` | uint32 | Maker subaccount nonce, same value signed |
| `quote.taker` | string | Taker `inj1...` address from the request |
| `quote.signature` | string | `0x`-prefixed 65-byte signature |
| `quote.sign_mode` | string | Required. Use `"v2"`. |
| `quote.evm_chain_id` | uint64 | Required for v2. Same EVM chain ID used in the EIP-712 domain (`1439` testnet, `1776` mainnet). |
| `quote.min_fill_quantity` | string | Optional in some helper paths. If present, match the signed value. Use `"0"` when absent. |
| `quote.nonce` | uint64 | Blind quotes only. Do not send for live taker-bound quotes unless the helper path requires it. |

The v2 signature binds chain and contract through the EIP-712 domain (`evm_chain_id` and `verifying_contract_bech32`). `chain_id` and `contract_address` are still sent for indexer compatibility.

---

## Expiry

Live quotes should be short-lived. A typical live expiry is:

```python
expiry = int(time.time() * 1000) + 2_000
```

The contract checks expiry at settlement block time. Taker quote collection and transaction inclusion both consume the expiry window, so do not set live expiries too tight and do not do slow work after receiving an RFQ.

Blind or TP/SL liquidity can use longer expiries, but those paths need stricter nonce, inventory, and cancel handling.

---

## Quote ACK

After `send_quote`, the indexer may return:

```json
{
  "message_type": "quote_ack",
  "quote_ack": {
    "rfq_id": 1770848375348,
    "status": "success"
  }
}
```

or an error:

```json
{
  "message_type": "error",
  "error": {
    "code": "invalid_signature",
    "message": "..."
  }
}
```

`quote_ack.status="success"` means the indexer accepted and routed the quote. It does not mean the taker accepted it and it does not mean settlement succeeded.

---

## Fill and settlement state

Subscribe to quote and settlement updates when your client supports them.

Track these separately:

| Event | Meaning |
| --- | --- |
| `quote_ack` | Indexer accepted the quote payload |
| `quote_update` | Quote lifecycle update, when emitted |
| `settlement_update` | Taker or relayer attempted settlement |
| Chain transaction / position state | Source of truth for actual fills |

If no quote or settlement update arrives before your quote expiry, treat the quote as not accepted.

After a settlement attempt, reconcile:

- `rfq_id`
- quoted price
- executed quantity
- executed margin
- taker and maker addresses
- maker subaccount nonce
- transaction hash
- any skipped-quote reason

---

## Common mistakes

| Symptom | Likely cause |
| --- | --- |
| `sign_mode required` | Quote omitted `sign_mode: "v2"` |
| Wrong chain in signature | EVM chain ID placed in `chain_id` instead of `evm_chain_id`, or Cosmos chain ID used in the EIP-712 domain |
| Signature verifies locally but fails remotely | Decimal string changed between signing and sending |
| Quote ACK succeeds but no fill happens | Taker did not accept before expiry, another quote won, or settlement skipped this quote |
| Settlement skips maker quote | Expired quote, mark-band rejection, `worst_price` failure, insufficient maker margin, or `min_fill_quantity` mismatch |

Next: [Testnet runbook](/sdk-trading/runbook).
