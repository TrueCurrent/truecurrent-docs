---
title: "Testnet runbook"
description: "Operational testnet checklist for SDK traders: environment setup, wallet config, balances, authz, smoke tests, E2E AcceptQuote settlement, and TP/SL signed-intent validation."
updatedAt: "2026-05-06"
---

Use this runbook when validating a taker or maker SDK integration on testnet. It is intentionally procedural: complete the steps in order, fix failures before moving on, then run the full settlement flow.

This page is adapted from the RFQ onboarding runbook and assumes you are using the `rfq-testing` reference client.

---

## 1. Install the reference client

```bash
cd rfq-testing
python3 -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -e ".[dev]"
```

---

## 2. Configure environment

Create `.env` with the role keys you need.

```bash
RFQ_ENV=testnet

TESTNET_MM_PRIVATE_KEY=<hex>
TESTNET_RETAIL_PRIVATE_KEY=<hex>
# TESTNET_ADMIN_PRIVATE_KEY=<hex>   # only for register_maker
# TESTNET_RELAYER_PRIVATE_KEY=<hex> # only for AcceptSignedIntent
```

Pre-source `.env` before running standalone snippets:

```bash
set -a
. ./.env
set +a
```

---

## 3. Verify balances

Each active wallet needs INJ for gas and USDC margin in the relevant exchange subaccount.

```bash
curl -s \
  "https://testnet.sentry.lcd.injective.network/cosmos/bank/v1beta1/balances/<address>" \
  | python3 -m json.tool
```

For exchange subaccount balances, query the Injective exchange module or use the balance helpers in `rfq-testing`.

---

## 4. Verify maker whitelist

Market makers must be registered before MakerStream routes requests to them and before their quotes can settle.

```bash
curl -s \
  "https://testnet.sentry.lcd.injective.network/cosmwasm/wasm/v1/contract/inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk/smart/$(echo -n '{\"list_makers\":{}}' | base64)" \
  | python3 -m json.tool
```

Use the contract client's `list_makers` helper when available; it avoids hand-building the encoded smart query.

---

## 5. Grant authz

If grant queries return an empty list, run the setup from [Authorization setup](/sdk-trading/authz).

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

---

## 6. Smoke test

This confirms config loading, MakerStream reachability, chain reachability, and EIP-712 v2 signing. It does not prove the MakerStream auth handshake is fully exercised; the full E2E flow does that.

```python
import asyncio
import os
import time

os.environ["RFQ_ENV"] = "testnet"

from rfq_test.config import get_environment_config
from rfq_test.clients import ChainClient, MakerStreamClient
from rfq_test.crypto.wallet import Wallet
from rfq_test.crypto.eip712 import sign_quote_v2

env = get_environment_config()
evm_chain_id, contract_addr = env.signing_context_v2
print(f"[1/4] config loads: contract={contract_addr} evm_chain_id={evm_chain_id}")

async def ws_check():
    ws = MakerStreamClient(
        env.indexer.ws_endpoint,
        maker_address="<MM_ADDR>",
        timeout=10.0,
    )
    await ws.connect()
    await ws.close()
    print("[2/4] MakerStream WS connects")

asyncio.run(ws_check())

async def chain_check():
    chain = ChainClient(env.chain)
    await chain.connect()
    await chain.close()
    print(f"[3/4] chain client connects: chain_id={env.chain.chain_id}")

asyncio.run(chain_check())

maker = Wallet.from_private_key(os.getenv("TESTNET_MM_PRIVATE_KEY"))
taker = Wallet.from_private_key(os.getenv("TESTNET_RETAIL_PRIVATE_KEY"))

sig = sign_quote_v2(
    private_key=maker.private_key,
    evm_chain_id=evm_chain_id,
    verifying_contract_bech32=contract_addr,
    market_id=env.markets[0].id,
    rfq_id=1,
    taker=taker.inj_address,
    direction="long",
    taker_margin="1",
    taker_quantity="1",
    maker=maker.inj_address,
    maker_subaccount_nonce=0,
    maker_margin="1",
    maker_quantity="1",
    price="1",
    expiry_ms=int(time.time() * 1000) + 2_000,
    min_fill_quantity=None,
)
print(f"[4/4] sign_quote_v2 returns signature len={len(sig)}")
```

---

## 7. Full E2E settlement

The full testnet cycle is:

1. Taker connects to TakerStream.
2. Maker connects to MakerStream and answers `MakerChallenge`.
3. Taker sends RFQ request and receives the indexer-assigned `rfq_id`.
4. Maker waits for that `rfq_id`, signs a quote with EIP-712 v2, and sends it.
5. Taker collects matching quotes.
6. Taker submits `AcceptQuote` onchain.
7. Maker receives quote and settlement updates.

Use `examples/test_settlement.py` in `rfq-testing` as the canonical executable script.

```bash
set -a
. ./.env
set +a

python examples/test_settlement.py
```

The MakerStream client must be constructed with auth fields so it can answer the inbound challenge:

```python
mm_client = MakerStreamClient(
    config.indexer.ws_endpoint,
    maker_address=mm_wallet.inj_address,
    subscribe_to_quotes_updates=True,
    subscribe_to_settlement_updates=True,
    auth_private_key=mm_private_key,
    auth_evm_chain_id=1439,
    auth_contract_address=contract_address,
    timeout=10.0,
)
```

If the maker connects but never receives `request` events, check the auth challenge first.

---

## 8. TP/SL signed-intent test

TP/SL exits are taker-signed conditional orders. The taker signs a `SignedTakerIntent`, submits it through TakerStream, and a relayer executes `AcceptSignedIntent` when mark price crosses the trigger.

```python
import os
import time

from rfq_test.crypto.wallet import Wallet
from rfq_test.crypto.eip712 import sign_conditional_order_v2
from rfq_test.clients.websocket import TakerStreamClient

taker = Wallet.from_private_key(os.environ["TESTNET_RETAIL_PRIVATE_KEY"])
rfq_id = int(time.time() * 1000)
deadline_ms = rfq_id + 86_400_000

intent_sig = sign_conditional_order_v2(
    private_key=taker.private_key,
    evm_chain_id=evm_chain_id,
    verifying_contract_bech32=contract_address,
    version=1,
    taker=taker.inj_address,
    epoch=1,
    lane_version=1,
    subaccount_nonce=0,
    rfq_id=rfq_id,
    market_id=MARKET.id,
    deadline_ms=deadline_ms,
    direction="short",
    quantity="1",
    margin="0",
    worst_price="19.5",
    min_total_fill_quantity="1",
    trigger_type="mark_price_lte",
    trigger_price="20.0",
)

async with TakerStreamClient(
    env.indexer.ws_endpoint,
    request_address=taker.inj_address,
) as client:
    ack = await client.send_conditional_order(
        order_body={
            "version": 1,
            "chain_id": chain_id,
            "contract_address": contract_address,
            "taker": taker.inj_address,
            "epoch": 1,
            "rfq_id": rfq_id,
            "market_id": MARKET.id,
            "subaccount_nonce": 0,
            "lane_version": 1,
            "deadline_ms": deadline_ms,
            "direction": "short",
            "quantity": "1",
            "margin": "0",
            "worst_price": "19.5",
            "min_total_fill_quantity": "1",
            "trigger_type": "mark_price_lte",
            "trigger_price": "20.0",
            "unfilled_action": None,
            "cid": None,
            "allowed_relayer": None,
            "evm_chain_id": evm_chain_id,
        },
        signature=intent_sig,
        sign_mode="v2",
        evm_chain_id=evm_chain_id,
        wait_for_ack=True,
    )

print(f"Signed intent ACK: rfq_id={ack['rfq_id']} status={ack['status']}")
```

Conditional-order payloads must carry `sign_mode="v2"` and `evm_chain_id`. TakerStream may name these fields `conditional_order_sign_mode` and `conditional_order_evm_chain_id` internally; the reference client sets them when you pass `sign_mode` and `evm_chain_id`.

---

## 9. Cancel signed intents

Use `CancelIntentLane` to cancel all active intents for one `(taker, market_id, subaccount_nonce)` lane.

```python
from rfq_test.clients.contract import ContractClient

contract = ContractClient(env.contract, env.chain)

tx_hash = await contract.cancel_intent_lane(
    private_key=RETAIL_PRIVATE_KEY,
    market_id=MARKET.id,
    subaccount_nonce=0,
)
print(f"Cancelled lane: {tx_hash}")
```

Use `CancelAllIntents` only when you need to invalidate every lane for the taker.

---

## Common blockers

| Symptom | Likely cause | Fix |
|---|---|---|
| Maker stream connects but receives no requests | Maker did not answer `MakerChallenge` | Configure `auth_private_key`, `auth_evm_chain_id`, and `auth_contract_address` |
| `authorization not found` | Missing or expired authz grant | Re-run [Authorization setup](/sdk-trading/authz) |
| Quote rejected for signature | Decimal string changed after signing, wrong EVM chain ID, or wrong contract domain | Quantize before signing and send the exact signed strings |
| No quotes collected | Maker not whitelisted, offline, or filtering the wrong `rfq_id` | Verify whitelist and correlate from ACK-returned `rfq_id` |
| `all quotes rejected` | Expired quote, failed `worst_price`, mark-band rejection, or maker balance issue | Inspect quote results and maker balances |
| Conditional order accepted but never fires | Trigger not reached, relayer unavailable, expired intent, or no executable maker quote | List intent status and recreate if needed |

For the full protocol details, see [Protocol reference](/sdk-trading/protocol-reference) and [Troubleshooting](/sdk-trading/troubleshooting).
