---
title: "The RFQ Flow"
description: "End-to-end sequence, websocket protocol, message formats"
updatedAt: "2026-04-17"
---

## End-to-End Sequence

![RFQ complete lifecycle](/img/rfq-complete-lifecycle.png)

> The complete RFQ lifecycle from subscription to settlement.

## WebSocket Protocol

The indexer exposes two gRPC-web-over-WebSocket streams.
Market makers connect to **MakerStream**;
retail users connect to **TakerStream**.
Both use the `grpc-ws` subprotocol with binary protobuf framing.

| Stream | Path | Direction | Messages |
| --- | --- | --- | --- |
| MakerStream | `/injective_rfqrpc...MakerStream`| Bidirectional | Receives: RFQ requests <br /> Sends: quotes, pings  |
| TakerStream | `/injective_rfqrpc...TakerStream` | Bidirectional | Sends: RFQ requests <br /> Receives: quotes, ACKs |

<Warning>
**Ping Required**

The server requires a ping every ~1 second to keep the connection alive.
Send a protobuf-encoded ping message (`message_type='ping'`) on a regular interval.
Failure to ping will result in the server closing the connection.
</Warning>

**gRPC-web Frame Format**:
Each message is framed as:
`[compression_flag: 1 byte][length: 4 bytes BE][protobuf payload]`.
Trailer frames (compression flag 0x80) should be ignored.

## Message Formats (JSON-RPC — Local Dev)

<Info>
**Protocol Note**

The JSON-RPC examples below apply to the local dev indexer (`ws://localhost:4464/ws`).
On testnet/mainnet, the transport is gRPC-web over WebSocket with protobuf framing
(see section on "WebSocket Protocol").
The signing logic is identical.
Only the transport layer differs.
On testnet, direction is a string ("long"/"short"), not an integer.
</Info>

**Subscribe to request stream**:

```json
{
  "jsonrpc": "2.0",
  "method": "subscribe",
  "id": 1,
  "params": {
    "query": { "stream": "request" }
  }
}
```

**Incoming RFQ request**:

```json
{
  "rfq_id": 1738000000000,
  "market_id": "0x17ef4803...",
  "direction": "long", // "long" or "short"
  "margin": "200",
  "quantity": "150",
  "worst_price": "2.8",
  "request_address": "inj1abc...",
  "expiry": 1738000300000
}
```

**Send quote**:

```json
{
  "jsonrpc": "2.0",
  "method": "quote",
  "id": 1738000000001,
  "params": {
    "quote": {
      "chain_id": "injective-888",
      "contract_address": "inj1t8hyy...",
      "rfq_id": 1738000000000,
      "market_id": "0x17ef4803...",
      "taker_direction": "long",
      "margin": "200",
      "quantity": "150",
      "price": "2.65",
      "expiry": 1738000020000,
      "maker": "inj1xyz...",
      "taker": "inj1abc...",
      "signature": "0x1a2b3c..."
    }
  }
}
```
