---
title: "FAQ & troubleshooting"
description: "Frequently asked questions and troubleshooting guide for TrueCurrent market maker integration: settlement mechanics, quote expiry, partial fills, connection issues, and TP/SL participation."
updatedAt: "2026-04-18"
---

**Do I submit an on-chain tx for every trade?**
No. You sign quotes off-chain. The retail user submits `AcceptQuote`. The contract verifies your signature.

**What if my quote expires before the taker accepts?**
Contract rejects expired quotes. Use 20–60s expiry for live quotes. Blind quotes can live longer.

**Can I quote a different quantity than requested?**
Yes — partial fills. Set `maker_quantity` / `maker_margin` to your intended amounts. Unfilled remainder depends on the taker's `unfilled_action`.

**What if I don't respond?**
Nothing. No quotes → no trade. No penalty (but your response-rate metric does affect standing).

**How are positions settled?**
Synthetic positions on the Injective exchange module between your subaccount and the taker's. Real positions with real margin, PnL, and liquidation.

**My quotes aren't reaching takers.**
Check, in order:
1. Whitelisted? Query `list_makers`.
2. Authz granted? See §1.3.
3. Signature verifies? Check field order and that `id`/`e` are JSON numbers.
4. Quote expired? Verify `expiry` and clock sync.

**What's the competition window?**
After a request is sent, the indexer waits for quotes before delivering to the taker. Testnet ~30–45s; mainnet ~2–10s. Best price wins (lowest for long, highest for short).

**Connection drops.**
1. Confirm ping every ~1s.
2. Use the `grpc-ws` subprotocol.
3. Reconnect with exponential backoff.

**What changes for TP/SL?**
Nothing on your send path if you only respond to live RFQs. To participate in pre-posted blind quotes, include a `nonce` on your signed quote and leave `expiry` longer — the indexer keeps your quote in a blind book until it's consumed or expires.

{/* TODO: add related docs - e2e testnet run book, TPSL flow, RFQ/orderbook/arb flow, admin ops of RFQ contract */}
