---
title: "Runbook"
description: "Procedural testnet checklist for validating TrueCurrent RFQ taker, maker, settlement, and TP/SL signed-intent integrations."
updatedAt: "2026-05-27"
---

Use this runbook when you need to prove an RFQ integration works end to end on testnet. Complete the checks in order. Do not continue past a failed check; downstream failures usually become noisier versions of the same setup issue.

The commands assume the [`injective-rfq-toolkit`](https://github.com/InjectiveLabs/injective-rfq-toolkit) reference client. That repo is the ground truth for the Python examples used by these docs.

---

## 0. Before you start

<Warning>
This flow is EIP-712 v2 only. Quotes must carry `sign_mode="v2"` and `evm_chain_id=1439` on testnet. TakerStream conditional orders must carry `conditional_order_sign_mode="v2"` and `conditional_order_evm_chain_id=1439`. The quote `chain_id` / `chainId` remains the Cosmos chain ID, `injective-888`; do not put `1439` in `quote.chain_id`.
</Warning>

<Info>
MakerStream requires an auth challenge response before RFQ requests are forwarded. Configure `MakerStreamClient` with `auth_private_key`, `auth_evm_chain_id`, and `auth_contract_address`. If the stream connects and pongs but never receives requests, debug the auth challenge first.
</Info>

You need:

| Requirement | Details |
| --- | --- |
| Python | 3.11 or newer |
| Wallets | One maker wallet and one taker wallet; low-level TP/SL executor tests may also use an executor wallet |
| Gas | Testnet INJ in every wallet that broadcasts transactions |
| Margin | Testnet USDC in the relevant exchange subaccounts |
| Maker status | Maker address whitelisted on the RFQ contract |
| Authz | Maker and taker grants to the current RFQ contract |

---

## 1. Install the reference client

```bash
git clone https://github.com/InjectiveLabs/injective-rfq-toolkit.git
cd injective-rfq-toolkit
python3 -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -e ".[dev]"
```

If you already have the repo locally, pull the latest main branch before testing.

---

## 2. Configure `.env`

Create `injective-rfq-toolkit/.env`:

```bash
RFQ_ENV=testnet

TESTNET_MM_PRIVATE_KEY=<hex>
TESTNET_RETAIL_PRIVATE_KEY=<hex>
# TESTNET_RELAYER_PRIVATE_KEY=<hex> # only for low-level signed-intent executor tests
# TESTNET_ADMIN_PRIVATE_KEY=<hex>   # only for register_maker admin flows
```

Load `.env` before running standalone scripts:

```bash
set -a
. ./.env
set +a
```

Derive an Injective address from a hex private key:

```python
from rfq_test.crypto.wallet import Wallet

print(Wallet.from_private_key("<hex>").inj_address)
```

---

## 3. Verify wallet and subaccount balances

Each active wallet needs:

- INJ for gas in the bank balance.
- USDC margin in the exchange subaccount used by RFQ settlement.

Bank balance query:

```bash
curl -s \
  "https://testnet.sentry.lcd.injective.network/cosmos/bank/v1beta1/balances/<inj_address>" \
  | python3 -m json.tool
```

Look for:

- `inj`
- `erc20:0x0C382e685bbeeFE5d3d9C29e29E341fEE8E84C5d` for testnet USDC

For exchange subaccount balances, use the balance helpers in `injective-rfq-toolkit` or query the Injective exchange module directly. The maker subaccount must match the registered `maker_subaccount_nonce`; if `list_makers` returns `null`, fund and quote with nonce `0`.

---

## 4. Verify maker whitelist

Makers must be registered before MakerStream routes RFQ requests to them and before quotes can settle.

The raw contract query checks only the first page:

```bash
curl -s \
  "https://testnet.sentry.lcd.injective.network/cosmwasm/wasm/v1/contract/inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk/smart/$(echo -n '{\"list_makers\":{}}' | base64)" \
  | python3 -m json.tool
```

<Warning>
`list_makers` is paginated. The first page can omit a registered maker if the address sorts after the first 20 results. Prefer the toolkit helper below when checking your own address.
</Warning>

```python
import asyncio
import os

os.environ["RFQ_ENV"] = "testnet"

from rfq_test.config import get_environment_config
from rfq_test.clients.contract import ContractClient

async def check():
    env = get_environment_config()
    contract = ContractClient(env.contract, env.chain)
    ok = await contract.is_maker_registered("<MM_ADDR>")
    print("whitelisted" if ok else "NOT whitelisted")

asyncio.run(check())
```

If the maker is not registered, follow [Maker whitelist](/sdk-trading/maker-whitelist).

---

## 5. Verify and grant authz

Check maker grants:

```bash
curl -s \
  "https://testnet.sentry.lcd.injective.network/cosmos/authz/v1beta1/grants?granter=<MM_ADDR>&grantee=inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk" \
  | python3 -m json.tool
```

Check taker grants:

```bash
curl -s \
  "https://testnet.sentry.lcd.injective.network/cosmos/authz/v1beta1/grants?granter=<TAKER_ADDR>&grantee=inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk" \
  | python3 -m json.tool
```

Expected grants:

| Role | Required message types |
| --- | --- |
| Maker | `MsgPrivilegedExecuteContract`, `MsgSend` |
| Taker | `MsgPrivilegedExecuteContract`, `MsgBatchUpdateOrders`, `MsgSend` |

If either response is empty or missing a required grant, run the script in [Authorization setup](/sdk-trading/authz). Submit grants sequentially and wait between transactions; parallel grant broadcasts commonly fail with account sequence errors.

---

## 6. Run smoke checks

This smoke verifies config loading, indexer reachability, chain reachability, and EIP-712 v2 quote signing. It does not fully prove the MakerStream challenge flow; the full E2E test does that.

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

maker = Wallet.from_private_key(os.environ["TESTNET_MM_PRIVATE_KEY"])
taker = Wallet.from_private_key(os.environ["TESTNET_RETAIL_PRIVATE_KEY"])

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
    expiry_ms=int(time.time() * 1000) + 2_000,  # must be at least now + 1500ms
    min_fill_quantity=None,
)
print(f"[4/4] sign_quote_v2 returns signature len={len(sig)}")
```

If WebSocket connectivity works here but the maker receives no requests in the full test, the likely failure is the MakerStream auth challenge, not basic networking.

---

## 7. Run live `AcceptQuote` settlement

Use the reference script first:

```bash
set -a
. ./.env
set +a

python examples/test_settlement.py
```

Native gRPC variant:

```bash
python examples/test_settlement_grpc.py
```

A successful run proves the full path:

1. Taker connects to TakerStream.
2. Maker connects to MakerStream and answers `MakerChallenge`.
3. Taker sends an RFQ request and receives the indexer-assigned `rfq_id`.
4. Maker waits for that same `rfq_id`, signs a quote with EIP-712 v2, and sends it with `sign_mode="v2"`.
5. Taker collects matching quotes.
6. Taker submits `AcceptQuote`.
7. Maker receives quote or settlement updates.

Success should end with a transaction hash. Open it in [testnet explorer](https://testnet.explorer.injective.network) and confirm both parties' derivative positions changed on the selected market.

The maker client must include auth fields:

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

Operational notes:

- `client_id` should be a UUID. The indexer assigns the real `rfq_id` in the ACK.
- MakerStream can broadcast other takers' RFQs. Filter by the ACK-returned `rfq_id`.
- `quote_ack` means "accepted by the indexer", not "filled by the taker".
- Live quote expiry is intentionally short but must be at least 1500 ms. Longer expiries improve match odds, but increase stale-price exposure.

---

## 8. Run TP/SL signed-intent validation

TP/SL exits are taker-signed conditional orders. The taker signs a `SignedTakerIntent`, submits it through TakerStream, and the TP/SL executor executes `AcceptSignedIntent` when the trigger condition is satisfied.

Before testing:

- The taker has an open position to close.
- The executor wallet has INJ for gas if you are testing low-level executor submission.
- The executor can source a normal RFQ quote when the trigger fires. No maker-specific TP/SL setup is required beyond the standard MakerStream quote loop.

Minimum signing and submission shape:

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

The helper sets TakerStream wire fields `conditional_order_sign_mode="v2"` and `conditional_order_evm_chain_id` when `sign_mode` and `evm_chain_id` are passed.

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

Use `CancelAllIntents` only when you need to invalidate every lane for the taker. Future intents must use the incremented `epoch` or `lane_version`.

---

## Common blockers

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Maker stream connects but receives no requests | Maker did not answer `MakerChallenge`, or answered with the wrong domain | Configure `auth_private_key`, `auth_evm_chain_id`, and `auth_contract_address`; verify challenge signing against `StreamAuthChallenge` |
| `authorization not found` or `unauthorized` | Missing or expired authz grant | Re-run [Authorization setup](/sdk-trading/authz) and re-query grants |
| Quote rejected for signature | Signed decimal strings differ from sent strings, wrong EVM chain ID, wrong contract domain, or wrong field order | Quantize before signing and send the exact signed values |
| No quotes collected by taker | Maker not whitelisted, offline, filtering wrong `rfq_id`, or request was not ACKed | Verify whitelist and correlate from the ACK-returned `rfq_id` |
| `No quote was filled` | Expired quote, failed `worst_price`, mark-band rejection, wrong maker subaccount nonce, or margin issue | Inspect quote results, `list_makers`, and subaccount balances |
| Conditional order accepted but never fires | Trigger not reached, expired intent, executor unavailable, or no executable RFQ quote | Check intent status, trigger price, deadline, and executor logs |
| `rfq_id mismatch` | Executor paired a signed intent with a quote for another RFQ | Re-request RFQ liquidity for the exact signed-intent `rfq_id` |

For field-level details, see [Protocol reference](/sdk-trading/protocol-reference) and [Troubleshooting](/sdk-trading/troubleshooting).
