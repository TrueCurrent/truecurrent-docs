---
title: "Authorization setup (maker)"
description: "Recipe: the two authz grants a maker wallet must give the TrueCurrent contract before its quotes can settle. Python code sample, verification, and revocation."
updatedAt: "2026-05-05"
---

This is the operational recipe for granting the TrueCurrent contract the two message types it needs to settle your maker-side fills. It is a one-time setup per wallet.

For the conceptual model — why authz, what it protects, what the security tradeoffs are — see [Authorization model](/technical/authz-model). For the taker-side recipe (three grants), see [Takers — Authorization setup](/takers/authz-setup).

---

## Required grants

As a maker, you need to grant the following message types to the TrueCurrent contract:

| Message type | Purpose |
|---|---|
| `/injective.exchange.v2.MsgPrivilegedExecuteContract` | Allows the contract to execute privileged exchange actions (open positions, settle trades) |
| `/cosmos.bank.v1beta1.MsgSend` | Allows the contract to transfer collateral as needed for margin management |

<Info>
**Note on `MsgWithdraw`**:

The contract source (`admin.md`) lists `MsgWithdraw` as a canonical grant,
but the working testnet setup omits it.
The current settlement path does not exercise it,
and granting it creates an unused attack surface.
Do not grant `MsgWithdraw` unless the TrueCurrent team explicitly requests it.
</Info>

---

## Setting up grants

**Using the TrueCurrent UI:**

*(Once available)* The TrueCurrent app will guide you through granting the required permissions with a one-click flow.

**Programmatic setup (Python):**

```python
import asyncio
import os
# from dotenv import load_dotenv; load_dotenv()  # uncomment if not pre-sourcing .env

from rfq_test.clients.chain import ChainClient

MM_PRIVATE_KEY = os.getenv("MM_PRIVATE_KEY")
CONTRACT_ADDRESS = os.getenv("CONTRACT_ADDRESS")

chain = ChainClient(env_config.chain)
await chain.connect()

MM_MSG_TYPES = [
    "/injective.exchange.v2.MsgPrivilegedExecuteContract",
    "/cosmos.bank.v1beta1.MsgSend",
]

for msg_type in MM_MSG_TYPES:
    tx_hash = await chain.grant_authz(
        private_key=MM_PRIVATE_KEY,
        grantee=CONTRACT_ADDRESS,          # TrueCurrent contract address
        msg_type=msg_type,
    )
    print(f"Granted {msg_type}: {tx_hash}")
    await asyncio.sleep(3)                 # wait for LCD to catch up; <3s causes sequence mismatch

await chain.close()
```

Each grant requires a separate transaction. Wait **3 seconds** between grants — faster submission outpaces the LCD node and causes `account sequence mismatch` errors.

> **Pre-source `.env`** before running standalone scripts: `set -a; . .env; set +a` in your shell, or uncomment `load_dotenv()` in the script above. Without it, `os.getenv(...)` returns `None` for every key and the script crashes with `AttributeError`.

---

## Grant expiry

By default, authz grants can be set to expire after a specified duration. TrueCurrent recommends setting grants with no expiry (or a very long expiry), so your maker operations aren't interrupted by expired authorizations.

If you set an expiry, make sure to renew grants before they expire. Expired grants will cause settlement failures.

---

## Verifying your grants

To confirm your grants are in place, query the Injective chain:

```python
# Query existing grants for your address
grants = await chain.query_authz_grants(
    granter=mm_wallet.inj_address,
    grantee=CONTRACT_ADDRESS,
)
print(grants)
```

If you attempt to settle a trade without the required grants, the transaction will fail with an `unauthorized` error. Re-run the grant setup if this happens.

---

## Revoking grants

You can revoke your authz grants at any time, which immediately prevents the contract from submitting further settlement transactions on your behalf:

```python
await chain.revoke_authz(
    private_key=MM_PRIVATE_KEY,
    grantee=CONTRACT_ADDRESS,
    msg_type="/injective.exchange.v2.MsgPrivilegedExecuteContract",
)
```

Revoking grants does not close any open positions. Your existing positions remain open and can be managed manually.

---

## Security model

Grants are scoped to specific message types and a specific grantee address (the TrueCurrent contract); they are revocable at any time, and the contract's own onchain checks (signature verification, whitelist check, balance check) are the actual security boundary regardless of what authz allows. For the full security analysis and trust-model walkthrough, see [Authorization model](/technical/authz-model#why-not-just-use-allowances-or-signatures-per-trade).

For production, use a dedicated maker wallet separate from your personal or treasury wallet.
