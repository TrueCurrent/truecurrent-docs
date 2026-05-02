---
title: "Testnet configuration"
description: "Complete testnet configuration reference for TrueCurrent market maker integration: chain settings, RFQ indexer endpoints, contract address, faucet, and environment variables."
updatedAt: "2026-04-18"
---

```yaml
# Chain
chain_id:      "injective-888"
grpc_endpoint: "testnet-grpc.injective.dev:443"
lcd_endpoint:  "https://testnet.sentry.lcd.injective.network"

# RFQ Indexer (gRPC-web over WebSocket)
ws_endpoint:   "wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC"
#                ↳ append /MakerStream for MM, /TakerStream for takers
grpc_endpoint: "testnet.rfq.grpc.injective.network:443"
grpc_web_endpoint: "https://testnet.rfq.grpc.injective.network/injective_rfq_rpc.InjectiveRfqRPC"
http_endpoint: "https://testnet.rfq.injective.network"

# RFQ Contract
contract_address: "inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk"  # 0.1.0-alpha.6

# Faucet
faucet: "https://testnet-faucet.injective.dev"
```

### Environment variables

```bash
MM_PRIVATE_KEY=<hex>
CHAIN_ID=injective-888
CONTRACT_ADDRESS=inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk
```
