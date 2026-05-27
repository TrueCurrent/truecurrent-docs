---
title: "TP/SL and makers"
description: "Clarifies that makers do not implement a separate TP/SL flow on TrueCurrent; the executor submits ordinary RFQ requests when trigger orders fire."
updatedAt: "2026-05-27"
---

Makers do not need a separate TP/SL integration.

Take-profit and stop-loss orders are taker-side signed intents. The TrueCurrent executor monitors the trigger condition. When a trigger fires, the executor requests liquidity through the normal RFQ path.

From a maker system's perspective, the resulting request is just another MakerStream RFQ:

- It has a normal `rfq_id`.
- It has a normal market, direction, margin, quantity, and `worst_price`.
- It is priced, signed, and sent with the same `SignQuote` flow as any other taker-bound RFQ.
- It does not require the maker to inspect, store, or understand the taker's TP/SL intent.
- It does not require a separate blind-quote book or TP/SL-specific quote mode.

If you are building a maker, keep your focus on the normal MakerStream loop:

1. Receive request.
2. Price request.
3. Sign quote with EIP-712 v2.
4. Send quote with `sign_mode="v2"` and `evm_chain_id`.
5. Reconcile quote ACKs and settlement updates.

The executor owns trigger monitoring, signed-intent submission, and `AcceptSignedIntent` execution.

---

## What changes for a maker?

Nothing in the quote path.

You may choose to identify executor-driven flow in logs if the request metadata exposes a recognizable source, but your quoting code should not depend on that. Treat all MakerStream RFQ requests the same unless your own risk model decides to filter by market, size, direction, or inventory.

---

## What belongs in taker/executor docs?

The signed-intent details are relevant for taker SDKs and executor infrastructure:

- taker `SignedTakerIntent` signing
- trigger type and trigger price
- `epoch` and `lane_version`
- conditional-order submission fields
- `AcceptSignedIntent`
- intent cancellation

Those details are not maker integration requirements.

---

## Read next

- [MakerStream](/sdk-trading/maker-stream)
- [RFQ requests](/sdk-trading/rfq-requests)
- [Building and signing quotes](/sdk-trading/signing-quotes)
- [Signed taker intents](/sdk-trading/signed-intents)
