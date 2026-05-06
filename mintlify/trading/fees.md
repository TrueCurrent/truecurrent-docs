---
title: "Fees"
description: "Understand TrueCurrent’s fee model, including zero taker fees, subsidized Injective gas, funding payments, and maker-side fees. "
updatedAt: "2026-04-30"
---

## Takers pay nothing

**TrueCurrent charges zero taker fees.** When you open or close a position, you pay no protocol fee. No taker fee, no transaction fee, no gas fee.

This makes TrueCurrent one of the most cost-effective venues to trade perpetuals anywhere – onchain or off.

### Dollar examples

| Trade size | Taker fee | Maker fee (paid by liquidity provider) |
| --- | --- | --- |
| \$1,000 | \$0.00 | \$0.40 |
| \$10,000 | \$0.00 | \$4.00 |
| \$1,000,000 | \$0.00 | \$400.00 |

As a taker you pay zero regardless of trade size. The maker fee is borne entirely by the institutional liquidity provider who fills your order.

### No volume tiers or fee discounts

**TrueCurrent does not operate volume tiers, VIP programmes, or fee discounts.** The taker fee is 0 bps at all volume levels. There is no benefit to concentrating volume to reduce fees – you already pay nothing.

---

## Gas fees

**Free on TrueCurrent.** TrueCurrent subsidizes Injective-side gas for trading actions, so you do not need to hold or spend INJ to trade.

### How zero gas works

Every trade and position-management action on TrueCurrent involves an Injective blockchain interaction. TrueCurrent subsidizes the Injective network gas for these actions on your behalf.

This does not remove external costs charged outside TrueCurrent, such as source-chain gas when bridging from another network or withdrawal fees charged by a centralized exchange.

---

## Funding fees

Holding an open position over time incurs **funding payments**. These are periodic payments exchanged between long and short holders to keep the perpetual price anchored to spot – not fees paid to TrueCurrent.

See [Funding rates](/trading/funding-rates) for how these are calculated and how to factor them into your trading.

---

## Deposit and withdrawal fees

**Free.** Moving USDC in or out of your TrueCurrent account has no protocol fee.

---

## Fee summary

| Action | Cost to taker |
| --- | --- |
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
| --- | --- | --- | --- |
| **TrueCurrent** | 0 bps | 4 bps | Free |
| Typical DEX | 5–20 bps | 0–5 bps | Transaction gas fee |
| Typical CEX (standard tier) | 3–5 bps | 1–2 bps | None (centralised) |
| Typical CEX (VIP tier) | 1–3 bps | 0–1 bps | None (centralised) |

For a \$10,000 trade, a typical CEX taker fee of 4 bps costs \$4.00. On TrueCurrent, you pay \$0.

---

## For liquidity providers

Institutional liquidity providers who quote on TrueCurrent pay a **4 bps (0.04%) fee** on filled volume. This is how the protocol is sustained while keeping trading completely free for users.

The 4 bps maker fee is **flat – it does not vary by volume, market, or liquidity provider tier**. There are no rebates or tiered maker fee schedules currently.

See [Maker SDK trading](/sdk-trading/makers) for details.

---

## Builder codes

TrueCurrent does not currently operate a public builder code or referral programme. If you are an institutional partner or venue operator interested in fee-sharing or co-marketing arrangements, contact the TrueCurrent team directly.

See [Maker SDK trading](/sdk-trading/makers) for the current maker fee model and whitelisting process.
