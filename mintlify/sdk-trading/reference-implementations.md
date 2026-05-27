---
title: "Reference implementations"
description: "Python, TypeScript, and Go reference implementation guide for TrueCurrent RFQ maker and taker integrations."
updatedAt: "2026-05-27"
---

All reference implementations live in [`InjectiveLabs/injective-rfq-toolkit`](https://github.com/InjectiveLabs/injective-rfq-toolkit). Start there before copying snippets from the docs into a production service.

```bash
git clone https://github.com/InjectiveLabs/injective-rfq-toolkit.git
cd injective-rfq-toolkit
python3 -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -e ".[dev]"
```

---

## Which example to use

| Goal | Entry point | Notes |
| --- | --- | --- |
| Prove full WebSocket RFQ settlement | `examples/test_settlement.py` | Canonical E2E test: TakerStream request, MakerStream auth, quote, `AcceptQuote`. |
| Prove full native gRPC RFQ settlement | `examples/test_settlement_grpc.py` | Same path using native gRPC transport. |
| Run a reusable Python WebSocket maker | `examples/python-mm/mark_quote_loop.py` | Quotes mark +/- edge bps, supports fixed-price tests, handles MakerStream auth, logs quote and settlement updates. |
| Inspect the simpler Python maker loop | `examples/python-mm/main.py` | Useful for understanding the WebSocket client shape. |
| Run a Python native gRPC maker | `examples/python-mm/main-grpc.py` | Native gRPC maker reference. |
| Run a TypeScript native gRPC maker | `examples/ts-mm/main-grpc.ts` | Uses `ethers` signing helpers for EIP-712 v2. |
| Run a Go native gRPC maker | `examples/go-mm/main-grpc/main.go` | Go transport and quote loop reference. |

---

## Python WebSocket / gRPC-web

| Area | Status | Implementation detail |
| --- | --- | --- |
| Transport | Production-compatible | gRPC-web framed over WebSocket using the public `injective_rfq_rpc.InjectiveRfqRPC` service path. |
| MakerStream auth | Implemented | `MakerStreamClient` signs `MakerChallenge` when configured with `auth_private_key`, `auth_evm_chain_id`, and `auth_contract_address`. |
| Quote signing | Implemented | `rfq_test.crypto.eip712.sign_quote_v2` signs the exact `SignQuote` typed-data schema. |
| Signed intents | Implemented | `sign_conditional_order_v2` plus TakerStream `sign_mode="v2"` and `evm_chain_id`. |
| Ping/pong | Implemented | Client handles periodic stream liveness. |

Recommended validation command:

```bash
set -a
. ./.env
set +a
python examples/test_settlement.py
```

---

## Python native gRPC

Use native gRPC when your runtime can keep long-lived gRPC streams open cleanly and you do not need WebSocket/gRPC-web framing.

```bash
python examples/test_settlement_grpc.py
```

Maker entry point:

```bash
python examples/python-mm/main-grpc.py
```

---

## TypeScript

The TypeScript reference focuses on native gRPC transport and EIP-712 v2 signing with `ethers`.

```bash
npx tsx examples/ts-mm/main-grpc.ts
```

Use it as a signing and stream-shape reference. The Python toolkit remains the most complete executable harness for full testnet validation.

---

## Go

The Go reference demonstrates native gRPC maker connectivity and quote submission.

```bash
go run ./examples/go-mm/main-grpc
```

Use the Go example for transport and protobuf wiring, then cross-check signature fields against [Building and signing quotes](/sdk-trading/signing-quotes).

---

## What to copy into production

Copy patterns, not test constants:

- Load chain ID, EIP-712 chain ID, contract address, and endpoints from config.
- Reuse the `SignQuote` field order exactly.
- Send the exact decimal strings that were signed.
- Answer the MakerStream auth challenge before expecting RFQ requests.
- Treat `quote_ack` as routing confirmation, not a fill.
- Filter maker requests and taker quote collection by the ACK-returned `rfq_id`.
- Log `rfq_id`, `client_id`, quote expiry, maker subaccount nonce, quote ACKs, and settlement updates.

For endpoint values, see [Testnet configuration](/sdk-trading/testnet-configuration). For operational validation, see [Runbook](/sdk-trading/runbook).
