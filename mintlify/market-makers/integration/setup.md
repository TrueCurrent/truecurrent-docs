---
title: "One-time setup"
description: "One-time onboarding steps for market makers: generate an Injective wallet, get whitelisted, grant authz permissions to the RFQ contract, and fund your exchange subaccount."
updatedAt: "2026-04-18"
---

### 1. Generate a wallet

Standard Ethereum-compatible secp256k1. The same private key produces both a `0x…` EVM address and an `inj1…` bech32 address.

```python
# Python
from eth_account import Account
import bech32

account = Account.create()
private_key = account.key.hex()

addr_bytes = bytes.fromhex(account.address[2:])
inj_address = bech32.bech32_encode("inj", bech32.convertbits(addr_bytes, 8, 5))
print(f"Private Key: {private_key}")
print(f"INJ Address: {inj_address}")
```

```typescript
// TypeScript
import { PrivateKey } from "@injectivelabs/sdk-ts";

const privateKey = PrivateKey.generate();
const injAddress = privateKey.toBech32();
console.log(`Private Key: ${privateKey.toHex()}`);
console.log(`INJ Address: ${injAddress}`);
```

### 2. Get whitelisted

Send your `inj1...` address to the TrueCurrent team. The contract admin calls `register_maker` to whitelist you.

### 3. Grant authz permissions

The RFQ contract settles trades on your behalf. You grant it narrow, message-type-scoped permissions — it cannot move arbitrary funds.

> **TODO (verify):** `rfq-contract/docs/admin.md` lists **three** grants for makers: `MsgSend`, `MsgWithdraw`, `MsgPrivilegedExecuteContract`. The working setup on current testnet integrations uses only the two shown below. Before going to mainnet, pair-check against `rfq-contract/src/handler/authz.rs` + `settlement.rs` for which grants the contract actually requires when settling from a custom vs default subaccount.

| Permission | Purpose |
|---|---|
| `/injective.exchange.v2.MsgPrivilegedExecuteContract` | Contract opens the maker-side synthetic position. |
| `/cosmos.bank.v1beta1.MsgSend` | Contract moves margin funds as part of settlement. |

```python
from pyinjective.composer_v2 import Composer
from pyinjective.core.broadcaster import MsgBroadcasterWithPk
from pyinjective.core.network import Network

network  = Network.testnet()
composer = Composer(network=network.string())

CONTRACT_ADDRESS = "inj1t8hyyle68vd0kzsdehxg0sywttrwmt58jzk29q"  # testnet 0.1.0-alpha.6

granted_msgs = [
    "/injective.exchange.v2.MsgPrivilegedExecuteContract",
    "/cosmos.bank.v1beta1.MsgSend",
]

authz_msgs = [
    composer.msg_grant_generic(
        granter=YOUR_INJ_ADDRESS,
        grantee=CONTRACT_ADDRESS,
        msg_type=msg_type,
        expire_in=365 * 24 * 3600,  # 1 year; TrueCurrent recommends no expiry in production
    )
    for msg_type in granted_msgs
]

broadcaster = MsgBroadcasterWithPk.new_using_gas_heuristics(
    network=network,
    private_key=YOUR_PRIVATE_KEY,
)
result = await broadcaster.broadcast(authz_msgs)
print(f"Authz granted! TxHash: {result['txResponse']['txhash']}")
```

### 4. Fund your subaccount

Deposit USDT into your Injective exchange subaccount — this is the margin backing your quotes.
