---
title: "Order types"
description: "Complete reference for all order types on TrueCurrent: RFQ fill, order book market and limit fallback, take profit, stop loss, and signed intents — when each executes, its price guarantee, and when to use it."
updatedAt: "2026-04-30"
---

TrueCurrent's execution model is RFQ-first, not CLOB-first. On every trade, the system solicits competitive signed quotes from professional market makers and fills at the best available price — with zero taker fees. An order book fallback is available for residual unfilled quantity, and conditional trigger orders automate exits. This page documents every order type, what guarantee it carries, and when to reach for it.

---

## Execution model overview

| Execution path | Who fills | Price guarantee | Taker fee |
|---|---|---|---|
| RFQ | Competing market makers | `worst_price` enforced onchain | Zero |
| Order book market fallback | Injective CLOB | None (market price) | Standard |
| Order book limit fallback | Injective CLOB | At or better than limit price | Standard |
| Trigger (TP/SL) | Market makers via relayer | `worst_price` in signed intent | Zero |

---

## RFQ fill (primary)

Every trade on TrueCurrent starts as an RFQ. When you submit a trade, TrueCurrent broadcasts a request to all registered market makers simultaneously. Each maker responds with a signed quote. TrueCurrent selects the best quote and settles the position onchain in a single transaction.

**When it executes:** immediately on trade submission; fills within the quote collection window (typically under one second)

**Price guarantee:** your `worst_price` parameter is enforced by the contract — the trade cannot settle at a less favorable price. If no maker can beat your worst price, the trade does not execute and your margin is returned.

**When to use:** always — this is the default path for every trade opened or closed through the UI or API

**Zero taker fees** apply to all RFQ fills. The market maker pays a 4 bps fee on filled volume; you pay nothing.

See [Price tolerance](/trading/slippage-and-worst-price) for how to set `worst_price` and recommended tolerance ranges.
See [How RFQ works](/technical/how-rfq-works) for the full protocol detail.

---

## Order book market fallback (`unfilled_action: market`)

If an RFQ returns quotes that only partially fill your requested quantity — or returns no quotes at all — you can instruct TrueCurrent to route the remaining quantity to the Injective order book as a market order.

**When it executes:** immediately after the RFQ phase, for any quantity not covered by RFQ quotes

**Price guarantee:** none on the unfilled portion — the order book fill executes at whatever price is available

**When to use:** when fill certainty matters more than exact price; useful for larger sizes where a single maker may not cover the full quantity

**How to enable:** pass `unfilled_action: { "market": {} }` in your API/SDK trade request

```json
{
  "unfilled_action": { "market": {} }
}
```

<Warning>
The order book market fill carries no price protection. In a fast-moving market, the fill price on the unfilled portion may be significantly worse than the RFQ fill price. Set an appropriate `worst_price` tolerance or use the limit fallback if price protection on the full quantity matters.
</Warning>

---

## Order book limit fallback (`unfilled_action: limit`)

If an RFQ partially fills your quantity, the remaining unfilled quantity is placed as a resting limit order on the Injective order book at your specified price.

**When it executes:** immediately after RFQ for any residual quantity; the limit order rests until it fills or you cancel it

**Price guarantee:** at or better than the limit price (same as any limit order)

**When to use:** when you want price protection on the full quantity and are willing to wait for the unfilled residual to fill later; more appropriate than the market fallback when you have a price target

**How to enable:** pass `unfilled_action: { "limit": {} }` in your API/SDK trade request

```json
{
  "unfilled_action": { "limit": {} }
}
```

<Note>
A resting limit order from this fallback continues to sit on the order book until it fills or is explicitly cancelled. Monitor open orders if you use this mode.
</Note>

---

## Take profit (TP)

A take profit order closes your position automatically when the price reaches a favorable level you specify in advance.

- **Long TP:** triggers when the quoted price rises to or above your target
- **Short TP:** triggers when the quoted price falls to or below your target

**When it executes:** when the mark price crosses your trigger level; the TrueCurrent relayer executes `AcceptSignedIntent` onchain and closes the position at the best available quote

**Price guarantee:** your `worst_price` in the signed intent is enforced; the exit will not settle at a less favorable price than your limit

**When to use:** to lock in gains automatically without watching the market; can be set at position open or added later

**Trigger evaluated against:** mark price (not the quoted price or index price); the contract re-evaluates the trigger at execution time

See [Trigger orders (TP/SL)](/trading/trigger-orders) for setup instructions and worked examples.

---

## Stop loss (SL)

A stop loss order closes your position automatically when the price moves against you to a level you specify.

- **Long SL:** triggers when the mark price falls to or below your stop price
- **Short SL:** triggers when the mark price rises to or above your stop price

**When it executes:** same as take profit — relayer fires `AcceptSignedIntent` when trigger condition is satisfied

**Price guarantee:** `worst_price` in the signed intent is enforced; in a fast-moving market, if no maker can beat your worst price the order will retry until it fills

**When to use:** to cap downside automatically; essential protection for leveraged positions held overnight or during high-volatility periods

<Warning>
Setting a stop loss very close to your entry price at high leverage risks being stopped out by normal market noise. Give your position room to breathe while still protecting against a meaningful adverse move. See [Managing risk with TP/SL](/trading/trigger-orders#managing-risk-with-tpsl).
</Warning>

See [Trigger orders (TP/SL)](/trading/trigger-orders) for setup instructions. For programmatic use, see [Signed taker intents](/takers/signed-intents).

---

## Signed intents (programmatic)

A signed taker intent is a pre-authorized, conditional trade instruction that you sign offline and submit to the TrueCurrent relayer. When the trigger condition is satisfied (mark price crosses a threshold, or immediately), the relayer submits `AcceptSignedIntent` onchain and settles the position.

**When it executes:** when the mark price satisfies the trigger (or immediately for `{ "immediate": {} }` trigger); relies on the relayer monitoring mark price

**Price guarantee:** `worst_price` in the intent is enforced by the contract

**When to use:** when you cannot be online at the exact moment a trade should fire; the canonical mechanism behind TP/SL orders; also used by programmatic takers building automated strategies

**V1 constraints:**
- Exit/protective flow only — `margin` must be `0` (cannot open new positions via signed intent)
- No order book fallback — `unfilled_action` must be `null`
- Maximum deadline: 30 days from signing

See [Signed taker intents](/takers/signed-intents) for the full protocol specification including signing, quote binding modes, cancellation, and execution semantics.

---

## What TrueCurrent does not offer

<Note>
This list is for traders arriving from platforms with a full CLOB order type suite. Understanding what TrueCurrent does not offer prevents building integrations against order types that do not exist.
</Note>

| Order type | Status | Note |
|---|---|---|
| TWAP (time-weighted average price) | **Not available** | TC has no native TWAP execution; replicate externally by splitting trades over time via the API |
| Scale / ladder orders | **Not available** | No built-in order scaling; implement by submitting multiple individual trades |
| Iceberg orders | **Not available** | Full quantity is broadcast to all market makers on each RFQ |
| Isolated-margin order forms | **Not available** | TC is cross-margin only — there is no isolated-margin trade flow. See [Margin trading](/trading/margin-trading) |
| Post-only / maker-only limit orders | **Not available** | TC's primary execution is RFQ; makers are institutional liquidity providers, not retail users posting to the book |
| Fill-or-kill (FOK) | **Not available** | Partial fills are handled via `unfilled_action`; there is no FOK mode that reverts the entire trade if not fully filled in one shot |
| Immediate-or-cancel (IOC) | **Not available** | No IOC mode; the RFQ collects quotes within a window and fills the best available |
| Conditional (price-alert) entry orders | **Not available in V1** | Signed intents V1 is exit/protective flow only; entry intents may be added in a future version |

---

## RFQ execution vs order book execution

These two paths have materially different properties:

| Property | RFQ fill | Order book fill |
|---|---|---|
| Who provides liquidity | Institutional market makers | Any order book participant |
| Price competition | Makers compete on every request | Best resting order wins |
| Taker fee | Zero | Standard (see [Fees](/trading/fees)) |
| Fill guarantee | `worst_price` enforced | None for market; limit price for limit |
| Latency | Sub-second (quote collection window) | Block time |
| Partial fill handling | Configurable via `unfilled_action` | Standard CLOB partial fill |

The order book is not the primary liquidity source on TrueCurrent — it is a fallback for quantity that RFQ cannot cover. Routing to the order book deliberately should be an informed choice, not a default.

---

## Related pages

- [Trigger orders (TP/SL)](/trading/trigger-orders) — UI setup and worked examples
- [Trigger orders — advanced](/trading/trigger-orders-advanced) — programmatic execution, silent failure modes, `worst_price` gap scenarios
- [Price tolerance](/trading/slippage-and-worst-price) — setting `worst_price` and tolerance ranges
- [How RFQ works](/technical/how-rfq-works) — RFQ protocol detail, timing, and validation
- [Signed taker intents](/takers/signed-intents) — signed intent specification for programmatic takers
