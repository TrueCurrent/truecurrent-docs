---
title: "Building & signing quotes"
description: "Complete guide to constructing and cryptographically signing RFQ quotes: signing payload field reference, critical gotchas, and reference implementations in Python and TypeScript."
updatedAt: "2026-04-18"
---

### 1. Decide your price

Taker going **long** → you fill the short side (sell higher). Taker going **short** → you fill the long side (buy lower). Your quoted `margin` / `quantity` may differ from the taker's for partial fills.

### 2. Build the signing payload

Canonical JSON with abbreviated field names. **Order matters** — the contract reconstructs this from the settlement tx and recovers your signer address; any mismatch fails verification.

```json
{
  "c":  "injective-888",
  "ca": "inj1t8hyyle68vd0kzsdehxg0sywttrwmt58jzk29q",
  "mi": "0x17ef48032cb24375ba7c2e39f384e56433bcab20cbee9a7357e4cba2eb00abe6",
  "id": 1770848375348,
  "t":  "inj1abc...taker",
  "td": "long",
  "tm": "100",
  "tq": "10",
  "m":  "inj1def...maker",
  "ms": 0,
  "mq": "10",
  "mm": "100",
  "p":  "4.5",
  "e":  { "ts": 1770848395000 }
}
```

Add `"mfq": "5"` as the **last** field when you want to declare a minimum fill quantity (V2 optional field). Omit it if not needed.

### Field reference

The 14 required fields plus one optional `mfq`. Order is fixed by the contract's `SignQuote` struct in `rfq-contract/src/handler/types.rs` — do not reorder.

| # | Key | Name | Type | Description |
|---|---|---|---|---|
| 1 | `c` | chain_id | string | `"injective-888"` (testnet), `"injective-1"` (mainnet). |
| 2 | `ca` | contract_address | string | RFQ contract address. |
| 3 | `mi` | market_id | string | Derivative market ID. |
| 4 | `id` | rfq_id | **number (u64)** | Request ID — JSON number, NOT string. |
| 5 | `t` | taker | string | Taker's `inj1...` (from `request_address`). Optional per struct but every working implementation includes it. |
| 6 | `td` | taker_direction | string | `"long"` or `"short"`, lowercase. |
| 7 | `tm` | taker_margin | string | Taker's margin, normalized (no trailing zeros). |
| 8 | `tq` | taker_quantity | string | Taker's quantity, normalized. |
| 9 | `m` | maker | string | Your `inj1...`. |
| 10 | `ms` | maker_subaccount_nonce | **number (u32)** | Subaccount index for the maker. **Required** — use `0` if you don't use subaccounts. Missing = signature reject. |
| 11 | `mq` | maker_quantity | string | Your quoted quantity, normalized. |
| 12 | `mm` | maker_margin | string | Your margin commitment, normalized. |
| 13 | `p` | price | string | Your quoted price. **Must be pre-quantized to market tick size.** |
| 14 | `e` | expiry | **object** | Wrapped enum: `{"ts": <unix_ms>}` (timestamp variant) or `{"h": <block_height>}` (height variant). Timestamp is the common case. |
| 15 | `mfq` | min_fill_quantity | string \| omitted | Optional V2 field — minimum partial fill you'll accept. Omit entirely if not declaring one. |

> ⚠️ **Critical gotchas.**
> - `id` is a JSON **number**, not a string. JSON encoders that stringify big ints will produce bytes the contract rejects.
> - `ms` is a JSON **number** and is **required**. Older docs omit it — they're stale; the `ms` field was added during the Zenith audit pass (commit `6acac03` "bind quotes to maker subaccount"). A payload without it will fail signature verification on-chain.
> - `e` is a JSON **object** `{"ts": <ms>}`, not a raw integer. The contract's `Expiry` type is an enum and serde requires the variant tag.
> - Normalize decimals: strip trailing zeros after the decimal point (`"15.500000"` → `"15.5"`, but `"500"` stays `"500"`). Never use scientific notation. The contract's `FPDecimal::Display` format is the source of truth; both `rfq-testing` and `rfq-qa-python-tests` implement this as `_normalize_decimal_str`.
> - Serialize with no spaces: `separators=(",", ":")` in Python, default `JSON.stringify()` in TS/JS. Never use `sort_keys=True`.

### 3. Sign

```
1. JSON.stringify(payload)     → canonical JSON string (no spaces)
2. keccak256(json_bytes)       → 32-byte hash
3. secp256k1_sign(hash, key)   → 65-byte signature (r + s + v)
```

This is **not** EIP-191. Do **not** use `eth_sign` / `personal_sign` — they prepend `"\x19Ethereum Signed Message:\n"`. This is a raw `keccak256` hash signed directly.

#### Python (reference: `rfq-testing` and `rfq-qa-python-tests`)

The canonical implementation is in `rfq-testing/src/rfq_test/crypto/signing.py::sign_quote`. Use it directly if you're integrating in Python:

```python
from rfq_test.crypto.signing import sign_quote

signature_hex = sign_quote(
    private_key           = MM_PRIVATE_KEY,
    rfq_id                = str(rfq_id),
    market_id             = market_id,
    direction             = "long",                     # taker's direction
    taker                 = taker_address,
    taker_margin          = "100",
    taker_quantity        = "10",
    maker                 = mm_address,
    maker_margin          = "100",
    maker_quantity        = "10",
    price                 = "4.5",                      # pre-quantized to tick size
    expiry                = int(time.time() * 1000) + 30_000,
    chain_id              = "injective-888",
    contract_address      = CONTRACT_ADDRESS,
    maker_subaccount_nonce = 0,                         # ms — required
    min_fill_quantity     = None,                       # mfq — optional (V2)
)

# indexer expects "0x" prefix on the wire
await mm_ws.send_quote({..., "signature": "0x" + signature_hex})
```

If you're building from scratch, here's the full implementation (match this byte-for-byte):

```python
import json
from dataclasses import dataclass
from decimal import Decimal
from typing import Optional, Union
from eth_account import Account
from eth_hash.auto import keccak


def _normalize_decimal_str(value: Union[str, Decimal, None]) -> str:
    if value is None:
        return ""
    s = format(Decimal(str(value)), "f")          # avoid scientific notation
    return s.rstrip("0").rstrip(".") if "." in s else s


def build_sign_payload(
    *, chain_id, contract_address, market_id, rfq_id, taker, direction,
    taker_margin, taker_quantity, maker, maker_subaccount_nonce,
    maker_quantity, maker_margin, price, expiry_ms,
    min_fill_quantity=None,
) -> dict:
    out = {
        "c":  chain_id,
        "ca": contract_address,
        "mi": market_id,
        "id": int(rfq_id),
        "t":  taker,
        "td": direction.lower(),
        "tm": _normalize_decimal_str(taker_margin),
        "tq": _normalize_decimal_str(taker_quantity),
        "m":  maker,
        "ms": int(maker_subaccount_nonce),
        "mq": _normalize_decimal_str(maker_quantity),
        "mm": _normalize_decimal_str(maker_margin),
        "p":  _normalize_decimal_str(price),
        "e":  {"ts": int(expiry_ms)},
    }
    if min_fill_quantity is not None:
        out["mfq"] = _normalize_decimal_str(min_fill_quantity)
    return out


def sign_quote_payload(payload: dict, private_key_hex: str) -> str:
    json_str  = json.dumps(payload, separators=(",", ":"))
    msg_hash  = keccak(json_str.encode("utf-8"))
    account   = Account.from_key(bytes.fromhex(private_key_hex.removeprefix("0x")))
    signature = account.unsafe_sign_hash(msg_hash)
    sig_bytes = (
        signature.r.to_bytes(32, "big")
        + signature.s.to_bytes(32, "big")
        + bytes([signature.v])
    )
    return sig_bytes.hex()                               # no 0x prefix; add when sending
```

#### TypeScript

```typescript
import { keccak256, toUtf8Bytes, ethers } from "ethers";

function normalizeDecimal(value: string | number): string {
  const s = typeof value === "number" ? value.toString() : value;
  return s.includes(".") ? s.replace(/0+$/, "").replace(/\.$/, "") : s;
}

type SignQuoteInput = {
  chainId: string;
  contractAddress: string;
  marketId: string;
  rfqId: number;
  taker: string;
  direction: "long" | "short";
  takerMargin: string;
  takerQuantity: string;
  maker: string;
  makerSubaccountNonce: number;            // required (ms)
  makerQuantity: string;
  makerMargin: string;
  price: string;                            // pre-quantized to tick size
  expiryMs: number;
  minFillQuantity?: string;                 // optional (mfq)
};

function buildSignPayload(i: SignQuoteInput): Record<string, unknown> {
  const p: Record<string, unknown> = {
    c:  i.chainId,
    ca: i.contractAddress,
    mi: i.marketId,
    id: i.rfqId,
    t:  i.taker,
    td: i.direction,
    tm: normalizeDecimal(i.takerMargin),
    tq: normalizeDecimal(i.takerQuantity),
    m:  i.maker,
    ms: i.makerSubaccountNonce,
    mq: normalizeDecimal(i.makerQuantity),
    mm: normalizeDecimal(i.makerMargin),
    p:  normalizeDecimal(i.price),
    e:  { ts: i.expiryMs },
  };
  if (i.minFillQuantity !== undefined) {
    p.mfq = normalizeDecimal(i.minFillQuantity);
  }
  return p;
}

function signQuote(input: SignQuoteInput, privateKeyHex: string): string {
  const payload = buildSignPayload(input);
  const jsonStr = JSON.stringify(payload);
  const hash    = keccak256(toUtf8Bytes(jsonStr));
  const key     = new ethers.SigningKey(
    Buffer.from(privateKeyHex.replace(/^0x/, ""), "hex"),
  );
  const sig = key.sign(hash);
  return ethers.Signature.from(sig).serialized;          // returns 0x-prefixed 65-byte hex
}
```
