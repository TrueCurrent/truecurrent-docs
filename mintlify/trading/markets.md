---
title: "Available markets"
description: "How perpetual futures markets are listed on TrueCurrent and how to find the live list of supported markets and their parameters."
updatedAt: "2026-05-27"
---

TrueCurrent offers perpetual futures markets on Injective. Each market is collateralized in USDC, has an onchain mark price used for liquidations, funding, and trigger-order evaluation, and exposes a streamed `indexPrice` used as the primary UI P&L reference.

The live list of supported markets, current parameters (tick size, max leverage, funding cadence), and 24h volume is shown in the trading interface. For programmatic access, query the Injective exchange module's `DerivativeMarket` endpoint — that is the onchain source of truth.

---

## How new markets are listed

New markets on TrueCurrent require at least one registered maker willing to provide liquidity. The listing process is:

1. The community or the team identifies demand for a new market
2. At least one maker commits to providing RFQ liquidity
3. The market is deployed via Injective governance (this sets tick size, lot size, IMR, MMR, funding cap, and max open interest)
4. The market is enabled in the TrueCurrent UI

If you are a maker interested in supporting a new market, see [Maker SDK trading](/sdk-trading/makers).

---

## Market reference prices

Each market exposes an onchain `mark_price` through Injective. TrueCurrent uses that mark price for liquidations, funding-rate calculations, TP/SL trigger evaluation, margin checks, and quote validation. It is public and verifiable onchain.

The indexer streams `indexPrice` alongside `markPrice`. TrueCurrent uses `indexPrice` as the primary source for displayed **Index Price** and unrealized P&L.

For the full index, mark, and quoted-price model, see [Index, mark, and quoted prices](/trading/prices).
