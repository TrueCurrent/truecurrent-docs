---
title: "Error reference"
description: "Common TrueCurrent RFQ errors and fixes for signatures, quote expiry, stream auth, authz, margin, and signed-intent settlement."
updatedAt: "2026-05-27"
---

Use this page as a compact error index. For step-by-step triage, see [Troubleshooting](/sdk-trading/troubleshooting).

---

## Quote and settlement errors

### `signature verification failed` / `invalid_signature`

The maker signature does not recover against the submitted quote fields.

Common causes:

- Signed decimal string differs from the sent decimal string.
- EVM chain ID in the EIP-712 domain is wrong (`1439` testnet, `1776` mainnet).
- `quote.chain_id` was set to the EVM chain ID instead of the Cosmos chain ID.
- RFQ contract address in the EIP-712 domain is stale.
- Field order differs from the v2 `SignQuote` schema.

Fix: compare the exact typed-data message to the exact wire payload. Use [Building and signing quotes](/sdk-trading/signing-quotes) as the field-order reference.

---

### `quote expired`

The quote's `expiry` timestamp was not greater than block time when settlement executed.

Fix: keep clocks synced and avoid slow work after receiving an RFQ. Live maker quotes commonly use `now_ms + 2_000`; extending expiry may reduce rejects but increases stale-price risk.

---

### `price exceeds worst_price` / `worst price exceeds limit`

The maker quote is worse than the taker's signed limit.

- Long taker: quote price must be less than or equal to `worst_price`.
- Short taker: quote price must be greater than or equal to `worst_price`.

Fix: makers should read the taker's `worst_price`, price within that bound, and use human-scale mark prices from v2 endpoints. Do not apply 1e18 scaling to mark or quote prices.

---

### `No quote was filled`

The transaction reached settlement, but every submitted quote was skipped or no submitted quote satisfied the taker's fill constraints.

Common causes:

- Quote expired.
- Quote signature failed.
- Maker subaccount nonce does not match `list_makers`.
- Maker or taker margin is insufficient.
- Quote violated `worst_price` or mark-band checks.
- `min_fill_quantity` or unfilled-action rules could not be satisfied.

Fix: inspect per-quote settlement results, then verify maker registration, subaccount nonce, balances, and exact signed fields.

---

### `unauthorized` / `authorization not found`

The required authz grant from taker or maker to the RFQ contract is missing, expired, or granted to the wrong contract address.

Fix: re-run [Authorization setup](/sdk-trading/authz), then query grants for both wallets.

---

### `insufficient funds` / `insufficient margin`

The wallet or exchange subaccount lacks INJ for gas or USDC margin.

Fix: fund the bank balance for gas and the correct exchange subaccount for margin. Maker margin must be in the registered `maker_subaccount_nonce`; if registration returns `null`, use nonce `0`.

---

### `maker not whitelisted` / `not_whitelisted`

The maker address is not registered on the RFQ contract, or a first-page `list_makers` query missed it.

Fix: use a paginated helper such as `ContractClient.is_maker_registered()`. If still absent, complete [Maker whitelist](/sdk-trading/maker-whitelist).

---

## Stream errors

### Maker stream connects but receives no RFQs

The MakerStream auth challenge was not answered or was answered with an invalid signature.

Fix: configure the maker client with `auth_private_key`, `auth_evm_chain_id`, and `auth_contract_address`. If implementing manually, sign `StreamAuthChallenge` with the same EIP-712 domain used for quotes.

---

### Taker receives no quotes

Common causes:

- Taker collects quotes for a locally generated ID instead of the ACK-returned `rfq_id`.
- Maker quote was rejected before routing.
- Maker is disconnected, not whitelisted, or not quoting that market.
- Taker connected with the wrong `request_address`.

Fix: log the request ACK, collect quotes for `ack["rfq_id"]`, and check maker-side `quote_ack` / quote error logs.

---

### `rfq_id` mismatch

Quotes arrive for a different request than the one your client expects.

Fix: treat `client_id` as client correlation only. The indexer assigns `rfq_id`; use the ACK-returned value for maker filtering, taker quote collection, and settlement.

---

## Signed-intent errors

### `invalid_intent_signature`

The signed taker intent fields differ from the submitted fields, or the EIP-712 domain is wrong.

Fix: keep `deadline_ms`, `worst_price`, `trigger_type`, `trigger_price`, `epoch`, `lane_version`, `rfq_id`, and `market_id` identical between signing and submission. Send v2 fields on the TakerStream payload.

---

### `trigger_not_satisfied`

The relayer submitted `AcceptSignedIntent`, but the contract re-read mark price and the trigger condition was no longer true.

Fix: the relayer should retry when the condition is true again, as long as the intent has not expired or been cancelled.

---

### `quote_rfq_id mismatch`

The maker quote's `rfq_id` does not match the `rfq_id` embedded in the signed taker intent.

Fix: the relayer must pair each signed intent with quotes bound to that exact `rfq_id`.

---

### `stale epoch` / `stale lane`

The taker cancelled all intents or cancelled the market lane after signing this intent.

Fix: read current `epoch` and `lane_version` before signing a replacement intent.
