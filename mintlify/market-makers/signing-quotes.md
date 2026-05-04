---
title: "Signing quotes"
description: "Technical documentation for signing TrueCurrent RFQ quotes with the EIP-712 v2 schema used by public testnet and mainnet integrations."
updatedAt: "2026-05-01"
---

Every quote you submit must be signed with your market maker wallet. The signature is verified by the TrueCurrent contract when a taker accepts the quote, so the signed fields must match the quote payload exactly.

<Warning>
Public integrations use EIP-712 v2 quote signing only. Send `sign_mode: "v2"` with every quote and do not sign manually serialized JSON strings.
</Warning>

---

## Why quotes must be signed

Signed quotes provide two guarantees:

1. **Authenticity.** The contract can verify that the quote came from the maker address in the payload.
2. **Binding commitment.** The maker cannot change price, quantity, expiry, or taker binding after the quote is signed.

The contract verifies the signature during `AcceptQuote`. If any signed field differs from the payload accepted by the taker, settlement reverts.

---

## EIP-712 domain

The v2 quote digest uses this EIP-712 domain:

| Field | Value |
|---|---|
| `name` | `"RFQ"` |
| `version` | `"1"` |
| `chainId` testnet | `1439` |
| `chainId` mainnet | `1776` |
| `verifyingContract` | EVM form of the RFQ contract address |

The domain `chainId` is the EVM chain ID. Do not use the Cosmos chain ID (`injective-888` or `injective-1`) in the EIP-712 domain.

Current public testnet RFQ contract:

```text
inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk
```

---

## What is signed

The v2 signature covers the `SignQuote` typed-data struct:

| Field | Notes |
|---|---|
| `marketId` | Injective market ID. |
| `rfqId` | Indexer-assigned RFQ ID. Use the ID from the request or ACK. |
| `taker` | Taker `inj1...` address for taker-bound quotes, or empty for blind quotes. |
| `takerDirection` | Taker direction, `"long"` or `"short"`. |
| `takerMargin` / `takerQuantity` | Exact decimal strings from the RFQ request. |
| `maker` | Maker `inj1...` address. |
| `makerSubaccountNonce` | Usually `0`; must match the wire payload. |
| `makerQuantity` / `makerMargin` | Exact decimal strings you quote. |
| `price` | Exact decimal string you quote, quantized to market tick. |
| `expiryKind` / `expiryValue` | Timestamp milliseconds or block height. |
| `minFillQuantity` | Optional partial-fill floor; omit unless you intentionally require it. |
| `bindingKind` | Derived from whether `taker` is set. It is not a wire field. |

`chain_id` and `contract_address` may still appear in the quote payload for indexer compatibility, but v2 binds chain and contract through the EIP-712 domain above.

---

## Signing in Python

Use `sign_quote_v2` from `rfq-testing`. The helper returns a `0x`-prefixed 65-byte signature, so send it as-is.

```python
import time

from rfq_test.crypto.eip712 import sign_quote_v2

expiry_ms = int(time.time() * 1000) + 2_000

signature = sign_quote_v2(
    private_key=MM_PRIVATE_KEY,
    evm_chain_id=1439,
    verifying_contract_bech32="inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk",
    rfq_id=int(rfq_id),
    market_id=market_id,
    direction="long",
    taker=taker_address,
    taker_margin="200",
    taker_quantity="100",
    maker=maker_address,
    maker_subaccount_nonce=0,
    maker_margin="200",
    maker_quantity="100",
    price="14.85",
    expiry_ms=expiry_ms,
    min_fill_quantity=None,
)
```

Send the same values on the quote:

```python
await mm_ws.send_quote({
    "rfq_id": rfq_id,
    "market_id": market_id,
    "taker_direction": "long",
    "margin": "200",
    "quantity": "100",
    "price": "14.85",
    "expiry": expiry_ms,
    "maker": maker_address,
    "taker": taker_address,
    "signature": signature,
    "sign_mode": "v2",
    "maker_subaccount_nonce": 0,
    "chain_id": "injective-888",
    "contract_address": "inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk",
})
```

Do not prepend `0x` again, do not sign a manually serialized JSON string, and do not pass `binding_kind` into the helper or quote payload.

---

## Decimal formatting

Decimal fields are hashed as raw UTF-8 strings. `"14.85"` and `"14.850"` produce different digests.

- Quantize price to the market tick before signing. The current INJ/USDC PERP testnet tick is `0.01`.
- Send the exact same decimal strings you signed.
- Use plain notation without trailing zeros.
- Do not use locale commas, scientific notation, or integer-scaled token units.

```python
from decimal import Decimal, ROUND_DOWN

TICK = Decimal("0.01")
price = format(Decimal(str(raw_price)).quantize(TICK, rounding=ROUND_DOWN).normalize(), "f")
```

---

## Common signing errors

| Error | Fix |
|---|---|
| Missing or empty `sign_mode` | Send `"sign_mode": "v2"` on every quote. |
| `invalid_signature` | Check EVM chain ID, RFQ contract address, maker address, compact `v=0/1`, and exact signed decimal strings. |
| Settlement reverts after quote ACK | Confirm the taker is accepting the same signed price, quantity, expiry, maker, taker, nonce, and signature. |
| Price rejected as non-canonical | Send `3.4`, not `3.40`, after quantizing to tick size. |

---

## Security considerations

Your private key signs binding price commitments. Store it like any production signing key:

- Never commit private keys to source control.
- Use environment variables or a secrets manager.
- Separate signing keys from wallets that hold large balances when possible.
- Monitor failed signature and quote-expiry errors; they usually indicate a serialization mismatch or stale clock.
