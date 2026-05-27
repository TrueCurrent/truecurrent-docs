---
title: "Building & signing quotes"
description: "Complete guide to constructing and signing TrueCurrent RFQ quotes with the EIP-712 v2 SignQuote schema."
updatedAt: "2026-05-27"
---

Every maker quote is an EIP-712 v2 `secp256k1` signature over the `SignQuote` typed-data payload. The contract verifies that signature during settlement.

<Warning>
This guide is EIP-712 v2 only. Quote payloads must include `sign_mode: "v2"` and `evm_chain_id`. Do not sign JSON strings, `eth_sign` messages, or generic wallet typed-data payloads unless you have reproduced the exact digest used by the protocol.
</Warning>

<Warning>
There are two different chain IDs in every v2 quote. EIP-712 uses a field named `chainId`, but that value is the numeric EVM chain ID: `1439` on testnet or `1776` on mainnet. The quote wire field `chain_id` / `chainId` is the Cosmos chain ID: `injective-888` on testnet or `injective-1` on mainnet. Passing `1439` or `1776` as `quote.chain_id` is invalid.
</Warning>

---

## 1. Decide whether to quote

For each request, decide:

- whether your maker is willing to fill it,
- how much quantity and margin to provide,
- what price to quote,
- how long the quote remains valid,
- and which maker subaccount nonce backs the quote.

Direction is from the taker's perspective:

| Taker direction | Maker exposure |
| --- | --- |
| `long` | Maker fills the short side |
| `short` | Maker fills the long side |

Your quoted `margin` and `quantity` may be smaller than the taker's request for partial fills.

---

## 2. Canonicalize decimals

Decimal fields are signed as exact UTF-8 strings. These are different signed messages:

- `"14.85"`
- `"14.850"`
- `"14.8500"`

Apply one canonicalization helper to every decimal field before signing and sending: price, taker margin, taker quantity, maker margin, maker quantity, minimum fill quantity, trigger price, and worst price.

```python
from decimal import Decimal, ROUND_DOWN

def to_canonical(x, tick) -> str:
    """Quantize x to the market tick and emit a plain decimal string."""
    return format(
        Decimal(str(x)).quantize(
            Decimal(str(tick)),
            rounding=ROUND_DOWN,
        ).normalize(),
        "f",
    )

# "4.50"    -> "4.5"
# "76462.0" -> "76462"   # integer-tick market
# "110.00"  -> "110"
```

Never send locale commas, scientific notation, 1e18-scaled integers, or values with trailing zeros after quantization.

---

## 3. EIP-712 domain

Quotes use this EIP-712 domain:

| Field | Value |
| --- | --- |
| `name` | `"RFQ"` |
| `version` | `"1"` |
| `chainId` testnet | `1439` |
| `chainId` mainnet | `1776` |
| `verifyingContract` | EVM address derived from the RFQ contract bech32 address |

The domain `chainId` is the EVM chain ID. It is not the Cosmos chain ID. This is the main naming collision in the RFQ payloads: `chainId` in the EIP-712 domain is not the same field as `quote.chain_id` / `quote.chainId` on the wire.

| Context | Testnet | Mainnet |
| --- | --- | --- |
| EIP-712 domain `chainId` | `1439` | `1776` |
| Quote `evm_chain_id` / `evmChainId` | `1439` | `1776` |
| Quote `chain_id` / `chainId` | `injective-888` | `injective-1` |

Current testnet RFQ contract:

```text
inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk
```

---

## 4. `SignQuote` typed-data layout

The `SignQuote` message has 16 fields. Field order matters.

| # | Field | Type | Encoding |
| --- | --- | --- | --- |
| 1 | `evmChainId` | `uint64` | Same EVM chain ID used in the domain and quote `evm_chain_id` |
| 2 | `marketId` | `string` | `keccak256(utf8(s))` |
| 3 | `rfqId` | `uint64` | Big-endian integer word |
| 4 | `taker` | `address` | 20-byte EVM address from taker `inj1...` |
| 5 | `takerDirection` | `uint8` | `0` for long, `1` for short |
| 6 | `takerMargin` | `string` | `keccak256(utf8(s))` |
| 7 | `takerQuantity` | `string` | `keccak256(utf8(s))` |
| 8 | `maker` | `address` | 20-byte EVM address from maker `inj1...` |
| 9 | `makerSubaccountNonce` | `uint32` | Big-endian integer word |
| 10 | `makerQuantity` | `string` | `keccak256(utf8(s))` |
| 11 | `makerMargin` | `string` | `keccak256(utf8(s))` |
| 12 | `price` | `string` | `keccak256(utf8(s))`; must be tick-quantized |
| 13 | `expiryKind` | `uint8` | `0` for timestamp ms, `1` for block height |
| 14 | `expiryValue` | `uint64` | Timestamp ms or block height |
| 15 | `minFillQuantity` | `string` | Use `"0"` when absent; never empty string |
| 16 | `bindingKind` | `uint8` | `1` for current taker-bound RFQ quotes |

The final digest is:

```text
keccak256(0x19 || 0x01 || domainSeparator || keccak256(typeHash || encoded SignQuote fields))
```

The helper derives `bindingKind` from whether `taker` is present. For current maker integrations, pass the taker address from the RFQ request.

---

## 5. Sign with the helper

Use `sign_quote_v2` from `injective-rfq-toolkit`.

```python
import time
from rfq_test.crypto.eip712 import sign_quote_v2

expiry_ms = int(time.time() * 1000) + 2_000  # must be at least now + 1500ms

sig = sign_quote_v2(
    private_key=MM_PRIVATE_KEY,
    evm_chain_id=1439,
    verifying_contract_bech32="inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk",
    market_id=MARKET.id,
    rfq_id=int(rfq_id),
    taker=taker_addr,
    direction="long",
    taker_margin="100",
    taker_quantity="10",
    maker=mm_addr,
    maker_subaccount_nonce=0,
    maker_margin="100",
    maker_quantity="10",
    price="14.85",
    expiry_ms=expiry_ms,
    min_fill_quantity=None,
)
```

The helper returns an `0x`-prefixed 65-byte signature (`r || s || v`) with compact y-parity `v=0/1`.

---

## 6. Send the same values

Send the exact strings and numbers you signed.

```python
ack = await mm_ws.send_quote({
    "chain_id": "injective-888",
    "contract_address": "inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk",
    "rfq_id": int(rfq_id),
    "market_id": MARKET.id,
    "taker_direction": "long",
    "margin": "100",
    "quantity": "10",
    "price": "14.85",
    "expiry": expiry_ms,
    "maker": mm_addr,
    "maker_subaccount_nonce": 0,
    "taker": taker_addr,
    "signature": sig,
    "sign_mode": "v2",
    "evm_chain_id": 1439,
}, wait_for_response=True, response_timeout=5.0)
```

`chain_id` is still the Cosmos chain ID. `evm_chain_id` is the EVM chain ID embedded in the EIP-712 domain. Do not send `1439` or `1776` in `chain_id`.

---

## Common failures

| Symptom | Fix |
| --- | --- |
| `sign_mode required` or empty signing mode | Include `"sign_mode": "v2"` on every quote payload |
| Wrong chain ID | Use `1439` / `1776` in EIP-712 domain `chainId` and quote `evm_chain_id`; use `injective-888` / `injective-1` in quote `chain_id` |
| `invalid_signature` | Check field order, EVM-form verifying contract, compact `v=0/1`, exact decimal strings, and `maker_subaccount_nonce` |
| Non-canonical price | Strip trailing zeros after quantizing; send `"76462"` for integer-tick markets, not `"76462.0"` |
| Digest mismatch on minimum fill | Use `None` in the helper or `"0"` in the signed field; never `""` |
| Quote accepted by indexer but skipped onchain | Check expiry, `worst_price`, mark-band validation, maker registration, and maker margin |

Next: [Sending quotes](/sdk-trading/sending-quotes).
