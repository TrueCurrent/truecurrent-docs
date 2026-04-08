---
title: "Architecture"
description: "System architecture overview of TrueCurrent's three-layer design with offchain RFQ indexer, CosmWasm smart contract for verification, and Injective exchange module for settlement."
updatedAt: "2026-04-06"
---

TrueCurrent is composed of three main layers: an offchain quote distribution layer, an onchain smart contract, and Injective's native exchange module for final settlement.

---

## System overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         TrueCurrent                             │
│                                                                 │
│  ┌────────────┐    ┌──────────────────┐    ┌─────────────────┐  │
│  │   Trader   │    │   RFQ Indexer    │    │  Market Maker   │  │
│  │  (Taker)   │    │  (Off-chain)     │    │  (Off-chain)    │  │
│  └──────┬─────┘    └────────┬─────────┘    └────────┬────────┘  │
│         │                  │                        │           │
│         │  RFQ Request     │                        │           │
│         ├─────────────────►│                        │           │
│         │                  │  Broadcast to MMs      │           │
│         │                  ├───────────────────────►│           │
│         │                  │                        │           │
│         │                  │  Signed Quote          │           │
│         │                  │◄───────────────────────┤           │
│         │  Best Quote      │                        │           │
│         │◄─────────────────┤                        │           │
│         │                  │                        │           │
│         │  AcceptQuote tx  │                        │           │
│         ├──────────────────┼────────────────────────┼──────────►│
│         │                  │            Injective Chain          │
│         │                  │       ┌──────────────────────────┐  │
│         │                  │       │  TrueCurrent Contract    │  │
│         │                  │       │  - Verify MM signature   │  │
│         │                  │       │  - Check worst_price     │  │
│         │                  │       │  - Check expiry          │  │
│         │                  │       │  - Settle via exchange   │  │
│         │                  │       │    module (authz)        │  │
│         │                  │       └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Components

### Takers

"Takers" is an umbrella term for anyone initiating a trade — both end users clicking through the TrueCurrent UI and programmatic clients (arbitrage bots, HFT systems, integrators). Both populations hit the same TakerStream and settle through the same contract.

- End users: see [How to trade](/trading/how-to-trade) for the UI walkthrough
- Programmatic takers: see [Takers: overview](/takers/overview) for the developer track

### RFQ Indexer (offchain)

The indexer is TrueCurrent's offchain coordination layer. It:

- Maintains the registry of whitelisted market maker addresses
- Operates the **TakerStream** WebSocket — see [Takers: TakerStream](/takers/taker-stream)
- Operates the **MakerStream** WebSocket — see [Market makers: MakerStream](/market-makers/maker-stream)
- Routes requests to all active market makers
- Collects and forwards quotes back to takers
- Selects the best quote for presentation

The indexer is a coordination layer only – it never holds funds or executes trades. Its role is purely informational: passing messages between takers and market makers. Even if the indexer were to behave maliciously, it could not forge a market maker's signature or alter the terms of a quote.

### TrueCurrent smart contract (onchain)

Deployed on Injective as a CosmWasm contract, the TrueCurrent contract is the trust anchor of the system. It:

- Verifies market maker signatures on each `AcceptQuote` call
- Enforces the trader's `worst_price` constraint
- Checks quote expiry
- Confirms both parties have sufficient margin
- Executes the settlement through Injective's exchange module using pre-granted `authz` permissions
- Routes any unfilled quantity to the Injective order book

The contract's logic is deterministic and publicly verifiable. All settlement decisions are made onchain.

### Injective exchange module

Injective has a native exchange module built into the chain consensus layer – not a smart contract, but a chain-level primitive. The TrueCurrent contract uses `MsgPrivilegedExecuteContract` to call into this module for final position settlement.

The exchange module handles:
- Margin accounting and position tracking
- Liquidation engine
- Funding rate calculations and payments
- Onchain order book (used for fallback fills)

---

## Data flow: full trade lifecycle

1. **Trader → Indexer (TakerStream):** Trader sends an RFQ request over WebSocket
2. **Indexer → All MMs (MakerStream):** Request is broadcast to all active makers simultaneously
3. **MMs → Indexer (MakerStream):** Each maker responds with a signed quote within a few hundred milliseconds {/* TODO: CK to add precise MM response deadline once benchmarked */}
4. **Indexer → Trader (TakerStream):** Best quote is returned to the trader
5. **Trader → Chain (AcceptQuote):** Trader submits the `AcceptQuote` transaction with the quote and their parameters
6. **Contract verification:** Onchain contract verifies signature, price, expiry, and margin
7. **Settlement (exchange module):** Contract uses `authz` to open positions for both taker and maker through Injective's exchange module
8. **Position update:** Both wallets' subaccounts reflect the new positions

---

## Trust model

**Who you trust when trading on TrueCurrent:**

- **The TrueCurrent smart contract** – open source, onchain, deterministic. Verifiable by anyone.
- **Injective chain and validators** – for block finality and execution of the exchange module.
- **The RFQ indexer** – only for routing. It cannot steal funds or alter prices. At worst, a malicious indexer could drop requests (degraded service), but couldn't cause unauthorized trades.

**What you do not need to trust:**

- Individual market makers to honor prices – the signature enforces it onchain
- TrueCurrent to hold or secure your funds – assets stay in your subaccount at all times
- Centralized infrastructure for settlement – all final settlement is onchain
