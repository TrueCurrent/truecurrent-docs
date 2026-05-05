---
title: "TP/SL liquidity"
description: "How liquidity providers participate in TrueCurrent take-profit and stop-loss execution using the same EIP-712 v2 quote flow as live RFQ requests."
updatedAt: "2026-05-05"
---

Take-profit and stop-loss orders are submitted as signed taker intents. When a mark-price trigger fires, the relayer sources maker liquidity and settles the close through the same RFQ quote path used for regular trades.

---

## Participation mode

### Live RFQ response

Stay connected to MakerStream.
When a trigger fires, you receive a standard exit RFQ request.
Price it like any other request.
Respond with a signed quote using `sign_quote_v2` and `sign_mode: "v2"`.

From your quoting system's perspective,
a trigger-driven exit uses the same quote schema as a user-initiated request.
The taker direction is still the direction being traded by the taker,
and your maker-side exposure is the opposite side.

---

## Signed-intent failure modes

If you do participate in the TP/SL path,
these are the protocol-level errors specific to signed intents (distinct from normal quote errors):

| Error | Meaning |
|---|---|
| `invalid_intent_signature` / `stale epoch` / `stale lane` | Signature failed verification, or the `epoch` / `lane_version` counter has been incremented since signing (intent was cancelled). |
| `trigger_not_satisfied` | The relayer submitted `AcceptSignedIntent` before the mark price crossed the trigger. The contract re-reads the mark at execution time — relayer-side clock skew or mid-block price reversion both produce this. The intent itself is still valid; the relayer must wait and retry. |
| `quote_rfq_id mismatch` | The maker quote's `rfq_id` does not match the `rfq_id` in the taker's signed intent. |

---

## Cancelling signed intents

Two onchain cancel paths exist,
both of which bump a counter that invalidates any intent signed with the older value:

- `CancelIntentLane`: Invalidates everything for one `(taker, market_id, subaccount_nonce)` lane.
  After this, increment `lane_version` in future intents.
- `CancelAllIntents`: Invalidates every intent for this taker.
  After this, increment `epoch` in future intents.

---

## Reduce-only quotes

To submit a reduce-only quote (one that can only close or shrink an existing position),
pass `margin: "0"` on the quote.
With no capital committed, the protocol cannot open new size.
The only valid fill is one that closes or shrinks existing exposure.
No separate flag is needed.
