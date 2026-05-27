---
title: "Troubleshooting"
description: "Failure-mode guide for TrueCurrent RFQ maker, taker, signing, settlement, and TP/SL integrations."
updatedAt: "2026-05-27"
---

Debug RFQ issues in this order:

1. Config: endpoint, contract, Cosmos chain ID, EIP-712 chain ID.
2. Maker readiness: whitelist, subaccount nonce, margin, authz.
3. Stream health: service path, ping/pong, MakerStream auth challenge.
4. Signing: exact field order and exact decimal strings.
5. Settlement: expiry, `worst_price`, margin, authz, market rules.

The [Testnet runbook](/sdk-trading/runbook) walks through the same order with commands.

---

## Connectivity and stream auth

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| WebSocket `503` or immediate close | Wrong public service path | Use `wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC/MakerStream` or `/TakerStream`. |
| Stream connects and pongs but maker receives no requests | Maker did not answer `MakerChallenge` | Configure `MakerStreamClient` with `auth_private_key`, `auth_evm_chain_id`, and `auth_contract_address`. |
| Stream closes after auth reply | Challenge signature mismatch | Sign `StreamAuthChallenge` with the same EIP-712 domain used for quotes; use raw challenge nonce bytes and reply before expiry. |
| Taker receives no quote messages | Collecting with wrong `rfq_id` or wrong `request_address` | Use the ACK-returned `rfq_id`; connect TakerStream with the taker's Injective address. |
| Maker sees unrelated requests | MakerStream broadcasts eligible flow to whitelisted makers | Filter by the target `rfq_id`, market, and taker before signing. |

---

## Whitelist, authz, and balances

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Maker is "not registered" but admin says it is | First-page `list_makers` query missed the address | Page through `list_makers` with `start_after` or use `ContractClient.is_maker_registered()`. |
| `not_whitelisted` | Maker address not registered on the RFQ contract | Complete [Maker whitelist](/sdk-trading/maker-whitelist), then re-query. |
| `authorization not found` or `unauthorized` | Missing or expired authz grant | Re-run [Authorization setup](/sdk-trading/authz); verify the RFQ contract is the grantee. |
| `account sequence mismatch` while granting authz | Multiple grant transactions broadcast too quickly | Submit grants sequentially and wait around 3 seconds between broadcasts. |
| `insufficient funds` | Bank gas balance or exchange subaccount margin is too low | Fund INJ for gas and USDC margin in the settlement subaccount. |
| `No quote was filled` with otherwise valid quote | Maker subaccount nonce mismatch | Query `list_makers` and use its registered `subaccount_nonce`; if it is `null`, use `0`. |

---

## Signing and quote validation

| Error / symptom | Likely cause | Fix |
| --- | --- | --- |
| `invalid_signature` | Signed fields differ from sent fields | Log the typed-data message and wire payload side by side; every signed decimal string must match exactly. |
| Signature rejected after adding `evm_chain_id` | EVM chain ID placed in `quote.chain_id` or wrong domain | Use `quote.chain_id="injective-888"` on testnet and `evm_chain_id=1439` for EIP-712. |
| `quote.chain_id` is `1439` or `1776` | `chainId` from the EIP-712 domain was copied into the quote wire field | Put the numeric ID in `quote.evm_chain_id`; put `injective-888` or `injective-1` in `quote.chain_id`. |
| Signature works locally but fails at settlement | Wrong contract address in the EIP-712 domain | Use the active RFQ contract address from [Testnet configuration](/sdk-trading/testnet-configuration). |
| BTC quotes fail tick or signature checks | Integer-tick price sent as `"76462.0"` | Canonicalize integer ticks as `"76462"` before signing and sending. |
| Quote expires before settlement | Expiry too close, clock drift, or slow post-RFQ work | Use NTP and avoid slow calls after RFQ receipt. Live quotes commonly use `now_ms + 2_000`. |
| `worst price exceeds limit` | Price or mark/index value read with the wrong scale | Use human-scale prices returned by v2 endpoints; do not apply 1e18 scaling. |
| Retail collects zero quotes | Maker quote was rejected, expired, or for a different `rfq_id` | Check maker logs for `quote_ack`, quote error responses, and `rfq_id` correlation. |

---

## Settlement behavior

<AccordionGroup>
  <Accordion title="Do makers submit an onchain transaction for every trade?">
    No. Makers sign quotes offchain and send them to the indexer. The taker submits `AcceptQuote`; the contract verifies the maker signature and settles atomically if the quote is still valid.
  </Accordion>

  <Accordion title="Does quote_ack mean the taker filled my quote?">
    No. `quote_ack.status="success"` only means the indexer accepted and routed the quote. A fill is confirmed by onchain settlement or by a maker-side `settlement_update`.
  </Accordion>

  <Accordion title="Can the taker partially fill my quote?">
    Yes. The taker can submit multiple quotes, and the contract may skip invalid quotes while filling valid ones. Your quote also controls `maker_quantity`, `maker_margin`, and optional `min_fill_quantity`.
  </Accordion>

  <Accordion title="What happens if I do not respond to an RFQ?">
    No quote means no trade for you and no onchain penalty. Response-rate metrics may still affect maker standing.
  </Accordion>

  <Accordion title="How are positions represented after settlement?">
    Settlement creates or updates derivative positions in the Injective exchange module for both parties' subaccounts. Margin, PnL, liquidation, and funding then follow exchange-module rules.
  </Accordion>

  <Accordion title="How do reduce-only quotes work?">
    Send `margin: "0"`. No separate reduce-only flag is required; zero margin means the fill can only close or shrink existing exposure.
  </Accordion>
</AccordionGroup>

---

## Signed-intent and TP/SL failures

| Error / symptom | Likely cause | Fix |
| --- | --- | --- |
| TakerStream rejects conditional order | Missing v2 fields | Send `conditional_order_sign_mode="v2"` and `conditional_order_evm_chain_id`; the toolkit sets them when `sign_mode` and `evm_chain_id` are passed. |
| `invalid_intent_signature` | Signed intent fields differ from submitted fields | Keep `deadline_ms`, `worst_price`, `trigger_type`, `trigger_price`, `epoch`, and `lane_version` identical between signing and submission. |
| `trigger_not_satisfied` | Mark-price condition was not true at execution | Re-check trigger type and trigger price against current mark price. |
| `intent expired` | `deadline_ms` passed before relay | Submit a fresh intent with a new deadline. |
| `quote_rfq_id mismatch` | Relayer paired the intent with a quote for another RFQ | Re-query maker liquidity for the exact `rfq_id` embedded in the signed intent. |
| Intent was cancelled but still submitted | `epoch` or `lane_version` changed | Read current counters before signing new intents after `CancelAllIntents` or `CancelIntentLane`. |
| Conditional order accepted but never fires | Trigger not reached, no relayer, or no executable maker quote | Check trigger source, relay logs, and maker quote availability. |

---

## Fast triage checklist

- Check [Testnet configuration](/sdk-trading/testnet-configuration) values first.
- Confirm maker whitelist with a paginated helper, not only the first `list_makers` page.
- Confirm authz grants for both maker and taker.
- Confirm maker subaccount nonce and USDC margin.
- Confirm MakerStream auth challenge is answered before waiting for RFQs.
- Confirm quotes use `sign_mode="v2"` and `evm_chain_id=1439` on testnet.
- Confirm `quote.chain_id` / `quote.chainId` remains `injective-888`, not `1439`.
- Confirm the taker collects quotes for the ACK-returned `rfq_id`.
- Confirm quote expiry is still in the future at settlement time.
- Confirm `worst_price`, tick size, and signed decimal strings all match the selected market.
