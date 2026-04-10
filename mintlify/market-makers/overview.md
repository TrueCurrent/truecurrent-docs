---
title: "Market makers: overview"
description: "Overview of TrueCurrent's market maker program including RFQ liquidity provision, MakerStream integration, quote signing requirements, fee structure, and whitelist application process for professional liquidity providers on Injective."
updatedAt: "2026-04-06"
---

TrueCurrent's liquidity is powered by professional market makers. Makers receive real-time trade requests from users and respond with competitive signed quotes. The best quote wins the trade – keeping spreads tight and prices competitive for everyone.

---

## The market maker role

As a market maker on TrueCurrent, you:

1. **Connect to the MakerStream** – a WebSocket endpoint delivering live trade requests from users
2. **Evaluate each request** – assess the market, direction, size, and your current inventory
3. **Respond with a signed quote** – within a response window of a few hundred milliseconds, submit a cryptographically signed price commitment {/* TODO: add precise MM response deadline once benchmarked */}
4. **Settle onchain** – when a user accepts your quote, the TrueCurrent smart contract settles both sides atomically on Injective

You compete against other registered market makers for every trade. Best price wins.

---

## Fee structure

**Users pay nothing.** TrueCurrent charges zero taker fees – retail traders trade for free.

**Market makers pay 4 bps (0.04%) per filled trade.** This is how the protocol is sustained while keeping trading free for users. The fee is assessed on the notional value of each fill.

---

## Economics for market makers

**Revenue:** You earn the spread between your quoted price and the true mid-market price, minus the 4 bps protocol fee. Tighter spreads win more trades; wider spreads earn more per trade. Finding the optimal balance is the core skill.

**Risk:** You take on the opposite side of each trade. If you fill a taker long on INJ and the price drops, you're short and losing mark-to-market. Managing net inventory is the primary challenge.

**Hedging:** Most professional market makers hedge fills immediately on external venues (CEXes, spot markets) to remain directionally neutral while capturing the spread.

---

## Becoming a market maker

TrueCurrent uses a whitelist of approved market makers. To apply, see [Getting whitelisted](/market-makers/getting-whitelisted).

---

## Technical requirements

- An Injective wallet with sufficient USDT margin
- WebSocket connectivity to TrueCurrent's MakerStream
- An automated quoting system that responds within 2 seconds
- Implementation of TrueCurrent's quote signing specification
- `authz` grants from your wallet to the TrueCurrent contract

**Integration guides:**

- [Connecting to MakerStream](/market-makers/maker-stream)
- [Signing quotes](/market-makers/signing-quotes)
- [Authorization setup](/market-makers/authz-setup)
- [Quoting guidelines](/market-makers/quoting-guidelines)

---

## Capital requirements

There is no hard minimum, but you must maintain sufficient USDT in your subaccount to back the quotes you submit. Quotes submitted without adequate margin result in failed settlements. A healthy market maker maintains enough capital to cover the maximum aggregate notional of their open quotes.
