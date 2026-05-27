---
title: "Maker SDK trading"
description: "Onboarding guide for market makers integrating with TrueCurrent RFQ: whitelist approval, wallet setup, authz, MakerStream, quote signing, and operational controls."
updatedAt: "2026-05-27"
---

Makers provide active liquidity to TrueCurrent takers. A maker system connects to MakerStream, receives RFQ requests, prices them, signs quotes, and sends those quotes back to the indexer.

The taker or TP/SL executor submits settlement. The maker does not broadcast an onchain transaction per trade. Your signature authorizes only the exact quote terms you signed, and the RFQ contract enforces those terms during settlement.

---

## Who this is for

The maker path is for teams that can operate a low-latency trading system with:

- Reliable WebSocket or gRPC infrastructure
- Real-time reference prices and tick-size handling
- Inventory and hedge management
- USDC margin monitoring
- EIP-712 v2 signing
- Quote and settlement reconciliation
- A kill switch for market, wallet, or feed incidents

Makers must be whitelisted before their quotes are routed to takers and before those quotes can settle.

---

## Maker lifecycle

| Step | Goal | Details |
| --- | --- | --- |
| 1. Generate wallet | Create a dedicated maker key | One private key has both `0x...` and `inj1...` address encodings. Do not use a treasury wallet. |
| 2. Get whitelisted | Register maker address | Send your `inj1...` address and supported markets to TrueCurrent. Confirm with `list_makers`. |
| 3. Grant authz | Allow contract settlement | Grant only the required message types to the current RFQ contract. Keep revocation ready. |
| 4. Fund subaccount | Back maker-side positions | Keep INJ for gas and USDC in the registered exchange subaccount nonce. |
| 5. Connect to MakerStream | Receive RFQs | Use gRPC-web over WebSocket and answer the EIP-712 MakerChallenge. |
| 6. Price, sign, send | Compete on each request | Sign EIP-712 v2 quotes with canonical decimal strings and required wire fields. |
| 7. Reconcile outcomes | Track quote and settlement state | `quote_ack` means routed, not filled. Settlement updates and chain state are the source of fill truth. |

---

## What each RFQ request gives you

Each MakerStream request includes the taker's market, direction, margin, quantity, `worst_price`, request address, expiry, and indexer-assigned `rfq_id`.

Your quoting system decides:

- Whether to quote at all
- Maker-side margin and quantity
- Price
- Expiry
- Minimum fill quantity, if any
- Maker subaccount nonce

The taker direction is from the taker's perspective. If the taker is long, you fill the short side. If the taker is short, you fill the long side.

---

## Required controls

Before quoting live, implement these controls:

| Control | Why it matters |
| --- | --- |
| Price-feed freshness | Avoid stale quotes and adverse selection |
| Mark-price tracking | Contract validation is centered on mark price |
| Tick-size quantization | Unquantized prices or quantities fail validation |
| Canonical decimal formatting | Signed strings must match sent strings byte-for-byte |
| Balance monitoring | Accepted quotes fail if maker margin is insufficient |
| Inventory limits | Prevent one market or direction from dominating risk |
| Quote expiry discipline | Live quote expiry is short; stale quotes are skipped |
| Reconnect logic | Keeps MakerStream availability high |
| Kill switch | Lets you stop quoting quickly during incidents |

If your system cannot price safely, do not quote. No quote means no trade. A stale or undercollateralized quote can still damage maker performance.

---

## Signing requirements

New maker integrations should use EIP-712 v2 only.

Keep these rules visible in your implementation checklist:

- Set `sign_mode` to `"v2"` on every quote.
- Include `evm_chain_id` on the quote wire payload.
- Use the EVM chain ID in the EIP-712 domain: `1439` on testnet, `1776` on mainnet.
- Do not put the EVM chain ID in `quote.chain_id`; that field remains the Cosmos chain ID (`injective-888` testnet, `injective-1` mainnet).
- Convert the RFQ contract bech32 address to its EVM address for the EIP-712 domain.
- Quantize every decimal field before signing.
- Send the exact decimal strings you signed.
- Use compact `v=0/1` signatures.
- Include the maker subaccount nonce used in the signed payload.

For the full layout, see [Building & signing quotes](/sdk-trading/signing-quotes).

---

## Quote lifecycle

`quote_ack.status="success"` means the indexer accepted and routed your quote. It does not mean:

- the taker accepted it,
- the quote filled,
- settlement succeeded,
- or your position changed.

If the taker accepts and settlement is attempted, monitor settlement updates and chain state. If no update arrives before your quote expiry, treat the quote as not accepted.

---

## Read next

1. [Maker whitelist](/sdk-trading/maker-whitelist)
2. [Maker setup](/sdk-trading/maker-setup)
3. [MakerStream](/sdk-trading/maker-stream)
4. [RFQ requests](/sdk-trading/rfq-requests)
5. [Building & signing quotes](/sdk-trading/signing-quotes)
6. [Sending quotes](/sdk-trading/sending-quotes)
7. [Testnet runbook](/sdk-trading/runbook)
