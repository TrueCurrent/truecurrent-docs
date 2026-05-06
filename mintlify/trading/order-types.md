---
title: "Order types"
description: "Complete reference for all order types on TrueCurrent: RFQ fill, take profit, stop loss, and signed intents — when each executes, its price guarantee, and when to use it."
updatedAt: "2026-05-06"
---

TrueCurrent's execution model is RFQ-only. On every trade, the system solicits competitive signed quotes from professional market makers and fills at the best available price — with zero taker fees. Conditional trigger orders (TP/SL) automate exits using the same RFQ quote path. This page documents every order type, what guarantee it carries, and when to reach for it.

---

## Execution model overview

| Execution path | Who fills | Price guarantee | Taker fee |
|---|---|---|---|
| RFQ | Competing market makers | `worst_price` enforced onchain | Zero |
| Trigger (TP/SL) | Market makers via relayer | `worst_price` in signed intent | Zero |

---

## RFQ fill (primary)

Every trade on TrueCurrent is an RFQ. When you submit a trade, the indexer broadcasts the request to all registered market makers. Each maker responds with a signed quote within the collection window (~2 seconds). The best quote wins and settles onchain in a single atomic transaction.

**When it executes:** immediately on trade submission; fills within the quote collection window.

**Price guarantee:** your `worst_price` parameter is enforced by the contract — the RFQ fill cannot settle at a less favorable price. If no maker can satisfy your `worst_price`, the trade does not execute and your margin is released.

**When to use:** always — this is the path for every trade opened or closed through the UI or API.

**Zero taker fees** apply to all RFQ fills. The market maker pays a 4 bps fee on filled volume; you pay nothing.

See [Price tolerance](/trading/slippage-and-worst-price) for how to set `worst_price` and recommended tolerance ranges. See [How RFQ works](/technical/how-rfq-works) for the full protocol detail.

---

## Take profit (TP)

A take profit order closes your position automatically when the price reaches a favorable level you specify in advance.

- **Long TP:** triggers when the mark price rises to or above your target
- **Short TP:** triggers when the mark price falls to or below your target

**When it executes:** when the mark price crosses your trigger level; the TrueCurrent relayer executes `AcceptSignedIntent` onchain and closes the position at the best available quote.

**Price guarantee:** your `worst_price` in the signed intent is enforced; the exit will not settle at a less favorable price than your limit.

**When to use:** to lock in gains automatically without watching the market; can be set at position open or added later.

**Trigger evaluated against:** mark price, not quoted price. The contract re-evaluates the trigger at execution time.

See [Trigger orders (TP/SL)](/trading/trigger-orders) for setup instructions and worked examples.

---

## Stop loss (SL)

A stop loss order closes your position automatically when the price moves against you to a level you specify.

- **Long SL:** triggers when the mark price falls to or below your stop price
- **Short SL:** triggers when the mark price rises to or above your stop price

**When it executes:** same as take profit — the relayer fires `AcceptSignedIntent` when the trigger condition is satisfied.

**Price guarantee:** `worst_price` in the signed intent is enforced; in a fast-moving market, if no maker can beat your `worst_price` the order will retry until it fills.

**When to use:** to cap downside automatically; essential protection for leveraged positions held overnight or during high-volatility periods.

<Warning>
Setting a stop loss very close to your entry price at high leverage risks being stopped out by normal market noise. Give your position room to breathe while still protecting against a meaningful adverse move. See [Managing risk with TP/SL](/trading/trigger-orders#managing-risk-with-tpsl).
</Warning>

See [Trigger orders (TP/SL)](/trading/trigger-orders) for setup instructions. For programmatic use, see [Taker SDK trading](/sdk-trading/takers).

---

## Signed intents (programmatic)

A signed taker intent is a pre-authorized, conditional trade instruction that you sign offline and submit to the TrueCurrent relayer. When the trigger condition is satisfied (mark price crosses a threshold, or immediately), the relayer submits `AcceptSignedIntent` onchain and settles the position.

**When it executes:** when the mark price satisfies the trigger (or immediately for `{ "immediate": {} }` trigger); relies on the relayer monitoring mark price.

**Price guarantee:** `worst_price` in the intent is enforced by the contract.

**When to use:** when you cannot be online at the exact moment a trade should fire; the canonical mechanism behind TP/SL orders; also used by programmatic takers building automated strategies.

**v1 constraints:**
- Exit/protective flow only — `margin` must be `0` (cannot open new positions via signed intent)
- `unfilled_action` must be `null`
- Maximum deadline: 30 days from signing

See [Taker SDK trading](/sdk-trading/takers) for the SDK-level signed-intent flow.

---

## What TrueCurrent does not offer

<Note>
This list is for traders arriving from platforms with a full CLOB order type suite. Understanding what TrueCurrent does not offer prevents building integrations against order types that do not exist.
</Note>

| Order type | Status | Note |
|---|---|---|
| Resting limit orders | **Not available** | TC has no public order book; every fill is an RFQ |
| Order book market / limit fallback | **Not available** | Every trade is RFQ-only; partial-fill quantity is simply not traded |
| TWAP (time-weighted average price) | **Not available** | TC has no native TWAP execution; replicate externally by splitting trades over time via the API |
| Scale / ladder orders | **Not available** | No built-in order scaling; implement by submitting multiple individual trades |
| Iceberg orders | **Not available** | Full quantity is broadcast to all market makers on each RFQ |
| Isolated-margin order forms | **Not available** | TC is cross-margin only — there is no isolated-margin trade flow. See [Margin trading](/trading/margin-trading) |
| Post-only / maker-only limit orders | **Not available** | TC's primary execution is RFQ; makers are institutional liquidity providers, not retail users posting to the book |
| Fill-or-kill (FOK) | **Not available** | Partial fills are handled by the contract's quote-by-quote consumption; there is no FOK mode |
| Immediate-or-cancel (IOC) | **Not available** | RFQ collects quotes within a window and fills the best available; the closest analogue is a tight `worst_price` |
| Conditional (price-alert) entry orders | **Not available in v1** | Signed intents v1 is exit/protective flow only; entry intents may be added in a future version |

---

## Related pages

- [Trigger orders (TP/SL)](/trading/trigger-orders) — UI setup and worked examples
- [Trigger orders — advanced](/trading/trigger-orders-advanced) — programmatic execution, silent failure modes, `worst_price` gap scenarios
- [Price tolerance](/trading/slippage-and-worst-price) — setting `worst_price` and tolerance ranges
- [How RFQ works](/technical/how-rfq-works) — RFQ protocol detail, timing, and validation
- [Taker SDK trading](/sdk-trading/takers) — SDK flow for programmatic trading and signed intents
