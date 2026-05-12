---
title: "MakerStream (legacy pointer)"
description: "Legacy pointer; the canonical TrueCurrent MakerStream WebSocket reference, including connection, auth handshake, and reconnection, now lives in SDK trading."
updatedAt: "2026-05-05"
---

The canonical MakerStream documentation — connection details, the EIP-712 v2 auth challenge handshake, the request and quote message schemas, and reconnection patterns — lives in the [Makers integration guide](/market-makers/integration/index):

- [Connecting to the RFQ stream](/market-makers/integration/connecting) — endpoint, gRPC-web framing, ping cadence, and the `MakerChallenge` / `MakerAuth` handshake
- [Receiving RFQ requests](/market-makers/integration/rfq-requests) — request fields and how to interpret them
- [Building & signing quotes](/market-makers/integration/rfq-quotes-sign) — EIP-712 v2 `SignQuote` schema and the `sign_quote_v2` helper
- [Sending quotes](/market-makers/integration/rfq-quotes-send) — wire format, required fields, and ACK / error responses
- [TP/SL liquidity](/market-makers/integration/rfq-quotes-blind) — how exit RFQs flow through the same stream
- [Testnet configuration](/market-makers/integration/testnet-configuration) — endpoints, contract address, EVM chain ID

