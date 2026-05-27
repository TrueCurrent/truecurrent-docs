---
title: "Integration guide"
description: "Overview of the TrueCurrent maker integration: what you connect to, what you sign, what you settle, and where to start in the guide depending on whether you're scaffolding a new maker or porting an existing one."
updatedAt: "2026-05-05"
---

This guide walks you from a fresh Injective wallet to a maker quoting live on TrueCurrent. The flow is short:

1. **Generate an Injective wallet** and share the `inj1…` address with the TrueCurrent team
2. **Get whitelisted** on the RFQ contract (admin call)
3. **Grant authz permissions** so the contract can settle your fills
4. **Connect to MakerStream** over gRPC-web WebSocket and complete the EIP-712 v2 auth handshake
5. **Receive RFQ requests** as they're broadcast from takers
6. **Price, sign, and send signed quotes** — `sign_quote_v2` from the reference SDK takes care of the EIP-712 v2 digest
7. **Settle** — the taker submits `AcceptQuote` on-chain; the contract verifies your signature and opens both sides atomically

That's the whole loop. The rest of this guide is the *reference detail* for each step.

---

## What changes from a generic MM integration

If you've integrated with other RFQ venues, three things on TrueCurrent are worth front-loading:

- **EIP-712 v2 signing only.** Every quote and every TakerStream conditional order is signed against an EIP-712 typed-data digest using the EVM chain ID (`1439` testnet, `1776` mainnet). Sign mode `v2` is required on the wire. There is no manual-JSON signing path for new integrations. See [Building & signing quotes](/market-makers/integration/rfq-quotes-sign).
- **Auth handshake before quotes start arriving.** When you connect to MakerStream, the indexer issues a one-shot `MakerChallenge` you must sign with EIP-712 v2 and reply to. Until you reply, the stream stays open but produces no `request` events. See [Connecting to the RFQ stream](/market-makers/integration/connecting#auth-handshake).
- **You sign quotes, the taker submits the transaction.** You never broadcast settlement on-chain yourself — the retail taker does, and the contract verifies your signature. Your only on-chain footprint is the one-time authz grant.

---

## Recommended reading order

**Scaffolding a new maker:**

1. [Architecture overview](/market-makers/integration/architecture) — the three parties, what each does
2. [Quick start checklist](/market-makers/integration/quick-start) — the seven-step path from wallet to first quote
3. [One-time setup](/market-makers/integration/setup) — wallet generation, whitelist request, authz, subaccount funding
4. [Connecting to the RFQ stream](/market-makers/integration/connecting) — endpoint, framing, ping, auth handshake
5. [Receiving RFQ requests](/market-makers/integration/rfq-requests) — request shape and field meanings
6. [Building & signing quotes](/market-makers/integration/rfq-quotes-sign) — EIP-712 v2 schema, helper, decimal hygiene
7. [Sending quotes](/market-makers/integration/rfq-quotes-send) — wire format, ACK / error responses
8. [Reference implementations](/market-makers/integration/reference-implementations) — Python, TypeScript, Go entry points

**Porting an existing maker stack:**

1. [Protocol reference](/market-makers/integration/protocol-reference) — full sequence diagrams (sync `AcceptQuote` and TP/SL `AcceptSignedIntent` paths) plus testnet market list
2. [Building & signing quotes](/market-makers/integration/rfq-quotes-sign) — the `SignQuote` field order is precise; mismatches fail signature verification silently
3. [Connecting to the RFQ stream](/market-makers/integration/connecting) — auth handshake details that aren't optional
4. [TP/SL and makers](/market-makers/integration/rfq-quotes-blind) — why makers do not need a separate TP/SL code path
5. [Testnet configuration](/market-makers/integration/testnet-configuration) — endpoints, contract address, EVM chain IDs
6. [FAQ & troubleshooting](/market-makers/integration/faq) — settlement mechanics, expiry, partial fills, "why aren't my quotes reaching takers"

---

## What you don't need to worry about

- **You don't broadcast settlement transactions.** The taker does. Your authz grant authorises the contract to open your position when the taker accepts your signed quote.
- **You don't manage taker margin.** The taker funds their own subaccount and the contract verifies sufficient balance at settlement.
- **You don't track quote IDs across reconnects.** Each `rfq_id` is independent; if you miss one because the stream dropped, the taker simply gets fewer competing quotes for that request.
- **You don't need to handle gas.** Settlement gas is paid by the taker's transaction. Your only gas cost is the one-time authz grant.

---

## Reference repository

All examples live in [`InjectiveLabs/injective-rfq-toolkit`](https://github.com/InjectiveLabs/injective-rfq-toolkit). The Python `rfq_test` package is the reference client for both makers and takers; the `examples/` directory has working scripts for Python (WebSocket and gRPC), TypeScript, and Go.
