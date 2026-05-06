---
title: "Taker SDK trading"
description: "High-level guide for traders and integrators using the TrueCurrent SDK as takers: setup, RFQ request flow, quote selection, settlement, TP/SL intents, and operational requirements."
updatedAt: "2026-05-06"
---

A taker is the trader requesting liquidity. With the SDK, a taker system can open and close TrueCurrent perpetual positions without using the web app.

This path is for:

- Strategy bots
- Arbitrage systems
- Execution desks
- Wallet or app integrations
- Programs that need predictable RFQ execution instead of manual clicking

If you are just placing trades yourself, use the app. If your software needs to trade, use the SDK.

---

## What a taker integration does

A taker integration runs this loop:

1. Choose market, direction, quantity, margin, and `worst_price`
2. Submit an RFQ request through TakerStream
3. Receive the indexer-assigned `rfq_id`
4. Collect maker quotes for that `rfq_id`
5. Choose the best executable quote or quote set
6. Submit `AcceptQuote` to the TrueCurrent contract
7. Monitor the resulting position on Injective

The quote is discovered offchain. The fill is settled onchain.

---

## Required setup

Before the first trade, prepare the wallet and environment:

| Requirement | Why it matters |
|---|---|
| Dedicated Injective wallet | Keeps automated trading isolated from treasury or personal funds |
| INJ gas balance | Needed to broadcast transactions |
| USDC margin | Collateral for perpetual positions |
| Exchange subaccount funding | TrueCurrent settles into Injective exchange subaccounts |
| Taker authz grants | Allows the contract to execute the settlement messages |
| Current market config | Gives your system the right market IDs, tick sizes, and endpoints |

For takers, the authz setup grants the TrueCurrent contract the message types it needs to settle your trades. Grants are scoped to the contract and can be revoked.

---

## Taker flow

### 1. Build the request

Your system decides the trade parameters:

| Parameter | Meaning |
|---|---|
| `market_id` | The perpetual market to trade |
| `direction` | `long` or `short` |
| `quantity` | Position size |
| `margin` | Collateral allocated to the trade |
| `worst_price` | The least favorable price you will accept |
| `client_id` | Your correlation ID for request tracking |

`worst_price` is the critical safety field. For a long, it is the highest price you are willing to pay. For a short, it is the lowest price you are willing to receive.

### 2. Request quotes

The SDK connects to TakerStream and submits the RFQ request. The indexer acknowledges the request and returns the real `rfq_id`.

Use the ACK-returned `rfq_id` for quote collection and settlement. Do not generate your own RFQ ID locally.

### 3. Collect quotes

Makers respond with signed quotes. Your taker system collects quotes for a short window and filters by `rfq_id`.

For a long, lower quote prices are better. For a short, higher quote prices are better.

You can settle one quote or submit multiple quotes in one `AcceptQuote` if you want to aggregate liquidity.

### 4. Settle onchain

After quote selection, the SDK prepares and submits `AcceptQuote`.

The contract re-checks every submitted quote. It verifies the maker signature, quote expiry, maker registration, mark-price band, taker `worst_price`, and available margin. Invalid quotes are skipped. At least one quote must fill for the settlement to succeed.

### 5. Monitor the position

After settlement, the position lives in Injective's exchange module. Monitor:

- Entry price
- Mark price
- Margin ratio
- Liquidation price
- Funding payments
- Open quantity
- Realized and unrealized P&L

---

## Automated exits

Taker SDK integrations can also create TP/SL exits with signed intents.

Instead of submitting a trade immediately, the taker signs an intent that says:

- Which market and subaccount it applies to
- Whether the trigger is `mark_price_gte` or `mark_price_lte`
- What quantity can close
- What `worst_price` must be respected
- When the intent expires

A relayer submits the intent when mark price crosses the trigger. The contract re-checks the trigger at execution time and still requires a valid quote that satisfies `worst_price`.

Signed intents are useful for automated take-profit and stop-loss workflows where the taker does not want to stay online.

---

## What the SDK abstracts

A taker SDK integration should not need to hand-roll:

- gRPC-web WebSocket framing
- TakerStream request encoding
- ACK-based `rfq_id` correlation
- Quote collection and filtering
- Signature and expiry conversion for contract submission
- `AcceptQuote` message construction
- Signed-intent submission and cancellation helpers
- Chain transaction broadcasting

Your application still owns strategy, sizing, risk checks, and position monitoring.

---

## Operational guidance

Use a dedicated trading wallet. Keep only the capital needed for the strategy in that wallet.

Treat quote expiry as a hard latency budget. Collect quotes briefly, choose, and settle immediately.

Always calculate `worst_price` from current mark price and your own risk tolerance. Do not accept arbitrary UI estimates or stale cached prices.

Log the full request, ACK, selected quote, and settlement result. Most integration issues are caused by ID mismatches, stale quotes, missing authz grants, or price strings that changed between signing and submission.

Monitor authz grants and keep a revocation path ready. Revoking the privileged execution grant prevents future SDK settlements for that wallet.

---

## Read next

- [How RFQ works](/technical/how-rfq-works)
- [Authorization model](/technical/authz-model)
- [Smart contract](/technical/smart-contract)
- [Error reference](/technical/error-reference)
