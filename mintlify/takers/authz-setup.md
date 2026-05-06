---
title: "Authorization setup (taker)"
description: "Recipe: the three authz grants a programmatic taker wallet must give the TrueCurrent contract before any trade can settle. Python and TypeScript code samples, verification, and revocation."
updatedAt: "2026-05-05"
---

This is the operational recipe for granting a TrueCurrent contract the three message types it needs to settle trades on your behalf. It is a one-time setup per wallet. Once grants are in place, every `AcceptQuote` settles atomically without requiring per-trade signatures.

For the conceptual model — why authz, what it protects, what the security tradeoffs are — see [Authorization model](/technical/authz-model). For the maker-side recipe (two grants instead of three), see [Makers — Authorization setup](/market-makers/authz-setup).

---

## Required grants

A programmatic taker must grant the following three message types to the TrueCurrent contract address. This is the canonical set the gold-standard onboarding script grants and the test framework's `RETAIL_AUTHZ_GRANTS` exposes — granting more is unnecessary attack surface, granting fewer breaks settlement.

| Message type | Purpose |
|---|---|
| `/injective.exchange.v2.MsgPrivilegedExecuteContract` | Open derivative positions via the exchange module |
| `/injective.exchange.v2.MsgBatchUpdateOrders` | Reserved orderbook permission for future contract paths |
| `/cosmos.bank.v1beta1.MsgSend` | Move margin funds as part of settlement |

---

## Setting up grants

**Python** – using the helper from `injective-rfq-toolkit`:

```python
from rfq_test.clients.chain import ChainClient
from rfq_test.config import get_environment_config
from rfq_test.crypto.wallet import Wallet
from rfq_test.utils.setup import RETAIL_AUTHZ_GRANTS, setup_authz_grants

config = get_environment_config()
chain = ChainClient(config.chain)
await chain.connect()

taker = Wallet.from_private_key(TAKER_PRIVATE_KEY)

await setup_authz_grants(
    chain_client=chain,
    wallet=taker,
    contract_address=config.contract.address,
    msg_types=RETAIL_AUTHZ_GRANTS,
)

await chain.close()
```

`RETAIL_AUTHZ_GRANTS` is the list of three message types above. Each becomes its own transaction – the helper waits for each to confirm before submitting the next.

**Python** – without the helper, using `injective-py` directly:

```python
from pyinjective.composer_v2 import Composer
from pyinjective.core.broadcaster import MsgBroadcasterWithPk

composer = Composer(network=network.string())
CONTRACT_ADDRESS = "inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk"

MSG_TYPES = [
    "/injective.exchange.v2.MsgPrivilegedExecuteContract",
    "/injective.exchange.v2.MsgBatchUpdateOrders",
    "/cosmos.bank.v1beta1.MsgSend",
]

for msg_type in MSG_TYPES:
    msg = composer.msg_grant_generic(
        granter=taker_address,
        grantee=CONTRACT_ADDRESS,
        msg_type=msg_type,
        expire_in=0,                  # 0 = no expiry
    )
    result = await broadcaster.broadcast([msg])
    print(f"Granted {msg_type}: {result.tx_response.txhash}")
```

**TypeScript** – using `@injectivelabs/sdk-ts`:

```ts
import {
  MsgGrant,
  MsgBroadcasterWithPk,
  getGenericAuthorizationFromMessageType,
} from "@injectivelabs/sdk-ts";
import { Network } from "@injectivelabs/networks";

const CONTRACT_ADDRESS = "inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk";

const MSG_TYPES = [
  "/injective.exchange.v2.MsgPrivilegedExecuteContract",
  "/injective.exchange.v2.MsgBatchUpdateOrders",
  "/cosmos.bank.v1beta1.MsgSend",
];

const broadcaster = new MsgBroadcasterWithPk({
  privateKey: TAKER_PRIVATE_KEY,
  network: Network.TestnetSentry,
});

for (const messageType of MSG_TYPES) {
  const msg = MsgGrant.fromJSON({
    granter: takerInjAddress,
    grantee: CONTRACT_ADDRESS,
    authorization: getGenericAuthorizationFromMessageType(messageType),
  });

  const { txHash } = await broadcaster.broadcast({ msgs: msg });
  console.log(`Granted ${messageType}: ${txHash}`);
}
```

Submit grants sequentially – account sequence number increments on each, and parallel submissions will collide.

---

## Grant expiry

By default, `authz` grants can include an expiration time. TrueCurrent recommends setting **no expiry** (pass `0` or `undefined` depending on the SDK) so you're not surprised by a failed settlement six months from now.

If you must set an expiry, renew grants well before they lapse. An expired grant causes any `AcceptQuote` that depends on it to fail with `unauthorized`.

---

## Managing grants from a GUI

If you'd rather manage your authz grants from a web interface than the CLI, **[`do.injective.network`](https://do.injective.network)** has a built-in authz manager. You can view existing grants, create new ones, and revoke them, all signed from your connected wallet.

This is useful for ad-hoc inspection, for sanity-checking that your programmatic grants went through, and for the quick revoke button during an incident.

---

## Verifying grants

Check your active grants against the Injective chain:

**Python:**

```python
grants = await chain.query_authz_grants(
    granter=taker.inj_address,
    grantee=CONTRACT_ADDRESS,
)
for g in grants:
    print(g.authorization.msg, "expires:", g.expiration)
```

**REST:**

```
GET https://testnet.sentry.lcd.injective.network/cosmos/authz/v1beta1/grants
    ?granter=inj1taker...
    &grantee=inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk
```

**CLI:**

```bash
injectived query authz grants \
  inj1taker... \
  inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk \
  --node https://testnet.sentry.tm.injective.network:443
```

You should see three entries, one per message type.

---

## Revoking grants

You can revoke any grant at any time. Revocation is a standard Injective transaction:

**Python:**

```python
await chain.revoke_authz(
    private_key=TAKER_PRIVATE_KEY,
    grantee=CONTRACT_ADDRESS,
    msg_type="/injective.exchange.v2.MsgPrivilegedExecuteContract",
)
```

**TypeScript:**

```ts
import { MsgRevoke } from "@injectivelabs/sdk-ts";

const msg = MsgRevoke.fromJSON({
  granter: takerInjAddress,
  grantee: CONTRACT_ADDRESS,
  messageType: "/injective.exchange.v2.MsgPrivilegedExecuteContract",
});

await broadcaster.broadcast({ msgs: msg });
```

Revoking `MsgPrivilegedExecuteContract` immediately prevents any further settlements on this wallet – it is a kill switch.

**Revocation does not affect open positions.** Your existing positions remain open and can be managed through the standard Injective exchange module. Revoking only prevents *new* contract-initiated actions.

---

## Security model

Grants are scoped to specific message types and a specific grantee address (the TrueCurrent contract); they are revocable at any time, and the contract's own onchain checks (`worst_price`, signature, expiry) are the actual security boundary regardless of what authz allows. For the full security analysis and trust-model walkthrough, see [Authorization model](/technical/authz-model#why-not-just-use-allowances-or-signatures-per-trade).

For production systems, run takers from wallets dedicated to RFQ trading. Don't grant from a treasury or cold wallet.
