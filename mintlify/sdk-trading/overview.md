---
title: "Programmatic RFQ trading with the TrueCurrent SDK"
description: "Trade or make markets on TrueCurrent programmatically: pick a taker or maker integration path, learn the RFQ lifecycle, and prepare what is needed to go live."
updatedAt: "2026-05-06"
---

TrueCurrent can be used from the web app or through an SDK integration. The SDK path is for teams that want to automate trading, embed RFQ execution in another product, or run institutional liquidity-provider infrastructure.

There are two integration roles:

| Role | What you do | Best for |
| --- | --- | --- |
| **Taker** | Request quotes, choose the best executable price, and settle a trade onchain | Trading bots, execution desks, strategy engines, apps embedding TrueCurrent trading |
| **Maker** | Receive RFQ requests, price them, sign quotes, and manage inventory | Professional liquidity providers and maker systems |

Most traders are takers. Makers must be approved before their quotes are routed to users.

---

## How SDK trading fits into TrueCurrent

SDK trading uses the same RFQ model as the web app:

1. A taker requests a trade
2. The request is routed to active makers
3. Makers respond with signed quotes
4. The taker selects the best quote or quote set
5. The TrueCurrent contract verifies the quote and settles both sides on Injective

The SDK does not create a different execution venue. It gives you programmatic access to the same quote discovery and settlement flow.

---

## What the SDK handles

The SDK and reference clients handle the repetitive parts of the RFQ flow so your system can focus on strategy, risk, and user experience.

For takers, the SDK path covers:

- Connecting to TakerStream
- Submitting RFQ requests
- Waiting for the indexer-assigned `rfq_id`
- Collecting quotes
- Preparing `AcceptQuote`
- Submitting settlement transactions
- Creating and cancelling signed TP/SL intents

For makers, the SDK path covers:

- Connecting to MakerStream
- Completing the maker auth challenge
- Receiving RFQ requests
- Signing EIP-712 v2 quotes
- Sending quotes with the required wire fields
- Receiving quote and settlement updates

---

## Before you integrate

Every SDK integration needs:

- An Injective wallet dedicated to SDK trading
- INJ for gas
- USDC margin in the relevant exchange subaccount
- Network connectivity to the TrueCurrent indexer
- The current RFQ contract address and market configuration
- A plan for authz grant creation, monitoring, and revocation

Makers also need:

- Whitelist approval
- Reliable price feeds
- A quoting model
- Inventory limits
- Automated balance and connection monitoring

---

## Which page should you read?

Use the taker page if you are building a trading bot, strategy runner, vault, app integration, or execution tool.

Use the maker page if you will provide liquidity to other traders and respond to RFQ requests with signed quotes.

<CardGroup cols={2}>
  <Card title="Trade as a taker" icon="code" href="/sdk-trading/takers">
    Build programmatic trading: request quotes, select execution, settle onchain, and manage TP/SL intents.
  </Card>

  <Card title="Make markets" icon="chart-candlestick" href="/sdk-trading/makers">
    Build a maker integration: connect to MakerStream, sign quotes, manage inventory, and monitor fills.
  </Card>
  <Card title="Set up authz" icon="key" href="/sdk-trading/authz">
    Grant the RFQ contract the narrow permissions required for taker and maker settlement.
  </Card>
  <Card title="Run testnet E2E" icon="terminal" href="/sdk-trading/runbook">
    Validate wallets, balances, grants, MakerStream auth, AcceptQuote, and TP/SL signed intents.
  </Card>
</CardGroup>

---

## Important mental model

The indexer routes messages. The contract enforces settlement.

That distinction matters. SDK clients use the indexer to discover quotes, but all final checks happen onchain: maker signature, quote expiry, `worst_price`, mark-price band, maker registration, and available margin. If a quote does not satisfy the contract, it cannot settle.

For protocol details, see [How RFQ works](/technical/how-rfq-works).