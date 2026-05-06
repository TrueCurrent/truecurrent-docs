---
title: "How TrueCurrent works"
description: "Learn how TrueCurrent's RFQ flow connects traders with competing liquidity providers for zero-fee, non-custodial perpetuals execution."
updatedAt: "2026-05-06"
---

When you place a trade on TrueCurrent, institutional liquidity providers compete to fill it at the best available price. TrueCurrent handles quote selection and execution automatically, so you get RFQ pricing without manually reviewing and accepting individual quotes.

Your funds never leave your wallet. Every fill is settled onchain, with sub-second finality.

---

## The trading experience

### You pick a market and a direction

Choose which perpetual market you want to trade, whether you want to go long (profit when price rises) or short (profit when price falls), and how much you want to trade.

{/* SCREENSHOT SLOT: trade-panel — full TrueCurrent trade panel after market selection (sidebar shown), light mode preferred. */}

### You set your parameters and confirm

Set your margin, leverage, and price slippage (from which we calculate the worst price you're willing to accept). When you click **Trade**, TrueCurrent automatically collects prices from institutional liquidity providers, picks the best one, and executes – all in under a second.

You'll see an estimated price and can set your execution limit before confirming.
The trade will never execute at a less favorable price than your limit.

### Your position is live

{/* SCREENSHOT SLOT: position-open — Positions panel showing one freshly opened long, with entry/mark/uPNL/liq columns visible. */}

---

## What you need to know

**No taker fees.** You pay nothing to trade. No protocol fees, no gas fees.

**Best price, automatically.** TrueCurrent collects quotes from competing liquidity providers and executes at the best one. You never need to manually accept a quote.

**Your execution limit is always respected.** You set a price tolerance before trading.
The trade will never execute at a worse price than your limit –
if no provider can beat it, the trade simply doesn't go through.

**Truly non-custodial.** Your assets stay in your wallet at all times. TrueCurrent never takes custody.

---

## Closing a position

Closing works exactly like opening – set your parameters, confirm, and TrueCurrent handles the rest. You can close fully or partially at any time.

See [How to trade](/trading/how-to-trade) for a step-by-step walkthrough.
