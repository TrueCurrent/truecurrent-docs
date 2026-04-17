---
title: "Authorization setup (authz)"
description: "Step-by-step guide to setting up authz grants on Injective for TrueCurrent market makers, including required message types, Python implementation, grant verification, and security best practices for automated settlement."
updatedAt: "2026-04-06"
---

TrueCurrent uses Injective's native `authz` module to allow the smart contract to execute trades on behalf of both traders and market makers. As a market maker, you need to grant the contract specific permissions before your quotes can settle.

---

## What is authz?

The Cosmos SDK `authz` module allows one address (the *granter*) to authorize another address (the *grantee*) to send specific message types on their behalf. On TrueCurrent, the *grantee* is the TrueCurrent smart contract.

This is how settlement works without requiring you to manually sign every trade: when a trader accepts a quote, the contract has pre-authorized permission to submit the settlement transaction using your wallet as counterparty.

---

## Required grants

{/* TODO: CK – confirm the canonical maker authz grant list against rfq-contract. The rfq-contract admin.md lists THREE grants for makers (MsgSend, MsgWithdraw, MsgPrivilegedExecuteContract) — this page is currently missing MsgWithdraw. Resolve which grants are actually required (including whether MsgWithdraw is needed for custom-subaccount makers) before updating this table. */}

As a market maker, you need to grant the following message types to the TrueCurrent contract:

| Message type | Purpose |
|---|---|
| `/injective.exchange.v2.MsgPrivilegedExecuteContract` | Allows the contract to execute privileged exchange actions (open positions, settle trades) |
| `/cosmos.bank.v1beta1.MsgSend` | Allows the contract to transfer collateral as needed for margin management |

---

## Setting up grants

**Using the TrueCurrent UI:**

*(Once available)* The TrueCurrent app will guide you through granting the required permissions with a one-click flow.

**Programmatic setup (Python):**

```python
from rfq_test.clients.chain import ChainClient

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

await chain.close()
```

Each grant requires a separate transaction. Wait for each transaction to confirm (approximately 1–2 seconds) before submitting the next.

---

## Grant expiry

By default, authz grants can be set to expire after a specified duration. TrueCurrent recommends setting grants with no expiry (or a very long expiry), so your market making operations aren't interrupted by expired authorizations.

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

## Security note

Authz grants are scoped to specific message types and the specific contract address. You're not giving the contract open-ended access to your wallet – only the ability to submit the specific message types listed above. Review the grant parameters carefully before signing.

For maximum security in production, use a dedicated market making wallet separate from your personal wallet.
