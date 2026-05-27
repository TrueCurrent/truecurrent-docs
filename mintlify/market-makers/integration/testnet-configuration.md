---
title: "Testnet configuration"
description: "Complete testnet configuration reference for TrueCurrent maker integration: chain settings, RFQ indexer endpoints, contract address, faucet, and environment variables."
updatedAt: "2026-05-05"
---

```yaml
# Chain
chain_id:      "injective-888"
grpc_endpoint: "testnet-grpc.injective.dev:443"
lcd_endpoint:  "https://testnet.sentry.lcd.injective.network"
```

```yaml
# RFQ Indexer
ws_endpoint:       "wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC"
grpc_web_endpoint: "https://testnet.rfq.grpc.injective.network/injective_rfq_rpc.InjectiveRfqRPC"
#                  append /MakerStream for makers or /TakerStream for takers
```

```yaml
# RFQ Contract
contract_address: "inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk"  # 0.1.0-alpha.6
```

```yaml
# EIP-712
evm_chain_id: 1439
```

<Warning>
Testnet uses both `chain_id: "injective-888"` and `evm_chain_id: 1439`. `chain_id` / `quote.chainId` is the Cosmos chain ID. `evm_chain_id` and EIP-712 domain `chainId` are the numeric EVM chain ID. Do not send `1439` as `quote.chain_id`.
</Warning>

```yaml
# Tokens
tokens:
  usdc: "erc20:0x0C382e685bbeeFE5d3d9C29e29E341fEE8E84C5d"
```

```yaml
# Testnet utilities
faucet:   "https://testnet-faucet.injective.dev"  # 500 USDC + 2 INJ, once per 24h per address
explorer: "https://testnet.explorer.injective.network"
```

### Environment variables

```bash
MM_PRIVATE_KEY=<hex>
CHAIN_ID=injective-888
CONTRACT_ADDRESS=inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk
```

### Notes

- `ws_endpoint` is the **base URL only**.
  The `injective-rfq-toolkit` client appends `/MakerStream` (makers) or `/TakerStream` (takers) automatically.
  Do not include the service path in this value.
- The USDC denom `erc20:0x0C382e685bbeeFE5d3d9C29e29E341fEE8E84C5d` is used when
  funding your exchange subaccount and checking balances via the chain REST API.
- Use `https://testnet.explorer.injective.network` to verify settlement transactions by tx hash.
