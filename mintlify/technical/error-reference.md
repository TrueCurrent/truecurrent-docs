---
title: "Error reference"
description: "Troubleshooting guide for common TrueCurrent integration errors including signature verification failures, quote expiry, authz grants, insufficient margin, and WebSocket connection issues."
updatedAt: "2026-04-08"
---

This page lists common errors you may encounter when integrating with TrueCurrent, along with their causes and resolutions.

---

## Smart contract errors

### `signature verification failed`

**Cause:** The market maker's signature on the quote does not match the parameters in the `AcceptQuote` transaction.

**Common causes:**
- Incorrect field ordering in the signed message
- Numeric values not serialized as strings (e.g., passing integer `100` instead of string `"100"`)
- Price decimal format mismatch (e.g., `4.455` vs `4.4550`)
- Wrong `chain_id` or `contract_address` in the signed message

**Resolution (market makers):** Review the [Signing quotes](/market-makers/signing-quotes) documentation carefully. Test on testnet and compare your signed message construction against the reference implementation.

**Resolution (programmatic takers):** Most taker-side signature errors come from encoding mistakes when building the `AcceptQuote` message, not from the maker actually signing badly. See [Accepting quotes – Three encoding rules you must follow](/takers/accepting-quotes) for the common gotchas (`rfq_id` as number, `expiry` wrapped as `{"ts": ...}`, `signature` as base64 not hex).

---

### `quote expired`

**Cause:** The quote's `expiry` timestamp is in the past at the time of onchain settlement.

**Common causes:**
- Quote `expiry` was set too short (less than the time needed for the taker to accept + block confirmation)
- System clock skew on the market maker's server
- Network delay between quote submission and onchain settlement

**Resolution:** Ensure your server uses NTP time synchronization and monitor average confirmation times to make sure your expiry setting covers the full round-trip. {/* TODO: CK to confirm the precise minimum quote expiry once benchmarked */}

---

### `price exceeds worst_price`

**Cause:** The quoted price is worse than the taker's `worst_price` parameter.

**For longs:** `quote.price > worst_price`
**For shorts:** `quote.price < worst_price`

**Resolution (traders):** Widen your `worst_price` setting or wait for better market conditions.

**Resolution (market makers):** Ensure your quoted price respects the taker's `worst_price` constraint. Although the contract will catch this, submitting out-of-range quotes wastes your quoting resources and impacts your response metrics.

---

### `unauthorized` / `authz grant not found`

**Cause:** The required `authz` grant from the taker or maker to the contract is missing or expired.

**Resolution:** Re-run the authz grant setup.

- Market makers: see [Authorization setup](/market-makers/authz-setup)
- Programmatic takers: see [Takers: authorization setup](/takers/authz-setup) – note that takers need **four** grants, not three
- UI traders: see [Connect your wallet](/getting-started/connect-wallet)

---

### `insufficient funds` / `insufficient margin`

**Cause:** The taker's or maker's subaccount doesn't have enough USDC to cover the required margin for the trade.

**Resolution (UI traders):** Deposit more funds or reduce position size. See [Deposit funds](/getting-started/deposit-funds).

**Resolution (programmatic takers):** Verify the taker's exchange subaccount balance, not the main bank balance – margin lives in the subaccount. See [Takers: subaccount balance](/takers/best-practices).

**Resolution (market makers):** Replenish your subaccount balance. Consider implementing balance monitoring that alerts when margin drops below a threshold and reduces quoting activity accordingly.

---

### `maker not whitelisted`

**Cause:** The maker address in the quote is not on the TrueCurrent approved market maker whitelist.

**Resolution:** Apply for whitelist approval. See [Getting whitelisted](/market-makers/getting-whitelisted).

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
- Multiple concurrent requests from the same address with different IDs
- ID generation collision (avoid using sequential small integers; use timestamps or UUIDs)

**Resolution:** Use millisecond timestamps as `rfq_id` values to ensure uniqueness. Ensure your quote collection is filtering by the correct `rfq_id`.
