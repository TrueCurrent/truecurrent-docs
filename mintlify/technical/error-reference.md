---
title: "Error reference"
description: "Troubleshooting guide for common TrueCurrent integration errors including signature verification failures, quote expiry, authz grants, insufficient margin, and WebSocket connection issues."
updatedAt: "2026-05-01"
---

This page lists common errors you may encounter when integrating with TrueCurrent, along with their causes and resolutions.

---

## Smart contract errors

### `signature verification failed`

**Cause:** The market maker's signature on the quote does not match the parameters in the `AcceptQuote` transaction.

**Common causes:**
- Using the old JSON-signing path instead of `rfq_test.crypto.eip712.sign_quote_v2`
- Price decimal format mismatch (for example, signing `4.45` but sending `4.450`)
- Wrong EVM chain ID in the EIP-712 domain (`1439` on testnet, `1776` on mainnet)
- Wrong RFQ contract address in the EIP-712 domain

**Resolution:** Review the [Maker SDK trading](/sdk-trading/makers) signing requirements carefully. Test on testnet and compare your signed message construction against the reference implementation.

---

### `quote expired`

**Cause:** The quote's `expiry` timestamp is in the past at the time of onchain settlement.

**Common causes:**
- Quote `expiry` was set too short (less than the time needed for the taker to accept + block confirmation)
- System clock skew on the market maker's server
- Network delay between quote submission and onchain settlement

**Resolution:** Set quote expiry to `now + 2_000` ms (2 seconds) for live RFQ quotes — that is the canonical window every market-maker reference uses. Ensure your server uses NTP time synchronization. If your settlement chain RTT routinely chews through that budget, co-locate closer to the chain gRPC endpoint rather than extending the expiry — wider expiries widen your exposure to stale-price losses.

---

### `price exceeds worst_price`

**Cause:** The quoted price is worse than the taker's `worst_price` parameter.

**For longs:** `quote.price > worst_price`
**For shorts:** `quote.price < worst_price`

**Resolution (traders):** Widen your `worst_price` setting or wait for better market conditions. See [Slippage and worst price](/trading/slippage-and-worst-price).

**Resolution (market makers):** Ensure your quoted price respects the taker's `worst_price` constraint. Although the contract will catch this, submitting out-of-range quotes wastes your quoting resources and impacts your response metrics.

---

### `unauthorized` / `authz grant not found`

**Cause:** The required `authz` grant from the taker or maker to the contract is missing or expired.

**Resolution:** Re-run the required authz setup. See [Maker SDK trading](/sdk-trading/makers) for maker setup or [Taker SDK trading](/sdk-trading/takers) for taker setup.

---

### `insufficient funds` / `insufficient margin`

**Cause:** The taker's or maker's subaccount doesn't have enough quote-asset margin to cover the trade. On the current testnet RFQ market, the quote asset is USDC.

**Resolution (traders):** Deposit more funds or reduce position size. See [Deposit funds](/getting-started/deposit-funds).

**Resolution (market makers):** Replenish your subaccount balance. Consider implementing balance monitoring that alerts when margin drops below a threshold and reduces quoting activity accordingly.

---

### `maker not whitelisted`

**Cause:** The maker address in the quote is not on the TrueCurrent approved market maker whitelist.

**Resolution:** Apply for whitelist approval. See [Maker SDK trading](/sdk-trading/makers).

---

## Signed-intent errors

These apply specifically to TP/SL settlement via `AcceptSignedIntent`.

### `invalid_intent_signature` / `stale epoch` / `stale lane`

**Cause:** Either the v2 EIP-712 signature failed recovery, or the `epoch` / `lane_version` counter has been incremented since the intent was signed (i.e. the intent has been cancelled or superseded).

**Resolution:** Re-read the current `epoch` and `lane_version` from the contract before signing a new intent. See [Taker SDK trading](/sdk-trading/takers).

---

### `trigger_not_satisfied`

**Cause:** The relayer submitted `AcceptSignedIntent` before the mark price actually crossed the trigger. The contract re-reads mark at execution time, so relayer-side clock skew or a mid-block price reversion both produce this. The intent itself is still valid.

**Resolution:** No action needed from the taker — the relayer should wait and retry once the trigger is satisfied again. If it never re-fires, the intent expires at its `deadline_ms` and you can re-sign.

---

### `quote_rfq_id mismatch`

**Cause:** A maker quote's `rfq_id` does not match the `rfq_id` in the taker's signed intent. Most often hit when a relayer reuses a quote from a different RFQ to settle a triggered intent.

**Resolution:** The relayer must pair each signed intent with quotes that target that intent's `rfq_id` only. As a maker, ensure your TP/SL participation logic signs against the exit RFQ's `rfq_id`, not a stale or unrelated one. As a taker, no action — this is a relayer-side bug surface.

---

## WebSocket / stream errors

### No quotes received

**Symptom:** The taker's `collect_quotes()` returns an empty list after the collection window.

**Common causes:**
- No market makers are currently active for the requested market
- The requesting taker address is not recognized by the indexer (connection issue)
- The market ID in the request is incorrect
- Market makers are whitelisted but their MakerStream is disconnected

**Resolution:** Check that your market ID is correct. Verify MakerStream connections are active. On testnet, confirm the MM wallet is whitelisted. If routing to the order book as fallback, this is expected behavior and not an error.

---

### Connection dropped / timeout

**Symptom:** WebSocket connection closes unexpectedly or `wait_for_request` times out.

**Resolution:** Implement reconnection logic with exponential backoff. See [WebSocket streams](/technical/websocket-streams) for a reconnection example.

---

### `rfq_id` mismatch

**Symptom:** Quotes arrive with a different `rfq_id` than expected, or no quotes arrive for the submitted `rfq_id`.

**Common causes:**
- Multiple concurrent requests from the same address with different ACK-assigned IDs
- Client code is collecting quotes using a locally generated ID instead of the indexer's ACK-returned `rfq_id`

**Resolution:** Send a unique `client_id`, wait for the request ACK, and use `ack["rfq_id"]` for quote collection and settlement.
