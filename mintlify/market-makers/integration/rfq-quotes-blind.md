---
title: "Blind quotes (for TP/SL support)"
description: "How market makers can participate in TrueCurrent's signed taker intent system for take-profit and stop-loss orders, using either pre-posted blind quotes or live RFQ responses."
updatedAt: "2026-04-18"
---

Since contract `0.1.0-alpha.6`, the RFQ system supports **signed taker intents** — conditional trades (TP/SL) submitted by a relayer when a mark-price trigger fires. Two ways your quotes can participate:

### A. Pre-posted blind quotes

You publish standing quotes keyed by a **`nonce`** rather than a specific `rfq_id`. The indexer stores them in a blind-quote book. When a taker's TP/SL triggers, the relayer picks a matching blind quote and submits it.

The quote shape is identical to a live quote except:

- **Include `nonce`** on the wire (a `u64` you choose; tracks which blind quotes you've had consumed).
- **Omit the taker fields** from the signed payload when quoting blind — the `SignQuote` struct has `t`, `tm`, `tq` as optional (`skip_serializing_if = Option::is_none`), so a blind quote drops those three keys and the contract reconstructs the payload the same way.
- **Longer `expiry`** is acceptable (up to the contract's maker-nonce TTL).

> **TODO (verify):** nothing in `rfq-testing`'s current `sign_quote` exposes a blind-quote signing path yet (the helper always includes `t/tm/tq`). Before pre-posting blind quotes on testnet, confirm the exact signed-JSON shape against a contract-side test case (`rfq-contract/contracts/rfq/src/test/test_blind_quotes.rs`) or ask the team for the canonical helper.

Replay protection is **maker-side**: the contract tracks consumed `(maker, nonce)` pairs. Query your consumed nonces:

```json
{ "maker_nonces": { "maker": "inj1yourmaker..." } }
```

### B. Respond live to the relayer

When a trigger fires, the relayer may fire a live RFQ instead of using pre-posted liquidity. You quote it the normal way — the only difference is the relayer passes `quote_rfq_id` on `AcceptSignedIntent`, and the contract inserts a taker-side nonce for replay protection.

From your quoting infrastructure's perspective: **no code change**. Respond to the RFQ as you would any other.

### Which should you support?

- **Both, ideally.** Pre-posted blind quotes let your liquidity be used without you being online at the moment of trigger. Live-response is useful when blind coverage is thin or when you want to price on current conditions.
- **Protective-only scope:** v1 signed intents are zero-margin exits. So your blind/taker-specific quotes for TP/SL settle against a taker who is closing an existing position, not opening a new one.
