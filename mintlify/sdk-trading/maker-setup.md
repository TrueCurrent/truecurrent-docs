---
title: "Maker setup"
description: "One-time onboarding steps for TrueCurrent makers: wallet generation, whitelist registration, authz grants, and exchange subaccount funding."
updatedAt: "2026-05-27"
---

Maker setup is the ceremony you complete before your system can quote. Do this once per maker wallet and repeat it whenever you rotate keys, change subaccount nonce, or move to a new RFQ contract.

---

## 1. Generate a dedicated wallet

Use a dedicated maker wallet. Do not grant from a treasury or omnibus wallet.

Injective uses Ethereum-compatible `secp256k1` keys. The same private key has both an EVM `0x...` address and an Injective `inj1...` bech32 address.

```python
from eth_account import Account
import bech32

account = Account.create()
private_key = account.key.hex()

addr_bytes = bytes.fromhex(account.address[2:])
inj_address = bech32.bech32_encode(
    "inj",
    bech32.convertbits(addr_bytes, 8, 5),
)

print(f"Private Key: {private_key}")
print(f"INJ Address: {inj_address}")
```

```typescript
import { PrivateKey } from "@injectivelabs/sdk-ts";

const privateKey = PrivateKey.generate();
const injAddress = privateKey.toBech32();

console.log(`Private Key: ${privateKey.toHex()}`);
console.log(`INJ Address: ${injAddress}`);
```

Store the private key in your secret manager and expose it to the maker process only as needed for MakerStream auth and quote signing.

---

## 2. Get whitelisted

Send your maker `inj1...` address to the TrueCurrent team with the markets you plan to support. The contract admin calls `register_maker` for that address.

After registration, verify that your address appears in `list_makers`. The maker record may also specify a `subaccount_nonce`. If it is `null`, use nonce `0`. If it is nonzero, quote and fund that exact subaccount nonce.

See [Maker whitelist](/sdk-trading/maker-whitelist) for the verification queries.

---

## 3. Grant authz permissions

The RFQ contract settles accepted quotes on your behalf. It can only use the message types you grant, and only as the configured grantee contract.

For makers, the working grant set is:

| Permission | Purpose |
| --- | --- |
| `/injective.exchange.v2.MsgPrivilegedExecuteContract` | Opens or closes the maker-side derivative position through the exchange module |
| `/cosmos.bank.v1beta1.MsgSend` | Moves margin funds as part of settlement |

<Info>
`MsgWithdraw` appears in some contract notes as a canonical grant, but the current working setup intentionally omits it. The active settlement path does not exercise it, and granting unused permissions increases attack surface.
</Info>

Use the shared [Authorization setup](/sdk-trading/authz) page for the combined maker and taker grant script.

---

## 4. Fund the registered subaccount

Keep enough USDC in the maker exchange subaccount to back accepted quotes. Settlement uses the maker subaccount nonce recorded in `list_makers`.

| Maker registration | Quote with | Fund |
| --- | --- | --- |
| `subaccount_nonce: null` | `maker_subaccount_nonce: 0` | Default exchange subaccount |
| `subaccount_nonce: 0` | `maker_subaccount_nonce: 0` | Default exchange subaccount |
| `subaccount_nonce: N` | `maker_subaccount_nonce: N` | The matching nonzero exchange subaccount |

Also keep enough INJ for setup transactions and operational account actions.

---

## 5. Configure the maker process

At minimum, your process needs:

```bash
RFQ_ENV=testnet
TESTNET_MM_PRIVATE_KEY=<hex>
```

It also needs the current RFQ contract address, EVM chain ID, Cosmos chain ID, market IDs, tick sizes, and MakerStream endpoint. Prefer loading these from `injective-rfq-toolkit` environment config rather than hardcoding.

For the full testnet reference, see [Testnet configuration](/sdk-trading/testnet-configuration).

---

## 6. Run the E2E flow

Before mainnet, prove the full ceremony on testnet:

1. Wallet derives to the expected `inj1...`.
2. Maker appears in `list_makers`.
3. Maker and taker authz grants exist.
4. Maker and taker balances are sufficient.
5. MakerStream connects and answers `MakerChallenge`.
6. Maker receives an RFQ request.
7. Maker signs and sends a quote with `sign_mode: "v2"` and `evm_chain_id`.
8. Taker collects the quote and submits `AcceptQuote`.
9. Settlement creates the expected derivative positions.

Use [Testnet runbook](/sdk-trading/runbook) as the operational checklist.
