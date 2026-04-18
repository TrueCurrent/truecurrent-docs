---
title: "Authorization setup"
description: "--"
updatedAt: "2026-04-08"
---

Before you can settle any trades on TrueCurrent, your taker wallet must grant the TrueCurrent contract permission to submit specific message types on your behalf. This uses Injective's native `authz` module.

This is a one-time setup per wallet. Once grants are in place, every `AcceptQuote` you send can settle atomically in a single transaction without requiring multiple signatures.

For the conceptual model and security analysis, see [Authorization model](/technical/authz-model).

---

## Why takers need authz

When you call `AcceptQuote`, the TrueCurrent contract needs to:

1. Open your derivative position via `MsgPrivilegedExecuteContract`
2. Move margin between your wallet and subaccount if required (`MsgSend`)
3. Optionally route any unfilled quantity to the Injective order book (`MsgBatchUpdateOrders`, `MsgCreateDerivativeMarketOrder`)

Rather than making you sign each of these messages individually at settlement time, you pre-authorize the contract once. Each authorization is narrow – scoped to a specific message type and a specific grantee address (the contract).

---

## Required grants

{/* TODO: confirm the canonical taker authz grant list against the contract source (rfq-contract/src/handler/). The rfq-contract's own admin.md lists three grants for takers (MsgSend, MsgWithdraw, MsgBatchUpdateOrders) while this page lists four. Specifically: (a) is MsgWithdraw actually required when the contract needs to pull from the exchange subaccount?; (b) is MsgPrivilegedExecuteContract granted by the taker or only by the maker, given the synthetic trade opens the maker's position from the contract's authz context?; (c) is MsgCreateDerivativeMarketOrder needed separately or does MsgBatchUpdateOrders cover both limit and market fallback (see handler/orderbook.rs:27 comment)? Resolve before shipping to mainnet. */}

A programmatic taker must grant the following four message types to the TrueCurrent contract address:

| Message type | Purpose |
|---|---|
| `/cosmos.bank.v1beta1.MsgSend` | Move margin funds as part of settlement |
| `/injective.exchange.v2.MsgPrivilegedExecuteContract` | Open derivative positions via the exchange module |
| `/injective.exchange.v2.MsgBatchUpdateOrders` | Route unfilled quantity to the orderbook via limit fallback |
| `/injective.exchange.v2.MsgCreateDerivativeMarketOrder` | Route unfilled quantity to the orderbook via market fallback (IOC) |

If you never intend to use `unfilled_action: {"market": {}}` or `{"limit": {...}}`, you can skip the last two grants – but then `AcceptQuote` calls that would have routed a remainder to the orderbook will fail. Most takers **should grant all four**.

---

## Setting up grants

**Python** – using the helper from `rfq-testing`:

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

`RETAIL_AUTHZ_GRANTS` is the list of four message types above. Each becomes its own transaction – the helper waits for each to confirm before submitting the next. Total setup takes ~5 seconds.

**Python** – without the helper, using `injective-py` directly:

```python
from pyinjective.composer_v2 import Composer
from pyinjective.core.broadcaster import MsgBroadcasterWithPk

composer = Composer(network=network.string())
CONTRACT_ADDRESS = "inj1t8hyyle68vd0kzsdehxg0sywttrwmt58jzk29q"

MSG_TYPES = [
    "/cosmos.bank.v1beta1.MsgSend",
    "/injective.exchange.v2.MsgPrivilegedExecuteContract",
    "/injective.exchange.v2.MsgBatchUpdateOrders",
    "/injective.exchange.v2.MsgCreateDerivativeMarketOrder",
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

const CONTRACT_ADDRESS = "inj1t8hyyle68vd0kzsdehxg0sywttrwmt58jzk29q";

const MSG_TYPES = [
  "/cosmos.bank.v1beta1.MsgSend",
  "/injective.exchange.v2.MsgPrivilegedExecuteContract",
  "/injective.exchange.v2.MsgBatchUpdateOrders",
  "/injective.exchange.v2.MsgCreateDerivativeMarketOrder",
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
    &grantee=inj1t8hyyle68vd0kzsdehxg0sywttrwmt58jzk29q
```

**CLI:**

```bash
injectived query authz grants \
  inj1taker... \
  inj1t8hyyle68vd0kzsdehxg0sywttrwmt58jzk29q \
  --node https://testnet.sentry.tm.injective.network:443
```

You should see four entries, one per message type.

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

## Security model summary

- Grants are scoped to **specific message types** – the contract cannot transfer arbitrary funds, only perform the operations you authorized.
- Grants are scoped to **a specific grantee** – only the TrueCurrent contract address, nothing else.
- Grants are **revocable at any time**, effective immediately.
- The onchain contract logic enforces your trading parameters (`worst_price`, quote signature, quote expiry) – even a malicious contract operator could not forge a trade against your authz because the onchain verification catches it.

For production systems, run takers from wallets dedicated to RFQ trading. Don't grant from a treasury or cold wallet.
