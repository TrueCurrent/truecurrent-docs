---
title: "Overview"
description: "Programmatic trading guide for TrueCurrent takers including the RFQ lifecycle, use cases for bots and integrators, comparison with orderbook trading, and pointers to quickstart, authz setup, and the AcceptQuote reference."
updatedAt: "2026-04-08"
---

This section is for developers building programmatic trading on TrueCurrent – arbitrage bots, HFT systems, multi-wallet trading apps, and integrators embedding RFQ execution into their own products.

If you're a human trader clicking buttons, you want [How to trade](/trading/how-to-trade) instead.

---

## What a programmatic taker does

A taker is the party *requesting* a trade. You submit a Request for Quote, receive signed quotes from whitelisted market makers, pick the best ones, and settle onchain. The RFQ flow is **pull-based**: you don't post orders to a book and wait – you broadcast a request, and makers respond within a brief window.

This is different from an orderbook exchange in three important ways:

- **You name the size.** You specify exactly the quantity and margin you want filled. Makers bid to fill it.
- **You see a firm price before signing.** The quote is a signed price commitment. The contract enforces it onchain. No slippage beyond what you declare.
- **Fills are quote-by-quote, not price-by-price.** You either take a quote or you don't. You can combine multiple quotes from different makers into a single atomic settlement.

---

## The taker lifecycle

```
1.  Setup            (one time)   Wallet, fund subaccount, grant authz
                                       │
                                       ▼
2.  Request          (per trade)  Submit RFQ to TakerStream
                                       │
                                       ▼
3.  Collect          (2–5 sec)    Receive signed quotes from N makers
                                       │
                                       ▼
4.  Select                        Sort by price, pick best K that fill your size
                                       │
                                       ▼
5.  Accept           (onchain)    Submit AcceptQuote transaction
                                       │
                                       ▼
6.  Settle                        Contract verifies, opens positions atomically
```

Steps 2–5 take a few hundred milliseconds on a warm connection. Step 6 finalizes in one Injective block (~1 second).

---

## When to build a programmatic taker

- **Arbitrage** – you see a price edge on a CEX or DEX and want to take the opposite side on TrueCurrent
- **Systematic strategies** – mean reversion, momentum, basis trades that need to execute at a known price
- **Integrators** – your app offers perps trading to end users and you want RFQ execution under the hood
- **Multi-wallet operations** – managing positions across many wallets (treasury, LP, desk execution)

---

## Trust model for takers

When you submit an `AcceptQuote` transaction, what you're trusting:

- **The TrueCurrent contract** – deterministic, open source, onchain.
- **Chain finality** – block production and the exchange module.
- **The maker's signature** – which the contract re-verifies on every call. You are *not* trusting the maker to honor their price. The signature binds them.
- **The indexer** – only for delivering quotes. If the indexer were compromised, it could drop or delay messages (degraded service), but could not cause a bad trade. Every onchain check happens again at settlement.

What you are **not** trusting:

- TrueCurrent with custody of your funds – your USDC stays in your wallet
- The indexer to route honestly – any tampered quote fails onchain signature verification
- Makers to be online – if no quotes arrive, no trade happens

---

## Section map

| Page | What it covers |
|---|---|
| [Quickstart](/takers/quickstart) | End-to-end working example in Python and TypeScript |
| [Authorization setup](/takers/authz-setup) | The four `authz` grants a taker must set before trading |
| [TakerStream](/takers/taker-stream) | WebSocket connection, request shape, quote collection |
| [Accepting quotes](/takers/accepting-quotes) | The `AcceptQuote` message in full, single vs multi-quote, encoding gotchas |
| [Best practices](/takers/best-practices) | Slippage, expiry races, reconnection, idempotency, testing |

---

## Before you start

- You will need an Injective wallet with a small amount of **INJ for gas** and **USDC in an exchange subaccount** for margin.
- You will need a machine that can hold a WebSocket connection open – serverless-in-a-box patterns work, but cold-start latency eats into your quote window.
- All programmatic taker traffic today is against **testnet** (`injective-888`). Mainnet endpoints go live at a later phase.
{/* TODO: add mainnet indexer info */}
- The reference implementation for everything in this section lives at [`InjectiveLabs/rfq-testing`](https://github.com/InjectiveLabs/rfq-testing) – Python library, TypeScript examples, and testnet config.

> **API key requirements (current vs future):** today, takers connect to the testnet indexer with no authentication and no rate limit – the reference scripts work out of the box. Once the [RFQ Gateway](https://github.com/InjectiveLabs/rfq-gateway) is deployed in front of the public indexer, every taker request will require an API key, and the default per-key rate limit (10 req/s, burst 20) is too low for HFT. Programmatic high-frequency takers will need an `api`-tier key provisioned with a custom rate limit. See [Authentication](/takers/taker-stream#authentication) and [Rate limiting](/takers/best-practices#rate-limiting) for what changes when that happens.
>
> {/* TODO: remove the "today/future" framing once the gateway is live in front of the public indexer. Document the actual key issuance process (who to contact, what tier to ask for). */}
