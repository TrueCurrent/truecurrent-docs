---
title: "TP/SL caveats for SDK and API users"
description: "Key caveats and edge cases for take-profit and stop-loss trigger orders on TrueCurrent, including off-chain execution, silent failures, worst_price gaps in fast markets, and differences between frontend and SDK behavior."
updatedAt: "2026-04-23"
---

Trigger orders (Take Profit/ Stop Loss, or TP/SL) on TrueCurrent behave differently from traditional exchange stop orders. If you are building against the SDK or API, you need to understand how they work under the hood and where they can fail.

This page covers the operational caveats. For the user-facing explanation, see [Trigger orders (TP/SL)](/trading/trigger-orders). For the full technical specification, see [Signed taker intents](/takers/signed-intents).

---

## TP/SL orders are off-chain until triggered

When you create a TP or SL order, it is **not** submitted to the Injective network. Instead:

1. You sign a `SignedTakerIntent` with your taker private key.
2. The signed intent is submitted to the **RFQ indexer**, which stores it off-chain.
3. The indexer monitors the mark price. When your trigger condition is met, the indexer's relayer submits `AcceptSignedIntent` onchain with maker quotes.

The order only gets added to the Injective network at step 3, at which point, it is *onchain*. Until then, it exists exclusively in the indexer's database. This means:

- **No onchain record** of the pending order exists before it fires.
- **If the indexer goes down**, your trigger order won't execute until it recovers.
- **Your signed intent has a deadline** (max 30 days). If it expires before the trigger fires, the order is dead.

---

## Settlement can fail silently

When a trigger fires, the relayer must find maker quotes and submit a valid `AcceptSignedIntent` transaction. Several things can prevent successful execution:

### No maker quotes available

If no market maker is quoting at the moment the trigger fires — because they're offline, have insufficient balance, or have withdrawn liquidity — there are no quotes to fill against. The relayer cannot construct a valid settlement transaction.

### Contract rejection

Even with quotes, the onchain settlement can fail if:

- The maker quote has expired by the time the transaction lands
- The maker's balance is insufficient
- The `lane_version` has advanced (another intent already settled on this lane)

### `worst_price` gap in fast-moving markets

This is the most important edge case for protective orders.

When you create a TP/SL, you sign a `worst_price` — the worst execution price you'll accept. This value is **fixed at creation time**. At execution time, the contract validates the maker's quote price against your `worst_price` *and* checks the trigger against the **live mark price**.

In a fast-moving market, this creates a gap scenario:

**Stop loss example:**
1. You open a long at $5.00 and set a stop loss at $4.80 with `worst_price = $4.75`.
2. The market drops sharply. The mark price blows through $4.80 and reaches $4.60 before the relayer can land the transaction.
3. At execution time, the trigger condition (mark ≤ $4.80) is satisfied. But no market maker is willing to quote at $4.75 or better when the mark price is at $4.60 — the asset is now worth less than your floor.
4. The settlement fails. Your stop loss did not execute.

**Take profit example:**
1. You open a short at $5.00 and set a take profit at $4.80 with `worst_price = $4.85`.
2. The market drops rapidly through $4.80 to $4.50.
3. At execution time, makers are quoting around $4.50. Your `worst_price` of $4.85 is the *minimum* you'll accept (for a short, this is the ceiling). Quotes at $4.50 are actually *more favorable* than $4.85, so this fills successfully.

The risk is directional: `worst_price` gaps hurt when the market moves **through** your trigger level and keeps going in the adverse direction. For stop losses, this means the market moved too far against you. For take profits on the wrong side, it means you might miss a fill — but the more common case is that take profits fill fine because the price moved in your favor.

**This is expected behavior under the signed intent model.** The `worst_price` is your hard limit — it exists to protect you from a fill at an unacceptable price. The tradeoff is that in a gap scenario, you get no fill rather than a bad fill.

<Warning>
**Set `worst_price` with gap scenarios in mind.** A wider tolerance increases the chance of filling during volatile moves but exposes you to worse execution. A tighter tolerance protects execution quality but increases the chance of a non-fill during fast markets. There is no right answer — it depends on whether you prefer guaranteed execution or guaranteed price quality.
</Warning>

---

## No retry mechanism in v1

If a triggered TP/SL fails to settle — for any of the reasons above — the relayer **flags it as failed but does not retry**. The intent is effectively dead.

This means:

- A stop loss that fails during a flash crash will not re-attempt once conditions stabilize.
- A take profit that fails due to quote expiry will not try again with fresh quotes.
- You must monitor your intent status and manually re-create failed orders.

SDK users should poll for intent status updates via the indexer's WebSocket stream and have logic to detect and re-create failed triggers.

---

## Creating TP/SL without an open position

The signed intent mechanism does not enforce that you have an open position at creation time. You can sign and submit a TP/SL for a market where you have no position.

- The **indexer accepts** the intent and stores it.
- The **contract rejects** at execution time, because the settlement tries to close a position that doesn't exist.

The frontend hides this path — the UI only offers TP/SL controls on open positions. But SDK and API users can construct intents freely, and the indexer won't block them. The failure only surfaces when the relayer tries to settle.

<Tip>
**Always verify you have an open position** in the target market and subaccount before creating a TP/SL intent. Query your position state from the chain or the indexer before signing.
</Tip>

---

## One active TP/SL per trigger type per market

The lane mechanism enforces a **one-intent-per-lane** rule. A lane is defined as `(taker, market_id, subaccount_nonce)`, and each lane has a `lane_version`.

When you create a new TP/SL for a lane that already has an active intent:

- The new intent must use the **current** `lane_version`.
- But the old intent also references the current `lane_version`.
- When either intent settles or you cancel the lane, the `lane_version` increments, invalidating the other.

In practice: **submitting a new TP or SL for the same market replaces the previous one.** The indexer handles this by cancelling the prior intent when you submit a new one for the same trigger type on the same lane.

This means you cannot have, for example, two different stop loss levels active simultaneously on the same position. If you set a SL at $4.80 and then change it to $4.50, the $4.80 order is gone.

---

## SDK users vs. frontend users

The frontend wraps these mechanics with guardrails that hide most failure modes:

| Behavior | Frontend | SDK/API |
|---|---|---|
| TP/SL without a position | Blocked by UI | Allowed; fails at execution |
| `worst_price` calculation | Auto-calculated from tolerance % | You set it manually |
| Failed trigger notification | Shown in UI | You must poll the stream |
| Intent re-creation after failure | Prompted by UI | You must detect and rebuild |
| Lane management | Handled transparently | You manage `lane_version` and `epoch` |

If you are building a programmatic trading system, you accept responsibility for:

- Setting appropriate `worst_price` values that account for potential gaps
- Monitoring intent lifecycle (created → triggered → settled or failed)
- Re-creating intents after failures or cancellations
- Querying position state before creating protective orders
- Managing `lane_version` and `epoch` to avoid stale intents

See [Best practices](/takers/best-practices) for operational guidance on running a programmatic taker.
