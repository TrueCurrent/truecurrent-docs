---
title: "Liquidity providers: overview"
description: "Overview of TrueCurrent's liquidity provider program including RFQ liquidity provision, MakerStream integration, quote signing requirements, fee structure, and whitelist application process for professional liquidity providers on Injective."
updatedAt: "2026-05-01"
---

TrueCurrent's liquidity is powered by professional liquidity providers. Makers receive real-time trade requests from users and respond with competitive signed quotes. The best quote wins the trade – keeping spreads tight and prices competitive for everyone.

---

## The liquidity provider role

As a liquidity provider on TrueCurrent, you:

1. **Connect to the MakerStream** – a WebSocket endpoint delivering live trade requests from users
2. **Evaluate each request** – assess the market, direction, size, and your current inventory
3. **Respond with a signed quote** – within a response window of a few hundred milliseconds, submit a cryptographically signed price commitment {/* TODO: add precise MM response deadline once benchmarked */}
4. **Settle onchain** – when a user accepts your quote, the TrueCurrent smart contract settles both sides atomically on Injective

You compete against other registered liquidity providers for every trade. Best price wins.

---

## Fee structure

**Users pay nothing.** TrueCurrent charges zero taker fees – retail traders trade for free.

**Liquidity providers pay 4 bps (0.04%) per filled trade.** This is how the protocol is sustained while keeping trading free for users. The fee is assessed on the notional value of each fill.

---

## Economics for liquidity providers

**Revenue:** You earn the spread between your quoted price and the true mid-market price, minus the 4 bps protocol fee. Tighter spreads win more trades; wider spreads earn more per trade. Finding the optimal balance is the core skill.

**Risk:** You take on the opposite side of each trade. If you fill a taker long on INJ and the price drops, you're short and losing mark-to-market. Managing net inventory is the primary challenge.

**Hedging:** Most professional liquidity providers hedge fills immediately on external venues (CEXes, spot markets) to remain directionally neutral while capturing the spread.

---

## Becoming a liquidity provider

TrueCurrent uses a whitelist of approved liquidity providers. To apply, see [Getting whitelisted](/liquidity-providers/getting-whitelisted).

---

## Technical requirements

- An Injective wallet with sufficient USDC margin for the current RFQ market
- WebSocket connectivity to TrueCurrent's MakerStream
- An automated quoting system that responds within 2 seconds
- Implementation of TrueCurrent's quote signing specification
- `authz` grants from your wallet to the TrueCurrent contract

**Integration guides:**

- [Connecting to MakerStream](/liquidity-providers/maker-stream)
- [Signing quotes](/liquidity-providers/signing-quotes)
- [Authorization setup](/liquidity-providers/authz-setup)
- [Quoting guidelines](/liquidity-providers/quoting-guidelines)

---

## Capital requirements

There is no hard minimum, but you must maintain sufficient quote-asset margin in your subaccount to back the quotes you submit. On the current testnet market, that quote asset is USDC. Quotes submitted without adequate margin result in failed settlements. A healthy liquidity provider maintains enough capital to cover the maximum aggregate notional of their open quotes.
