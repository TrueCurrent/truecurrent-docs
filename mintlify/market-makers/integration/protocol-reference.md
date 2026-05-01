---
title: "Protocol reference"
description: "Full protocol reference for TrueCurrent's RFQ system: sequence diagrams for the standard AcceptQuote flow and the TP/SL AcceptSignedIntent path, plus supported testnet markets."
updatedAt: "2026-05-01"
---

### Full lifecycle (sync `AcceptQuote` path)

```mermaid
sequenceDiagram
    participant MM as 🏦 Market Maker
    participant IDX as 📡 RFQ Indexer
    participant Taker as 👤 Retail User
    participant Chain as ⛓️ Injective Chain

    Note over MM: ONE-TIME SETUP
    MM->>Chain: Grant Authz to Contract
    Chain-->>MM: ✅ Granted

    Note over MM: CONNECT
    MM->>IDX: WebSocket Connect (MakerStream)
    IDX-->>MM: Connected
    loop Every 1 second
        MM->>IDX: Ping
        IDX-->>MM: Pong
    end

    Note over Taker,MM: TRADE FLOW
    Taker->>IDX: RFQ Request with client_id (TakerStream)
    IDX-->>Taker: Request ACK with rfq_id
    IDX->>MM: Broadcast Request
    MM->>MM: Calculate Price
    MM->>MM: SignQuote EIP-712 v2 digest
    MM->>MM: secp256k1 sign
    MM->>IDX: Send Signed Quote with sign_mode="v2"
    IDX-->>MM: Quote ACK ✅
    IDX->>Taker: Deliver Quote(s)
    Taker->>Chain: AcceptQuote(quote, signature)
    Chain->>Chain: Verify MM Signature ✅
    Chain->>Chain: Settle Synthetic Trade
    Note over MM,Chain: Positions opened for both parties
```

### TP/SL path (`AcceptSignedIntent`)

{/* TODO: add detailed TPSL flow doc */}

Summary from your side:

```mermaid
sequenceDiagram
    participant Taker as 👤 Taker (offline)
    participant IDX as 📡 Indexer / Relayer
    participant MM as 🏦 Market Maker
    participant Chain as ⛓️ Chain

    Taker->>IDX: Submit SignedTakerIntent (TP/SL, pre-signed)
    IDX->>IDX: Persist, monitor mark price

    IDX->>MM: Live RFQ for exit size
    MM->>IDX: Signed quote with sign_mode="v2"

    IDX->>Chain: Triggered settlement
    Chain->>Chain: Verify taker sig + trigger + maker sig
    Chain->>Chain: Settle synthetic trade
```

### Supported markets (testnet)

| Market | Symbol | Market ID |
|---|---|---|
| INJ/USDC Perp | `INJ/USDC PERP` | `0xdc70164d7120529c3cd84278c98df4151210c0447a65a2aab03459cf328de41e` |
| BTC/USDC Perp | `BTC/USDC PERP` | `0xfd704649cf3a516c0c145ab0111717c44640d8dbe52a462ae35cadf2f6df1515` |
| LINK/USDC Perp | `LINK/USDC PERP` | `0xdbb9bb072015238096f6e821ee9aab7affd741f8662a71acc14ac30ee6b687a5` |
| ETH/USDC Perp | `ETH/USDC PERP` | `0x135de28700392fb1c17d40d5170a74f30055a4ad522feddafec42fbbbb780897` |
