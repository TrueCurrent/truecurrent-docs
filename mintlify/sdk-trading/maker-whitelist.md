---
title: "Maker whitelist"
description: "How TrueCurrent approves and registers makers, what to provide before onboarding, and how to verify maker whitelist status."
updatedAt: "2026-05-27"
---

TrueCurrent uses an approved maker whitelist. Only registered maker addresses receive MakerStream RFQ broadcasts and only registered makers can settle quotes.

---

## Why there is a whitelist

Makers take the opposite side of user trades, so every active maker must be able to quote reliably and settle safely. The whitelist keeps the maker set limited to systems that have demonstrated:

- Stable MakerStream connectivity
- Correct MakerChallenge auth handling
- EIP-712 v2 quote signing
- Sufficient USDC margin
- Reasonable pricing and expiry controls
- Settlement reconciliation
- A tested operational runbook

---

## What to send TrueCurrent

Before registration, provide:

| Item | Notes |
| --- | --- |
| Maker `inj1...` address | The address your quoting system will use on MakerStream and quote signatures |
| Markets | Symbols or market IDs you plan to support |
| Subaccount nonce | Omit or use `0` unless your system intentionally isolates maker collateral in another nonce |
| Testnet status | Whether you have completed the E2E runbook |
| Contact path | Where TrueCurrent can reach your operator during incidents |
| Quoting summary | Short description of price feeds, risk limits, and inventory controls |

Use one maker wallet per operational maker identity. If you rotate keys, the new address must be registered.

---

## Verification flow

After TrueCurrent registers you, confirm the address appears in `list_makers`.

```bash
curl -s \
  "https://testnet.sentry.lcd.injective.network/cosmwasm/wasm/v1/contract/inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk/smart/$(echo -n '{\"list_makers\":{}}' | base64)" \
  | python3 -m json.tool
```

<Warning>
The raw `list_makers` query can be paginated. If your address sorts beyond the first page, a one-line query can look like a false negative. Use the contract helper below when available.
</Warning>

```python
import asyncio
import os

os.environ["RFQ_ENV"] = "testnet"

from rfq_test.config import get_environment_config
from rfq_test.clients.contract import ContractClient

async def main():
    env = get_environment_config()
    client = ContractClient(env.contract, env.chain)
    registered = await client.is_maker_registered("<YOUR_MM_ADDR>")
    print("whitelisted" if registered else "not whitelisted")

asyncio.run(main())
```

When you find your maker record, also record its `subaccount_nonce`. That nonce must match the `maker_subaccount_nonce` you sign and the subaccount you fund.

---

## Readiness checklist

Before asking to move beyond testnet, confirm:

- `list_makers` returns your maker address.
- Maker and taker authz grants are present.
- Maker subaccount has enough USDC for quoted margin.
- MakerStream receives `request` events after answering `MakerChallenge`.
- The maker returns quotes with `sign_mode: "v2"` and the correct `evm_chain_id`.
- Decimal strings are canonical and tick-quantized.
- The E2E settlement script lands a transaction.
- You can stop quoting quickly if price feeds, balances, or hedging fail.

See [Runbook](/sdk-trading/runbook) for the command sequence.
