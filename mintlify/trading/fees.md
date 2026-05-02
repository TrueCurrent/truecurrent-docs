---
title: "Fees"
description: "TrueCurrent charges zero taker fees, zero gas fees, and zero deposit or withdrawal fees for traders, with protocol sustainability through a 4 bps maker fee on institutional liquidity providers."
updatedAt: "2026-04-30"
---

## Takers pay nothing

**TrueCurrent charges zero taker fees.** When you open or close a position, you pay no protocol fee. No taker fee, no transaction fee, no gas fee.

This makes TrueCurrent one of the most cost-effective venues to trade perpetuals anywhere – onchain or off.

### Dollar examples

| Trade size | Taker fee | Maker fee (paid by liquidity provider) |
|------------|-----------|----------------------------------------|
| $1,000 | $0.00 | $0.40 |
| $10,000 | $0.00 | $4.00 |
| $1,000,000 | $0.00 | $400.00 |

As a taker you pay zero regardless of trade size. The maker fee is borne entirely by the institutional liquidity provider who fills your order.

### No volume tiers or fee discounts

**TrueCurrent does not operate volume tiers, VIP programmes, or fee discounts.** The taker fee is 0 bps at all volume levels. There is no benefit to concentrating volume to reduce fees — you already pay nothing.

---

## Gas fees

**Free.** You pay nothing for onchain actions – deposits, trades, withdrawals, or position management.

### How zero gas works

Every transaction on TrueCurrent involves an Injective blockchain interaction that, on a standard Injective deployment, would require a small amount of INJ as gas. TrueCurrent **subsidises these gas costs on your behalf**: the protocol pays the network gas for every user transaction so that you never need to hold or spend INJ to trade.

This is not a "gasless" UX trick or meta-transaction layer — TrueCurrent's smart contract architecture routes gas payment through the protocol, making the zero-gas experience native and permanent for traders.

---

## Funding fees

Holding an open position over time incurs **funding payments**. These are periodic payments exchanged between long and short holders to keep the perpetual price anchored to spot – not fees paid to TrueCurrent.

See [Funding rates](/trading/funding-rates) for how these are calculated and how to factor them into your trading.

---

## Deposit and withdrawal fees

**Free.** Moving USDC in or out of your TrueCurrent account has no protocol fee.

---

## Fee summary

| Action | Cost to you |
|--------|------------|
| Opening a position | **Free** |
| Closing a position | **Free** |
| Gas (all transactions) | **Free** (subsidised by protocol) |
| Deposit | **Free** |
| Withdrawal | **Free** |
| Funding payments | Between traders, not to protocol |

---

## Fee comparison

The table below illustrates how TrueCurrent's fee structure compares to typical centralised exchange (CEX) rates. CEX figures are indicative; exact rates vary by venue and VIP tier.

| Venue type | Taker fee | Maker fee | Network fee |
|------------|-----------|-----------|-------------------|
| **TrueCurrent** | 0 bps | n/a (paid by MM) | Free |
| Typical DEX | 5–20 bps | 0–5 bps | Transaction gas fee |
| Typical CEX (standard tier) | 3–5 bps | 1–2 bps | None (centralised) |
| Typical CEX (VIP tier) | 1–3 bps | 0–1 bps | None (centralised) |

For a $10,000 trade, a typical CEX taker fee of 4 bps costs $4.00. On TrueCurrent, you pay $0.

---

## For liquidity providers

Institutional liquidity providers who quote on TrueCurrent pay a **4 bps (0.04%) fee** on filled volume. This is how the protocol is sustained while keeping trading completely free for users.

The 4 bps maker fee is **flat — it does not vary by volume, market, or liquidity provider tier**. There are no rebates or tiered maker fee schedules currently.

See [Market maker overview](/market-makers/overview) for details.

---

## Builder codes

TrueCurrent does not currently operate a public builder code or referral programme. If you are an institutional partner or venue operator interested in fee-sharing or co-marketing arrangements, contact the TrueCurrent team directly.

See [Market maker overview](/market-makers/overview) for the current market maker fee model and whitelisting process.
