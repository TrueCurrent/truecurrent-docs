---
title: "TP/SL liquidity"
description: "How market makers participate in TrueCurrent take-profit and stop-loss execution using the same EIP-712 v2 quote flow as live RFQ requests."
updatedAt: "2026-05-01"
---

Take-profit and stop-loss orders are submitted as signed taker intents. When a mark-price trigger fires, the relayer sources maker liquidity and settles the close through the same RFQ quote path used for regular trades.

For public testnet and mainnet integrations, support the live RFQ response path first:

1. Stay connected to MakerStream.
2. Receive the relayer's exit RFQ request.
3. Price the close like any other request.
4. Sign with `sign_quote_v2`.
5. Send the quote with `sign_mode: "v2"`.

From your quoting system's perspective, a trigger-driven exit request uses the same quote schema as a user-initiated request. The taker direction is still the direction being traded by the taker, and your maker-side exposure is the opposite side.

Pre-posted blind quote books are not part of the public onboarding path yet. Do not ship a blind-quote integration unless the TrueCurrent team has provided the current production schema and test coverage for that path.
