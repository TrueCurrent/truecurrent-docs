---
title: "Testnet configuration"
description: "Complete testnet configuration reference for TrueCurrent market maker integration: chain settings, RFQ indexer endpoints, contract address, faucet, and environment variables."
updatedAt: "2026-05-05"
---

```yaml
# Chain
chain_id:      "injective-888"
grpc_endpoint: "testnet-grpc.injective.dev:443"
lcd_endpoint:  "https://testnet.sentry.lcd.injective.network"

# RFQ Indexer — base WebSocket URL (client appends /MakerStream or /TakerStream)
ws_endpoint:       "wss://testnet.rfq.ws.injective.network"
grpc_endpoint:     "testnet.rfq.grpc.injective.network:443"
grpc_web_endpoint: "https://testnet.rfq.grpc.injective.network/injective_rfq_rpc.InjectiveRfqRPC"
http_endpoint:     "https://testnet.rfq.injective.network"

# RFQ Contract
contract_address: "inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk"  # 0.1.0-alpha.6

# EIP-712
evm_chain_id: 1439   # testnet (mainnet: 1776)

# Tokens
tokens:
  usdc: "erc20:0x0C382e685bbeeFE5d3d9C29e29E341fEE8E84C5d"

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
  The `rfq-testing` client appends `/MakerStream` (makers) or `/TakerStream` (takers) automatically.
  Do not include the service path in this value.
- The USDC denom `erc20:0x0C382e685bbeeFE5d3d9C29e29E341fEE8E84C5d` is used when
  funding your exchange subaccount and checking balances via the chain REST API.
- Use `https://testnet.explorer.injective.network` to verify settlement transactions by tx hash.
