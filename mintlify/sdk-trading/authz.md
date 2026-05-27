---
title: "Authorization setup"
description: "Grant the TrueCurrent RFQ contract the narrow Injective authz permissions required for taker and maker SDK settlement."
updatedAt: "2026-05-27"
---

TrueCurrent settles RFQ trades through Injective's exchange module. To make settlement atomic, taker and maker wallets grant the RFQ contract narrow `authz` permissions before trading.

The grants are one-time setup per wallet. They are scoped to the current RFQ contract address, limited to specific message types, and revocable at any time.

---

## Required grants

| Role | Message type | Purpose |
| --- | --- | --- |
| Maker | `/injective.exchange.v2.MsgPrivilegedExecuteContract` | Lets the contract open or close maker-side derivative positions |
| Maker | `/cosmos.bank.v1beta1.MsgSend` | Lets the contract move maker margin during settlement |
| Taker | `/injective.exchange.v2.MsgPrivilegedExecuteContract` | Lets the contract open or close taker-side derivative positions |
| Taker | `/injective.exchange.v2.MsgBatchUpdateOrders` | Reserved exchange permission used by taker settlement paths |
| Taker | `/cosmos.bank.v1beta1.MsgSend` | Lets the contract move taker margin during settlement |

<Info>
`MsgWithdraw` appears in some contract notes as a canonical grant, but the current working setup intentionally omits it. The active settlement path does not exercise it, and granting unused permissions increases attack surface. If a future contract path requires a new message type, add it only after verifying the deployed contract and runbook.
</Info>

---

## Grant all testnet permissions

The script below grants the working testnet set for one maker wallet and one taker wallet. Run it after loading `.env` so the private keys are present.

```bash
set -a
. ./.env
set +a
python grant_all.py
```

```python
import asyncio
import os

os.environ["RFQ_ENV"] = "testnet"

from rfq_test.config import get_environment_config
from rfq_test.clients.chain import ChainClient

env = get_environment_config()
CONTRACT = env.contract.address

async def grant_all():
    chain = ChainClient(env.chain)
    await chain.connect()

    grants = [
        # Maker
        (os.getenv("TESTNET_MM_PRIVATE_KEY"), "/injective.exchange.v2.MsgPrivilegedExecuteContract"),
        (os.getenv("TESTNET_MM_PRIVATE_KEY"), "/cosmos.bank.v1beta1.MsgSend"),

        # Taker
        (os.getenv("TESTNET_RETAIL_PRIVATE_KEY"), "/injective.exchange.v2.MsgPrivilegedExecuteContract"),
        (os.getenv("TESTNET_RETAIL_PRIVATE_KEY"), "/injective.exchange.v2.MsgBatchUpdateOrders"),
        (os.getenv("TESTNET_RETAIL_PRIVATE_KEY"), "/cosmos.bank.v1beta1.MsgSend"),
    ]

    for private_key, msg_type in grants:
        if not private_key:
            raise RuntimeError(f"missing private key for {msg_type}")

        tx_hash = await chain.grant_authz(
            private_key=private_key,
            grantee=CONTRACT,
            msg_type=msg_type,
        )
        print(f"Granted {msg_type}: {tx_hash}")

        # Each grant is a separate tx. Wait for account sequence and LCD state.
        await asyncio.sleep(3)

    await chain.close()

asyncio.run(grant_all())
```

Submit grant transactions sequentially. Parallel grant broadcasts commonly fail with account sequence mismatch errors.

---

## Verify grants

Use the Injective LCD to confirm grants before running settlement tests.

```bash
# Maker grants
curl -s \
  "https://testnet.sentry.lcd.injective.network/cosmos/authz/v1beta1/grants?granter=<MM_ADDR>&grantee=inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk" \
  | python3 -m json.tool

# Taker grants
curl -s \
  "https://testnet.sentry.lcd.injective.network/cosmos/authz/v1beta1/grants?granter=<TAKER_ADDR>&grantee=inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk" \
  | python3 -m json.tool
```

Expected result:

- Maker grants include `MsgPrivilegedExecuteContract` and `MsgSend`.
- Taker grants include `MsgPrivilegedExecuteContract`, `MsgBatchUpdateOrders`, and `MsgSend`.
- The `grantee` is the current RFQ contract address.

---

## Direct GenericAuthorization

If your Injective SDK helper forces an expiration or uses a specialized authorization, build the grant manually as `GenericAuthorization`.

```python
from google.protobuf import any_pb2
from pyinjective.proto.cosmos.authz.v1beta1 import authz_pb2
from pyinjective.proto.cosmos.authz.v1beta1 import tx_pb2 as authz_tx_pb2

def create_grant_msg(granter: str, grantee: str, msg_type: str):
    generic_authz = authz_pb2.GenericAuthorization()
    generic_authz.msg = msg_type

    authz_any = any_pb2.Any()
    authz_any.type_url = "/cosmos.authz.v1beta1.GenericAuthorization"
    authz_any.value = generic_authz.SerializeToString()

    grant = authz_pb2.Grant()
    grant.authorization.CopyFrom(authz_any)
    # Leave grant.expiration unset if you want no expiration.

    grant_msg = authz_tx_pb2.MsgGrant()
    grant_msg.granter = granter
    grant_msg.grantee = grantee
    grant_msg.grant.CopyFrom(grant)
    return grant_msg
```

Use gas heuristics for grant broadcasts. Simulation can under-estimate grant transaction gas and produce avoidable failures.

---

## Revoking grants

Revoking `MsgPrivilegedExecuteContract` is the fastest kill switch. It prevents future contract-initiated settlements for that wallet.

```python
await chain.revoke_authz(
    private_key=PRIVATE_KEY,
    grantee=CONTRACT,
    msg_type="/injective.exchange.v2.MsgPrivilegedExecuteContract",
)
```

Revocation does not close open positions. It only prevents future TrueCurrent contract settlement actions for that wallet.

---

## Security checklist

- Use dedicated maker and taker wallets.
- Grant only the message types listed above.
- Grant only to the current RFQ contract address.
- Re-verify grants after contract upgrades.
- Keep a revocation path ready before going live.
- Monitor grant presence at bot startup.
- Re-run grant setup if settlement fails with `unauthorized` or `authorization not found`.

For the broader trust model, see [Authorization model](/technical/authz-model).
