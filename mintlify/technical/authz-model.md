---
title: "Authorization model"
description: "Detailed explanation of Injective's authz authorization model enabling TrueCurrent's smart contract to execute atomic settlements with granular message-type permissions and revocable grants."
updatedAt: "2026-04-06"
---

TrueCurrent uses Injective's native `authz` module to allow the smart contract to act on behalf of both traders and market makers during settlement. This page explains the model in detail.

---

## Overview

When you accept a quote on TrueCurrent, the onchain settlement involves the TrueCurrent contract calling Injective's exchange module to open positions for both you and the market maker. Rather than requiring both parties to sign separate transactions at settlement time, TrueCurrent uses pre-granted authorizations so the contract can bundle everything into a single atomic transaction.

```mermaid
flowchart LR
    subgraph Granters["Granters (one-time setup)"]
        Taker["Taker wallet"]
        Maker["Market Maker wallet"]
    end

    subgraph Grantee["Grantee"]
        Contract["TrueCurrent Contract"]
    end

    subgraph Chain["Injective Chain modules"]
        Exchange["Exchange Module<br/>open positions, orderbook"]
        Bank["Bank Module<br/>move funds"]
    end

    Taker -.->|"authz grants<br/>(specific msg types only)"| Contract
    Maker -.->|"authz grants<br/>(specific msg types only)"| Contract

    Contract -->|MsgPrivilegedExecuteContract| Exchange
    Contract -->|MsgBatchUpdateOrders<br/>(unfilled fallback)| Exchange
    Contract -->|MsgSend<br/>(margin moves)| Bank

    classDef wallet fill:#fff4e8,stroke:#c88a40,color:#111
    classDef contract fill:#eef4ff,stroke:#4a6db8,color:#111
    classDef module fill:#eefbf0,stroke:#3c9c60,color:#111
    class Taker,Maker wallet
    class Contract contract
    class Exchange,Bank module
```

The dotted lines are one-time grants. The solid lines are what the contract can do *because of* those grants — scoped to the specific message types you authorized.

This is secure because:

1. **Grants are message-type specific.** You're only authorizing specific message types (`MsgPrivilegedExecuteContract`, `MsgBatchUpdateOrders`, `MsgSend`) – not blanket wallet access.
2. **Grants are scoped to the contract address.** The authorization only applies to the TrueCurrent contract, not any other address.
3. **Grants are revocable.** You can revoke any grant at any time, immediately stopping all future contract-initiated actions.
4. **Onchain verification.** The Injective chain enforces that only the authorized grantee can submit authorized messages – this is not just convention, it's chain-level enforcement.

---

{/* TODO: authz grant lists on this page need reconciliation against rfq-contract source. See the TODO on /takers/authz-setup for the specific questions (MsgWithdraw for takers, MsgPrivilegedExecuteContract scope, MsgBatchUpdateOrders vs MsgCreateDerivativeMarketOrder). */}

## Required grants for traders

| Message type | Why it's needed |
|---|---|
| `/injective.exchange.v2.MsgPrivilegedExecuteContract` | Allows the contract to open and manage your exchange positions |
| `/injective.exchange.v2.MsgBatchUpdateOrders` | Allows the contract to interact with the order book on your behalf (for fallback OB routing) |
| `/cosmos.bank.v1beta1.MsgSend` | Allows the contract to move margin between your wallet and subaccount as needed for settlement |

---

## Required grants for market makers

| Message type | Why it's needed |
|---|---|
| `/injective.exchange.v2.MsgPrivilegedExecuteContract` | Same as traders – the contract opens the maker's opposing position |
| `/cosmos.bank.v1beta1.MsgSend` | Margin management |

---

## Grant lifecycle

**Initial setup:** Grants are submitted as transactions on Injective. Each grant type requires a separate transaction. After setup, grants persist indefinitely unless explicitly revoked or they reach an expiry date you set.

**During trading:** Every time a trade settles, the TrueCurrent contract uses your pre-granted permissions to execute the settlement messages. No further action is needed from you.

**Revoking:** Submit a revoke transaction for any specific grant. The revocation takes effect immediately – the contract can no longer submit that message type on your behalf.

---

## Viewing your active grants

You can query your active authz grants on Injective using:

```bash
injectived query authz grants [granter_address] [grantee_address] --node https://sentry.tm.injective.network:443
```

Or via the Injective REST API:

```
GET /cosmos/authz/v1beta1/grants?granter={your_address}&grantee={contract_address}
```

---

## Why not just use allowances or signatures per trade?

Alternative designs exist – for example, requiring the trader to sign each `AcceptQuote` transaction manually. TrueCurrent uses `authz` instead for UX reasons: waiting for a second user signature during the 30-second quote expiry window introduces friction and increases the chance the quote expires before settlement. The `authz` model allows instant, one-click settlement.

The security tradeoff is the pre-granted permission. TrueCurrent mitigates this by keeping grants narrow (specific message types only) and building the onchain contract checks (worst price, quote signature, expiry) as the actual security layer – the contract enforces your trading parameters regardless of its grant.
