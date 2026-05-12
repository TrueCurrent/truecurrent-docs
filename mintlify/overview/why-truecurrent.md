---
title: "Why TrueCurrent"
description: "Compare TrueCurrent against AMMs and order books across trading fees, execution quality, self-custody, MEV resistance, and onchain perpetuals settlement."
updatedAt: "2026-05-06"
---

TrueCurrent is built for traders who want professional execution without taker fees or custodial deposits. Instead of routing trades through an AMM pool or walking an order book, TrueCurrent asks competing liquidity providers for executable prices and settles the best available quote within your limits.

## Zero taker fees

Many perpetual exchanges charge taker fees of 0.02–0.07% per trade. On TrueCurrent, takers pay nothing. Zero protocol fees, zero gas fees.

{/* SCREENSHOT SLOT: fee-comparison — bar chart comparing taker fees TrueCurrent (0%) vs typical CEX vs typical DEX. */}

---

## Best-price execution, automatically

On an AMM, the quoted price can move between order submission and execution. Large trades push prices up or down, sometimes significantly. Bots can front-run your transaction.

On TrueCurrent, you set the worst price you're willing to accept and confirm.
TrueCurrent automatically collects prices from competing institutional liquidity providers and executes at the best one.
The trade never settles outside your limit –
and you never have to shop for a quote yourself.

---

## Self-custody from quote to settlement

Most exchanges, including many marketed as decentralized, require you to deposit funds into a wallet or smart contract they control. On TrueCurrent, your funds never leave your own wallet. There's no deposit into a shared pool, no counterparty holding your assets.

This is what real self-custody looks like.

---

## Bring funds from anywhere

You can fund your TrueCurrent account from any EVM-compatible chain or IBC-compatible chain — and from places where connecting a wallet is awkward (CEX withdraw forms, cold storage) using the QR-code deposit flow that bridges via account abstraction.

See [Deposit funds](/getting-started/deposit-funds) for all three paths.

---

## Fee Comparison

|  | AMM DEX | Order Book DEX | **TrueCurrent** |
| --- | --- | --- | --- |
| Taker fees | 0.05–0.30% | 0.02–0.07% | **0%** |
| Gas fees | High | Low–Medium | **Free** |
| Slippage on large trades | High | Medium | **None** |
| Funds stay in your wallet | ❌ | ❌ | **✅** |
| MEV / sandwich risk | High | Moderate | **Minimal** |
| Settlement speed | Varies | Varies | **\< 1 second** |
