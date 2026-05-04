---
title: "Building & signing quotes"
description: "Complete guide to constructing and signing RFQ quotes with the EIP-712 v2 quote scheme used by the live testnet contract."
updatedAt: "2026-05-01"
---

Every quote you return is an EIP-712 typed-data `secp256k1` signature over the `SignQuote` struct. The contract verifies it at settlement; any field, domain, decimal string, or `v` byte mismatch fails recovery and the trade reverts.

<Warning>
This guide is EIP-712 v2 only. The indexer requires `sign_mode: "v2"` on every quote. Do not sign manually serialized JSON strings for public testnet or mainnet integrations.
</Warning>

---

## 1. Decide your price

Taker going **long** → you fill the short side. Taker going **short** → you fill the long side. Your quoted `margin` / `quantity` may differ from the taker's request for partial fills.

Quantize your price to the market tick before signing. For the current INJ/USDC PERP testnet market, the tick is `0.01`.

---

## 2. EIP-712 domain

| Field | Value |
|---|---|
| `name` | `"RFQ"` |
| `version` | `"1"` |
| `chainId` · testnet | `1439` |
| `chainId` · mainnet | `1776` |
| `verifyingContract` | `bech32_to_evm(<RFQ contract>)` |

Current testnet RFQ contract:

```text
inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk
```

The `chainId` here is the EVM chain ID, not the Cosmos chain ID. Use `1439` on testnet and `1776` on mainnet; do not use `injective-888` or `injective-1` in the EIP-712 domain.

---

## 3. Quote shape on the wire

The indexer quote payload must include `sign_mode: "v2"` and the signature returned by the v2 helper.

```json
{
  "sign_mode": "v2",
  "rfq_id": 1770848375348,
  "market_id": "0xdc70164d7120529c3cd84278c98df4151210c0447a65a2aab03459cf328de41e",
  "taker_direction": "long",
  "margin": "200",
  "quantity": "100",
  "price": "14.85",
  "expiry": 1747812432345,
  "maker": "inj1...",
  "maker_subaccount_nonce": 0,
  "taker": "inj1...",
  "signature": "0x<r><s><v>",
  "chain_id": "injective-888",
  "contract_address": "inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk"
}
```

`chain_id` and `contract_address` are wire fields used by the indexer for compatibility. The v2 signature binds chain and contract through the EIP-712 domain, not through these payload fields.

`bindingKind` is derived inside the v2 digest: `1` when `taker` is set and `0` for blind quotes. It is not an `RFQQuoteType` wire field, and you should not pass `binding_kind` or `nonce` into the helper for live taker-bound quotes.

---

## 4. Sign with the Python helper

Use `sign_quote_v2` from `rfq-testing`. It builds the EIP-712 type hash, applies the domain separator, and produces a 65-byte signature with compact y-parity (`v=0/1`). Do not use `eth_sign`, `personal_sign`, or generic wallet typed-data helpers unless you have reproduced this exact digest.

```python
from rfq_test.crypto.eip712 import sign_quote_v2

sig = sign_quote_v2(
    private_key=MM_PRIVATE_KEY,
    rfq_id=rfq_id,
    market_id=MARKET.id,
    direction="long",
    taker=retail_addr,
    taker_margin="200",
    taker_quantity="100",
    maker=mm_addr,
    maker_subaccount_nonce=0,
    maker_margin="200",
    maker_quantity="100",
    price="14.85",
    expiry_ms=int(time.time() * 1000) + 2_000,
    min_fill_quantity=None,
    evm_chain_id=1439,
    verifying_contract_bech32="inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk",
)
```

Then send the quote:

```python
ack = await mm_ws.send_quote({
    "sign_mode": "v2",
    "rfq_id": rfq_id,
    "market_id": MARKET.id,
    "taker_direction": "long",
    "margin": "200",
    "quantity": "100",
    "price": "14.85",
    "expiry": expiry,
    "maker": mm_addr,
    "maker_subaccount_nonce": 0,
    "taker": retail_addr,
    "signature": sig,  # already 0x-prefixed
    "chain_id": "injective-888",
    "contract_address": "inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk",
}, wait_for_response=True, response_timeout=5.0)
```

---

## Decimal hygiene

Decimal fields are hashed as the raw UTF-8 string. `"14.85"` and `"14.850"` produce different digests.

- Quantize prices to the market tick before signing.
- Send the exact same price string you signed.
- Use canonical plain notation without trailing zeros.
- Never use locale commas, scientific notation, or 1e18-scaled integers.

For example:

```python
from decimal import Decimal, ROUND_DOWN

TICK = Decimal("0.01")
snap = lambda x: format(Decimal(str(x)).quantize(TICK, rounding=ROUND_DOWN).normalize(), "f")
```

---

## Common failures

| Symptom | Fix |
|---|---|
| `sign_mode required` / empty signing mode | Include `"sign_mode": "v2"` on every quote payload. |
| `invalid_signature` | Check EVM `chainId` (`1439` testnet), EVM-form `verifyingContract`, compact `v=0/1`, and exact decimal strings. |
| Price rejected as non-canonical | Strip trailing zeros after quantizing; send `3.4`, not `3.40`. |
| Quote accepted by indexer but settlement fails | Make sure the quote has not expired and the taker submits the same signed price, quantity, expiry, maker, and signature. |
