---
title: "Testnet configuration"
description: "Endpoint, contract, token, EIP-712, and market configuration reference for TrueCurrent RFQ SDK integrations on testnet."
updatedAt: "2026-05-27"
---

Use this page to configure clients and sanity-check `.env` values. Use the [Runbook](/sdk-trading/runbook) for procedural validation.

---

## Chain and contract

| Setting | Value |
| --- | --- |
| Cosmos chain ID | `injective-888` |
| EIP-712 chain ID | `1439` |
| Chain gRPC | `testnet-grpc.injective.dev:443` |
| Chain LCD | `https://testnet.sentry.lcd.injective.network` |
| RFQ contract | `inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk` |
| Contract version | `0.1.0-alpha.6` |
| Explorer | `https://testnet.explorer.injective.network` |

<Warning>
`injective-888` and `1439` are both required, but they are not interchangeable. `quote.chain_id` is the Cosmos chain ID and must be `injective-888` on testnet. `quote.evm_chain_id` and the EIP-712 domain `chainId` use `1439`. Raw MakerStream quote payloads use snake_case field names; passing `1439` as `quote.chain_id` is a known integration failure mode.
</Warning>

| Context | Testnet value |
| --- | --- |
| EIP-712 domain `chainId` | `1439` |
| Quote payload `evm_chain_id` | `1439` |
| Quote payload `chain_id` | `injective-888` |

---

## RFQ indexer endpoints

| Service | Endpoint |
| --- | --- |
| WebSocket base | `wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC` |
| MakerStream WSS | `wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC/MakerStream` |
| TakerStream WSS | `wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC/TakerStream` |
| gRPC-web base | `https://testnet.rfq.grpc.injective.network/injective_rfq_rpc.InjectiveRfqRPC` |
| Native gRPC | `testnet.rfq.grpc.injective.network:443` |
| HTTP | `https://testnet.rfq.injective.network` |

The public service alias is `injective_rfq_rpc.InjectiveRfqRPC`. The deployed paths use underscores and `InjectiveRfqRPC`; do not substitute the internal proto package name.

Most SDK clients store the WebSocket base endpoint and append `/MakerStream` or `/TakerStream` internally. Do not append the stream path twice.

---

## Tokens

| Asset | Denom | Use |
| --- | --- | --- |
| INJ | `inj` | Gas |
| USDC | `erc20:0x0C382e685bbeeFE5d3d9C29e29E341fEE8E84C5d` | RFQ margin |

Fund both the bank balance for gas and the exchange subaccount for margin. Maker margin must be available in the subaccount nonce registered for that maker.

Faucet: `https://testnet-faucet.injective.dev`

---

## Markets

| Symbol | Market ID | Price tick |
| --- | --- | --- |
| `INJ/USDC PERP` | `0xdc70164d7120529c3cd84278c98df4151210c0447a65a2aab03459cf328de41e` | `0.01` |
| `BTC/USDC PERP` | `0xfd704649cf3a516c0c145ab0111717c44640d8dbe52a462ae35cadf2f6df1515` | `1` |
| `LINK/USDC PERP` | `0xdbb9bb072015238096f6e821ee9aab7affd741f8662a71acc14ac30ee6b687a5` | `0.001` |
| `ETH/USDC PERP` | `0x135de28700392fb1c17d40d5170a74f30055a4ad522feddafec42fbbbb780897` | `0.1` |

Snap prices and quantities to the market rules before signing. The signature covers the exact decimal strings you send.

---

## Environment variables

Minimum `.env` for local tests:

```bash
RFQ_ENV=testnet

TESTNET_MM_PRIVATE_KEY=<hex>
TESTNET_RETAIL_PRIVATE_KEY=<hex>
```

Optional variables:

```bash
TESTNET_RELAYER_PRIVATE_KEY=<hex> # signed-intent relay tests
TESTNET_ADMIN_PRIVATE_KEY=<hex>   # maker registration admin flows
```

Manual client configuration:

```yaml
chain:
  chain_id: injective-888
  grpc_endpoint: testnet-grpc.injective.dev:443
  lcd_endpoint: https://testnet.sentry.lcd.injective.network

contract:
  address: inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk
  version: 0.1.0-alpha.6

signing:
  evm_chain_id: 1439

indexer:
  ws_endpoint: wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC
  grpc_web_endpoint: https://testnet.rfq.grpc.injective.network/injective_rfq_rpc.InjectiveRfqRPC
  http_endpoint: https://testnet.rfq.injective.network
  grpc_endpoint: testnet.rfq.grpc.injective.network:443

tokens:
  usdc: erc20:0x0C382e685bbeeFE5d3d9C29e29E341fEE8E84C5d
```

---

## Mainnet signing contrast

Mainnet uses Cosmos chain ID `injective-1` and EIP-712 chain ID `1776`. Pull both values from environment config instead of hardcoding when writing reusable clients.
