---
title: "TP/SL and makers"
description: "Clarifies that makers do not implement a separate TP/SL flow; the executor submits ordinary RFQs when trigger orders fire."
updatedAt: "2026-05-27"
---

Makers do not need to know whether an RFQ came from a take-profit or stop-loss trigger.

TP/SL orders are taker-side signed intents. The TrueCurrent executor monitors the trigger. When it needs liquidity, it submits a normal RFQ request through the same MakerStream path used for every other trade.

From your maker system's perspective:

- receive request
- price request
- sign with the normal EIP-712 v2 `SignQuote` helper
- send the quote with `sign_mode="v2"` and `evm_chain_id`
- reconcile ACKs and settlement updates

There is no separate TP/SL participation mode, no separate blind-quote workflow, and no TP/SL-specific maker code path to build.

Read [MakerStream](/sdk-trading/maker-stream), [RFQ requests](/sdk-trading/rfq-requests), and [Building and signing quotes](/sdk-trading/signing-quotes) for the current maker integration path.
