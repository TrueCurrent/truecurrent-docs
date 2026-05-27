---
title: "Smart contract"
description: "Reference documentation for TrueCurrent's RFQ CosmWasm contract, including AcceptQuote, AcceptSignedIntent, authz settlement, validation, and maker queries."
updatedAt: "2026-05-27"
---

TrueCurrent's RFQ contract is the onchain enforcement point for maker quotes and signed taker intents. The indexer can route messages, but the contract verifies signatures, authz, expiries, price constraints, and settlement permissions before positions change.

---

## Contract address

| Network | Address |
| --- | --- |
| Injective Testnet | `inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk` |
| Injective Mainnet | Contact TrueCurrent for the current production contract |

Testnet EIP-712 chain ID is `1439`. The Cosmos chain ID is `injective-888`.

---

## Entrypoints

| Entrypoint | Caller | Use case |
| --- | --- | --- |
| `AcceptQuote` | Taker | Taker is online, has collected one or more maker quotes, and submits settlement now. |
| `AcceptSignedIntent` | Relayer | Taker pre-signed a conditional reduce-only exit; relayer submits once the trigger condition is satisfied. |
| `CancelIntentLane` | Taker | Invalidates active signed intents for one `(market_id, subaccount_nonce)` lane. |
| `CancelAllIntents` | Taker | Invalidates all active signed intents for that taker. |

`AcceptSignedIntent` validates the taker intent, pairs it with maker liquidity for the same `rfq_id`, and then runs the same settlement machinery as `AcceptQuote`.

---

## `AcceptQuote`

Example execute shape:

```json
{
  "accept_quote": {
    "quotes": [
      {
        "maker": "inj1maker...",
        "maker_subaccount_nonce": 0,
        "margin": "200",
        "quantity": "100",
        "price": "4.45",
        "expiry": 1770848377348,
        "signature": "0x...",
        "sign_mode": "v2",
        "evm_chain_id": 1439
      }
    ],
    "rfq_id": "1770848375348",
    "market_id": "0xdc70164d7120529c3cd84278c98df4151210c0447a65a2aab03459cf328de41e",
    "direction": "long",
    "margin": "200",
    "quantity": "100",
    "worst_price": "4.70",
    "unfilled_action": null
  }
}
```

| Field | Description |
| --- | --- |
| `quotes` | One or more maker quotes selected by the taker |
| `quotes[].maker` | Maker Injective address |
| `quotes[].maker_subaccount_nonce` | Maker subaccount nonce registered on the RFQ contract |
| `quotes[].margin` | Maker margin commitment |
| `quotes[].quantity` | Quantity the maker is filling |
| `quotes[].price` | Maker quoted price |
| `quotes[].expiry` | Quote expiry in Unix milliseconds |
| `quotes[].signature` | EIP-712 v2 maker signature |
| `quotes[].sign_mode` | Must be `v2` for current integrations |
| `quotes[].evm_chain_id` | EIP-712 chain ID (`1439` testnet, `1776` mainnet) |
| `rfq_id` | Indexer-assigned RFQ identifier, serialized as the contract expects |
| `market_id` | Injective derivative market ID |
| `direction` | Taker direction, `long` or `short` |
| `margin` | Taker margin |
| `quantity` | Taker requested quantity |
| `worst_price` | Taker price limit |
| `unfilled_action` | Reserved product behavior; pass `null` unless a supported SDK helper sets it |

---

## Validation model

The contract validates each submitted quote against the taker's request. Quote-level failures are skipped where the settlement path permits; settlement succeeds only if the remaining valid quotes satisfy the taker's fill constraints.

| Validation | Failure result |
| --- | --- |
| Maker EIP-712 signature matches exact fields | Quote skipped |
| Quote `sign_mode` and `evm_chain_id` match v2 domain | Quote skipped |
| Quote `rfq_id` matches settlement or signed intent | Quote skipped or signed-intent error |
| Quote has not expired | Quote skipped |
| Maker is whitelisted and subaccount nonce matches registration | Quote skipped or no fill |
| Maker and taker authz grants exist | Transaction fails with authorization error |
| Maker and taker margin is sufficient | Quote skipped or transaction fails |
| Quote satisfies taker `worst_price` | Quote skipped |
| Trigger and deadline are valid for signed intents | `AcceptSignedIntent` fails |

This means a multi-quote settlement can still fill if some quotes fail and enough valid quotes remain.

---

## Settlement through authz

The RFQ contract settles by using pre-granted Injective `authz` permissions.

| Role | Required grants |
| --- | --- |
| Maker | `/injective.exchange.v2.MsgPrivilegedExecuteContract`, `/cosmos.bank.v1beta1.MsgSend` |
| Taker | `/injective.exchange.v2.MsgPrivilegedExecuteContract`, `/injective.exchange.v2.MsgBatchUpdateOrders`, `/cosmos.bank.v1beta1.MsgSend` |

On successful settlement, both parties' derivative positions and margin transfers are updated atomically through the Injective exchange module. There is no partial state where only one side receives the position.

---

## `AcceptSignedIntent`

Signed intents support TP/SL exits:

1. Taker signs a `SignedTakerIntent` with EIP-712 v2.
2. Taker submits it through TakerStream with v2 conditional-order signing fields.
3. Relayer monitors the trigger condition.
4. Relayer pairs the intent with maker liquidity bound to the same `rfq_id`.
5. Contract verifies the taker signature, trigger, deadline, cancellation counters, maker quote, and authz before settlement.

The contract re-checks trigger state at execution time. If mark price moves back before the transaction lands, settlement can fail with `trigger_not_satisfied`; the relayer should retry while the intent remains valid.

---

## Queries

List approved makers:

```json
{ "list_makers": {} }
```

Check one maker:

```json
{ "is_maker": { "address": "inj1..." } }
```

`list_makers` is paginated. A first-page query can miss a registered address; page through results or use the reference client's `ContractClient.is_maker_registered()` helper.

---

## Related pages

- [Protocol reference](/sdk-trading/protocol-reference)
- [Authorization setup](/sdk-trading/authz)
- [Signed intents](/sdk-trading/signed-intents)
- [Error reference](/technical/error-reference)

The TrueCurrent contract source code is published in the [InjectiveLabs RFQ repository](https://github.com/InjectiveLabs/rfq).
