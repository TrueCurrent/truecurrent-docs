---
title: "Maker SDK trading"
description: "High-level guide for market makers using the TrueCurrent SDK: whitelist approval, MakerStream connection, auth challenge, quote signing, risk controls, settlement, and monitoring."
updatedAt: "2026-05-06"
---

Market makers provide liquidity to TrueCurrent takers. A maker system listens for RFQ requests, prices each request, signs a quote, and sends it back through MakerStream.

The taker decides whether to accept the quote. If accepted, the TrueCurrent contract verifies the maker signature and settles both sides on Injective.

---

## Who this is for

The maker path is for professional liquidity providers and trading systems that can:

- Maintain reliable WebSocket infrastructure
- Price perpetual markets in real time
- Manage inventory and hedging
- Respond within the quote window
- Keep sufficient margin available
- Monitor quote quality and settlement outcomes

Market makers must be approved before their quotes are routed to takers.

---

## Maker lifecycle

### 1. Get approved

TrueCurrent uses a maker whitelist. Before your SDK system can quote, your maker wallet must be registered by the TrueCurrent team.

You should be ready to provide:

- Maker Injective address
- Markets you want to support
- Confirmation that your system can run against testnet
- A short description of your quoting and risk controls

### 2. Set up the wallet

A maker wallet needs:

| Requirement | Why it matters |
|---|---|
| INJ gas balance | Required for setup and operational transactions |
| USDC margin | Collateral for maker-side positions |
| Exchange subaccount funding | Quotes settle into Injective exchange subaccounts |
| Maker authz grants | Allows the contract to settle accepted quotes |
| Registered maker address | Required before quotes are accepted |

Use a dedicated maker wallet. Do not grant from a treasury wallet.

### 3. Connect to MakerStream

The SDK connects to MakerStream with your `maker_address`. MakerStream sends a one-time auth challenge before forwarding RFQ requests.

Your system signs the challenge using the EIP-712 v2 auth flow and replies with `MakerAuth`. Until the challenge is accepted, the stream may remain open but will not deliver request events.

### 4. Receive RFQ requests

Each request tells you:

- Market
- Taker direction
- Requested quantity
- Taker margin
- Taker `worst_price`
- Request expiry
- Taker address
- `rfq_id`

MakerStream can deliver requests from many takers. Always filter and track by `rfq_id`.

### 5. Price and sign quotes

Your system decides whether to quote and at what price.

A quote should account for:

- Current mark price
- External reference markets
- Market volatility
- Request size
- Inventory skew
- Funding carry
- Hedging cost
- Available margin
- Minimum fill size

Quotes use EIP-712 v2 signing. The wire payload must include `sign_mode: "v2"` and the correct `evm_chain_id`. Price and quantity strings must be quantized before signing and sent exactly as signed.

### 6. Send quotes

After signing, the SDK sends the quote to MakerStream. If the taker accepts it, the taker submits settlement onchain.

The maker does not submit the taker's settlement transaction. The maker's signature authorizes the quote terms, and the maker's authz grants allow the contract to settle the maker side if the quote is accepted.

### 7. Monitor fills

Track quote updates and settlement updates so your system knows which quotes were accepted, rejected, partially filled, or expired.

After a fill, update:

- Inventory
- Available margin
- Hedge state
- P&L
- Quote skew
- Per-market exposure limits

---

## Required controls

Market-making systems should include these controls before quoting live:

| Control | Why it matters |
|---|---|
| Price-feed freshness checks | Avoid stale quotes and adverse selection |
| Mark-price tracking | Contract validation is centered on mark price |
| Tick-size quantization | Unquantized prices or quantities fail validation |
| Balance monitoring | Accepted quotes fail if maker margin is insufficient |
| Inventory limits | Prevents one market or direction from dominating risk |
| Quote expiry discipline | Reduces stale-price exposure |
| Reconnect logic | Keeps MakerStream availability high |
| Kill switch | Lets you stop quoting quickly during incidents |

If your system cannot price a request safely, do not quote it. A no-quote is better than a stale or undercollateralized quote.

---

## What the SDK abstracts

A maker SDK integration should not need to hand-roll:

- MakerStream connection framing
- Maker auth challenge handling
- RFQ request decoding
- EIP-712 v2 quote signing helpers
- Required wire fields such as `sign_mode` and `evm_chain_id`
- Quote submission encoding
- Quote and settlement update parsing

Your system still owns pricing, inventory, hedging, capital allocation, and uptime.

---

## Signing requirements to remember

New integrations should use EIP-712 v2 only.

Keep these rules visible in your implementation checklist:

- Set `sign_mode` to `"v2"` on every quote
- Include `evm_chain_id` on the wire
- Use the EVM chain ID in the signing domain, not the Cosmos chain ID
- Quantize price and quantity before signing
- Send the exact decimal strings you signed
- Include the maker subaccount nonce used in the signed payload
- Treat missing or stale mark price data as a no-quote condition

On testnet, the EVM chain ID is `1439`. Mainnet uses `1776`.

---

## Maker standing

TrueCurrent monitors maker reliability because poor maker behavior directly affects trader execution.

You can lose standing if your system:

- Frequently misses the response window
- Quotes prices that are consistently not executable
- Allows accepted quotes to fail from insufficient margin
- Sends malformed or invalid signatures
- Quotes from an unregistered address
- Degrades trader execution quality

Normal market-making discretion is expected. You can widen spreads, reduce size, or stop quoting during volatility. The important requirement is that quotes you do send are intentional, valid, and backed by capital.

---

## Read next

- [How RFQ works](/technical/how-rfq-works)
- [Authorization model](/technical/authz-model)
- [Smart contract](/technical/smart-contract)
- [Error reference](/technical/error-reference)
