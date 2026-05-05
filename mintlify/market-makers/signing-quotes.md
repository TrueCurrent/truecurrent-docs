---
title: "Signing quotes"
description: "Pointer page: the canonical EIP-712 v2 quote signing reference lives in the integration guide. The SignQuote schema, the sign_quote_v2 helper, and the common signing failures are all there."
updatedAt: "2026-05-05"
---

The canonical quote-signing reference — EIP-712 v2 domain, the `SignQuote` typed-data schema with field order, the `sign_quote_v2` helper, decimal-formatting rules, and common signing failures — lives in the [Market makers integration guide](/market-makers/integration/rfq-quotes-sign).

Quick links:

- [Building & signing quotes](/market-makers/integration/rfq-quotes-sign) — `SignQuote` schema, Python helper, decimal hygiene
- [Sending quotes](/market-makers/integration/rfq-quotes-send) — wire format, ACK / error responses
- [TP/SL liquidity](/market-makers/integration/rfq-quotes-blind) — exit-RFQ signing path
- [Connecting to the RFQ stream](/market-makers/integration/connecting) — auth handshake (also EIP-712 v2)
- [Testnet configuration](/market-makers/integration/testnet-configuration) — EVM chain IDs (`1439` testnet, `1776` mainnet) and contract address
