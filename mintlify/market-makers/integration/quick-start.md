---
title: "Quick Start"
description: "5 minute quick start. Prerequisites, environment setup, one-time registration, run the market maker bot"
updatedAt: "2026-04-17"
---

## Prerequisites

| Requirement | Version | Notes |
| --- | --- | --- |
| Node.js | 18+ | For TypeScript examples |
| Python | 3.10+ | For Python examples (`pip install injective-py websockets eth-account`) |
| Go | 1.21+ | For Go examples (uses `InjectiveLabs/sdk-go`) |
| Private Key |  | An Injective-compatible secp256k1 private key (hexadecimal) |
| Testnet INJ |  | Get from https://testnet.faucet.injective.network/ |

## Environment setup

Create a `.env` file with the following variables.
The MM private key is used only for signing quotes off-chain.
It never leaves your machine.

```shell
MM_PRIVATE_KEY={your-private-key-hexadecimal}
ADMIN_PRIVATE_KEY={admin-key-for-registration}
CONTRACT_ADDRESS=inj1t8hyyle68vd0kzsdehxg0sywttrwmt58jzk29q
CHAIN_ID=injective-888
NETWORK=testnet
```

<Info>
**Important: Key Derivation**

Your Injective address (`inj1...`) is derived from your private key,
using EVM-style secp256k1 cryptography with bech32 encoding.
The same private key produces both an EVM address (`0x...`) and an Injective address (`inj1...`).
Use the SDK's `PrivateKey.fromHex().toBech32()` method.

Read more: [Convert addresses](https://docs.injective.network/developers/convert-addresses)
</Info>

## Setup

Before sending quotes, two on-chain transactions are required.
This is a one-time setup:

**Step 1** - MM grants authz permissions to the RFQ contract:

The MM wallet grants the contract permission to execute
`/injective.exchange.v2.MsgPrivilegedExecuteContract` and
`/cosmos.bank.v1beta1.MsgSend` on its behalf.
This allows the contract to settle trades and transfer funds.

**Step 2**  - Admin registers MM on the contract:

The contract admin calls `register_maker` with the MM's Injective address.
Only whitelisted MMs can have their quotes accepted.

```ts
# TypeScript
npm install
npm run ts-mm-setup
```

```py
# Python
pip install injective-py eth-keys websockets eth-account python-dotenv
python3 mm-scripts-python/setup.py
```

```go
# Go
cd mm-scripts-go && go run setup/setup.go
```

## Run the MM bot

Once registered, start the MM bot. It connects to the indexer via WebSocket, subscribes to the RFQ request
stream, and automatically sends signed quotes for every incoming request.

```ts
# TypeScript (reference implementation)
npm run ts-mm
```

```py
# Python
python3 mm-scripts-python/main.py
```

```go
# Go
cd mm-scripts-go && go run main/main.go
```

## Bot live!

Your MM bot is now live!
It will receive RFQ requests from retail users in real-time,
generate quotes with your pricing logic,
sign them with your private key,
and send them back through the WebSocket.
The retail client picks the best quote and settles on-chain.
