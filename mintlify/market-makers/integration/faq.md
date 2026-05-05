---
title: "FAQ & troubleshooting"
description: "Frequently asked questions and troubleshooting guide for TrueCurrent market maker integration: settlement mechanics, quote expiry, partial fills, connection issues, and TP/SL participation."
updatedAt: "2026-05-05"
---

**Do I submit an on-chain tx for every trade?**
No. You sign quotes off-chain. The retail user submits `AcceptQuote`. The contract verifies your signature.

**What if my quote expires before the taker accepts?**
Contract rejects expired quotes.
Use 2s expiry for live MM quotes;
blind/TP-SL quotes can use longer windows (minutes to hours).

**Can I quote a different quantity than requested?**
Yes — partial fills. Set `maker_quantity` / `maker_margin` to your intended amounts. Unfilled remainder depends on the taker's `unfilled_action`.

**What if I don't respond?**
Nothing. No quotes → no trade. No penalty (but your response-rate metric does affect standing).

**How are positions settled?**
Synthetic positions on the Injective exchange module between your subaccount and the taker's. Real positions with real margin, PnL, and liquidation.

**My quotes aren't reaching takers.**
Check, in order:

1. Whitelisted? Query `list_makers`.
   **Pagination caveat:** `list_makers` returns at most 20 makers per page.
   If your address sorts past the first 20,
   the default query returns "not registered" even when you are.
   Page through with `start_after`,
   or use `ContractClient.is_maker_registered()` which paginates automatically.
2. Auth challenge handled?
   Stream connects, pings return pongs,
   but no `request` events.
   This means your `MakerChallenge` handler is missing or produced a wrong signature.
   See [Auth handshake](/market-makers/integration/connecting#auth-handshake).
3. Authz granted? See §1.3.
4. Signature verifies?
   Check `evmChainId` field #1 in `SignQuote`, field order,
   and that `rfq_id` is a JSON number.
5. Quote expired? Verify `expiry` (`now + 2s` for live quotes) and clock sync.

**What's the competition window?**
After a request is sent, the indexer waits for quotes before delivering to the taker.
Testnet ~30–45s; mainnet ~2–10s.
Best price wins (lowest for long, highest for short).

**Connection drops.**
1. Confirm ping every ~1s.
2. Use the `grpc-ws` subprotocol.
3. Reconnect with exponential backoff.

**What changes for TP/SL?**
Nothing on your send path if you only respond to live RFQs.
TP/SL is opt-in:
If another maker has coverage at an acceptable price,
the protocol settles without you.
Not responding carries no reputational cost for TP/SL events.
To participate in pre-posted blind quotes,
see [TP/SL liquidity](/market-makers/integration/rfq-quotes-blind).

**`worst_price exceeds limit` error?**
The v2 indexer endpoints return mark price in human-readable scale
(e.g. `14.85`), not 1e18-scaled.
If you read mark price as `14850000000000000000` and use that as your reference,
your quoted price will be far outside the taker's `worst_price`.
Use the mark price value directly from the v2 endpoint.

**Reduce-only quotes?**
Pass `margin: "0"` on the quote.
With no capital committed, the only valid fill is one that closes or shrinks existing exposure.
The protocol enforces this.
No separate flag is needed.

{/* TODO: add related docs - e2e testnet run book, TPSL flow, RFQ/orderbook/arb flow, admin ops of RFQ contract */}
