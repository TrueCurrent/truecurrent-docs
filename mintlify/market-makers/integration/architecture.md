---
title: "Architecture overview"
description: "High-level overview of how TrueCurrent's RFQ system connects retail takers, the RFQ indexer, market makers, and the Injective chain for atomic on-chain settlement."
updatedAt: "2026-04-18"
---

```mermaid
flowchart LR
    Taker["Retail (Taker)"]
    Indexer["RFQ Indexer<br/>(WebSocket)"]
    MM["Market Maker (YOU)<br/>MakerStream"]
    Chain["Injective Chain<br/>(Settlement)"]

    Taker <-->|requests / quotes| Indexer
    Indexer <-->|broadcast / quote| MM
    Taker -->|AcceptQuote| Chain
    MM -.->|authz grants<br/>one-time| Chain

    classDef offchain fill:#eef4ff,stroke:#4a6db8,color:#111
    classDef onchain fill:#eefbf0,stroke:#3c9c60,color:#111
    class Taker,Indexer,MM offchain
    class Chain onchain
```

### How it works

1. **Retail user** sends an RFQ request via the **TakerStream** WebSocket.
2. **RFQ Indexer** broadcasts the request to all connected **Market Makers** via **MakerStream**.
3. **Market Makers** price, sign, and return a quote over MakerStream.
4. **Indexer** relays quotes back to the taker after a collection window.
5. **Retail user** picks a quote and calls `AcceptQuote` on the RFQ contract.
6. **Contract** verifies the MM signature and settles the trade as a synthetic position.

As a MM, you only:

- Connect to MakerStream
- Receive requests
- Sign and send quotes

You do **not** submit an on-chain transaction for each trade — settlement is the taker's responsibility. The contract enforces your signature cryptographically, so you can't be misquoted.

> **TP/SL note:** since contract `0.1.0-alpha.6` ([#23](https://github.com/InjectiveLabs/rfq/pull/23), [#27](https://github.com/InjectiveLabs/rfq/pull/27)) there is a second settlement path — `AcceptSignedIntent` — used for take-profit / stop-loss exits. A relayer submits the trade on the taker's behalf when a mark-price trigger fires. From your side as an MM, you can supply either:
>
> - **Blind quotes** (pre-posted, nonce-based) — the relayer picks from your pre-posted book when a trigger fires.
> - **Taker-specific quotes** (live RFQ response) — the relayer fires off a live RFQ at trigger time and you quote it the usual way.
>
> The wire-level signing and quote shape are identical in both paths. See [Blind quotes](/market-makers/integration/rfq-quotes-blind) below.
