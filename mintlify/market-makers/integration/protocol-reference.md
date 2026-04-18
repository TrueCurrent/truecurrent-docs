---
title: "Protocol reference"
description: "Full protocol reference for TrueCurrent's RFQ system: sequence diagrams for the standard AcceptQuote flow and the TP/SL AcceptSignedIntent path, plus supported testnet markets."
updatedAt: "2026-04-18"
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
    Taker->>IDX: RFQ Request (TakerStream)
    IDX->>MM: Broadcast Request
    MM->>MM: Calculate Price
    MM->>MM: Build SignQuote JSON
    MM->>MM: keccak256 → secp256k1 sign
    MM->>IDX: Send Signed Quote
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

    alt Trigger fires (Blind mode)
        Note right of IDX: Pick pre-posted blind quote
        MM-->>IDX: (quote already in blind book)
    else Trigger fires (Taker-specific mode)
        IDX->>MM: Live RFQ for exit size
        MM->>IDX: Signed quote
    end

    IDX->>Chain: AcceptSignedIntent(intent, sig, quotes, quote_rfq_id?)
    Chain->>Chain: Verify taker sig + trigger + maker sig
    Chain->>Chain: Settle synthetic trade
```

### Supported markets (testnet)

| Market | Symbol | Market ID |
|---|---|---|
| INJ/USDT Perp | `INJ/USDT PERP` | `0x17ef48032cb24375ba7c2e39f384e56433bcab20cbee9a7357e4cba2eb00abe6` |
| ATOM/USDT Perp | `ATOM/USDT PERP` | `0xd97d0da6f6c11710ef06315971250e4e9aed4b7d4cd02059c9477ec8cf243782` |
