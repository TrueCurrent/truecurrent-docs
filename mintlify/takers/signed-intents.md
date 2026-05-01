---
title: "Signed taker intents (TP/SL)"
description: "Pre-signed conditional orders on TrueCurrent. A signed taker intent authorizes a reduce-only exit in advance, gated by mark-price trigger conditions and signed with the EIP-712 v2 schema."
updatedAt: "2026-05-01"
---

A **signed taker intent** is a pre-authorized conditional order. You sign the exit parameters ahead of time, submit them to the indexer, and the relayer settles the trade when the mark-price trigger is satisfied.

This is how **take-profit / stop-loss** orders work on TrueCurrent. You can create the signed exit when opening a position or later, while the position is already open.

<Warning>
Current trigger orders are reduce-only close-position orders. Set `margin` to `"0"` and use `sign_mode: "v2"` on the wire. Do not use JSON-string signing for conditional orders.
</Warning>

---

## When to use signed intents

Use a signed intent when you cannot be online to submit at the exact trigger moment, but you can decide the trigger and exit constraints in advance.

- **Take profit:** exit long when mark price is greater than or equal to target, or exit short when mark price is less than or equal to target.
- **Stop loss:** exit long when mark price is less than or equal to stop, or exit short when mark price is greater than or equal to stop.

If you want to trade immediately while online, use [`AcceptQuote`](/takers/accepting-quotes).

---

## End-to-end flow

```mermaid
sequenceDiagram
    autonumber
    participant Taker
    participant Indexer as Indexer / Relayer
    participant MMs as Market Makers
    participant Chain as Injective Chain
    participant Contract as RFQ Contract

    Taker->>Taker: Build conditional order body
    Taker->>Taker: Sign SignedTakerIntent EIP-712 v2 digest
    Taker->>Indexer: Submit order + signature + sign_mode="v2"
    Indexer->>Indexer: Persist as untriggered
    Chain-->>Indexer: Mark-price updates
    Indexer->>Indexer: Check mark_price_gte / mark_price_lte
    Indexer->>MMs: Request exit quotes when trigger fires
    MMs-->>Indexer: Return signed v2 quotes
    Indexer->>Chain: Execute triggered settlement
    Chain->>Contract: Verify taker signature and quote signatures
    Contract->>Contract: Re-check mark-price trigger
    Contract-->>Indexer: Settlement result
```

The trigger is not latched. If the mark price moves back before the transaction lands, the contract can reject the settlement as `trigger_not_satisfied`.

---

## Conditional order body

These fields are sent to the indexer and contract. The v2 signature covers the `SignedTakerIntent` typed-data fields; `chain_id`, `contract_address`, and `unfilled_action` are wire/runtime fields.

| Field | Type | Description |
|---|---|---|
| `version` | `uint8` | Use `1`. This is the intent schema version, not the signing mode. |
| `chain_id` | `string` | Cosmos chain ID, for example `"injective-888"` on testnet. Wire/runtime field. |
| `contract_address` | `string` | RFQ contract address. Wire/runtime field; the signature binds the contract via the EIP-712 domain. |
| `taker` | `string` | Taker `inj1...` address. |
| `epoch` | `uint64` | Current taker epoch. Incremented by global intent cancellation. |
| `rfq_id` | `uint64` | Unique order ID. Use a fresh timestamp or generated ID. |
| `market_id` | `string` | Injective derivative market ID. |
| `subaccount_nonce` | `uint32` | Subaccount index, usually `0`. |
| `lane_version` | `uint64` | Current `(taker, market, subaccount)` lane version. Incremented by lane cancellation and successful settlement. |
| `deadline_ms` | `uint64` | Unix millisecond deadline. |
| `direction` | `"long" \| "short"` | Exit trade direction. |
| `quantity` | `string` | Quantity to close, as a canonical decimal string. |
| `margin` | `string` | Use `"0"` for reduce-only trigger orders. |
| `worst_price` | `string` | Worst acceptable quoted fill price. |
| `min_total_fill_quantity` | `string` | Minimum aggregate fill quantity required for settlement. |
| `trigger_type` | `string` | `"mark_price_gte"`, `"mark_price_lte"`, or `"immediate"`. |
| `trigger_price` | `string \| null` | Trigger threshold. Use `"0"` or `null` for `immediate`, depending on the helper path. |
| `unfilled_action` | `object \| null` | Wire-only and not part of the v2 digest. Use `null` unless your integration explicitly supports fallback handling. |
| `cid` | `string \| null` | Optional client identifier. Bound by the v2 signature. |
| `allowed_relayer` | `string \| null` | Optional relayer address. Bound by the v2 signature when set. |

---

## EIP-712 domain

Signed intents use the same domain separator as maker quotes:

| Field | Value |
|---|---|
| `name` | `"RFQ"` |
| `version` | `"1"` |
| `chainId` testnet | `1439` |
| `chainId` mainnet | `1776` |
| `verifyingContract` | EVM form of the RFQ contract address |

The domain `chainId` is the EVM chain ID, not the Cosmos chain ID.

---

## Sign a conditional order

Use `sign_conditional_order_v2` from `rfq-testing`. The helper signs the custom `SignedTakerIntent` typed-data digest and returns a `0x`-prefixed 65-byte signature.

```python
from rfq_test.crypto.eip712 import sign_conditional_order_v2

signature = sign_conditional_order_v2(
    private_key=TAKER_PRIVATE_KEY,
    evm_chain_id=1439,
    verifying_contract_bech32="inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk",
    version=1,
    taker=taker_address,
    epoch=epoch,
    rfq_id=rfq_id,
    market_id=market_id,
    subaccount_nonce=0,
    lane_version=lane_version,
    deadline_ms=deadline_ms,
    direction="short",
    quantity="1",
    margin="0",
    worst_price="132",
    min_total_fill_quantity="1",
    trigger_type="mark_price_gte",
    trigger_price="120",
    cid=None,
    allowed_relayer=None,
)
```

Use the same values in the order body:

```python
order_body = {
    "version": 1,
    "chain_id": "injective-888",
    "contract_address": "inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk",
    "taker": taker_address,
    "epoch": epoch,
    "rfq_id": rfq_id,
    "market_id": market_id,
    "subaccount_nonce": 0,
    "lane_version": lane_version,
    "deadline_ms": deadline_ms,
    "direction": "short",
    "quantity": "1",
    "margin": "0",
    "worst_price": "132",
    "min_total_fill_quantity": "1",
    "trigger_type": "mark_price_gte",
    "trigger_price": "120",
    "unfilled_action": None,
    "cid": None,
    "allowed_relayer": None,
}
```

---

## Submit via TakerStream

`TakerStreamClient.send_conditional_order` sends `conditional_order_sign_mode="v2"` by default.

```python
from rfq_test.clients.websocket import TakerStreamClient

async with TakerStreamClient(
    base_url=env_config.indexer.ws_endpoint,
    request_address=taker_address,
) as client:
    ack = await client.send_conditional_order(
        order_body=order_body,
        signature=signature,
        wait_for_ack=True,
        response_timeout=10.0,
    )

print(ack)
```

For REST submissions, include the same signing mode explicitly:

```python
await http.post(
    f"{indexer_http_endpoint}/v1/conditionalOrder",
    json={
        "order": order_body,
        "signature": signature,
        "sign_mode": "v2",
    },
)
```

---

## Trigger types

```json
{ "trigger_type": "immediate", "trigger_price": "0" }
{ "trigger_type": "mark_price_gte", "trigger_price": "5.20" }
{ "trigger_type": "mark_price_lte", "trigger_price": "4.80" }
```

- Use `mark_price_gte` for long take-profit and short stop-loss.
- Use `mark_price_lte` for long stop-loss and short take-profit.
- The contract re-evaluates the mark-price trigger during settlement.

---

## Epochs and lanes

Before signing, read the current `epoch` and `lane_version` for the taker. A signed intent is valid only for the exact values it signs.

- `CancelAllIntents` increments the taker's epoch and invalidates every outstanding intent for that taker.
- `CancelIntentLane` increments the lane version for one `(taker, market_id, subaccount_nonce)` lane.
- Successful settlement also advances the lane version, making trigger intents one-shot by construction.

---

## Common failures

| Symptom | Fix |
|---|---|
| Missing conditional order signing mode | Send `sign_mode: "v2"` on REST or let `send_conditional_order` set it on TakerStream. |
| Invalid signature | Check EVM `chainId`, verifying contract, exact decimal strings, trigger fields, `epoch`, and `lane_version`. |
| Trigger not satisfied | The mark price moved back before settlement. The relayer should retry when the trigger is satisfied again. |
| Stale lane or epoch | Refresh state from the contract and sign a new intent. |

---

## Next

- [Accepting quotes](/takers/accepting-quotes) covers the synchronous `AcceptQuote` path.
- [Trigger orders](/trading/trigger-orders) explains TP/SL behavior at the product level.
- [Best practices](/takers/best-practices) covers expiry races, idempotency, and `cid` usage.
