---
title: "FAQ & troubleshooting"
description: "Frequently asked questions and troubleshooting guide for TrueCurrent market maker integration: settlement mechanics, quote expiry, partial fills, connection issues, and TP/SL participation."
updatedAt: "2026-05-05"
---

<AccordionGroup>
  <Accordion title="Do I submit an on-chain transaction for every trade?">
    No.
    Quotes are signed off-chain and the retail user submits the on-chain transaction.

    You sign quotes off-chain.
    The retail user submits `AcceptQuote`.
    The contract verifies your signature.
  </Accordion>

  <Accordion title="What if my quote expires before the taker accepts?">
    The quote will not go on chain, as both maker and taker submit their intents off chain.

    The quote only goes on chain when the order is matched.
  </Accordion>

  <Accordion title="Can I quote a different quantity than requested?">
    Yes.
    Partial fills are supported by setting `maker_quantity` / `maker_margin` to your intended amounts.

    Any quantity not covered by the chosen quotes is simply not traded — the taker receives a partial fill or a no-fill, and their margin for the unfilled portion is released.
  </Accordion>

  <Accordion title="What if I don't respond to a quote request?">
    No response means no trade and no penalty,
    but your response-rate metric does affect standing.
  </Accordion>

  <Accordion title="How are positions settled?">
    Synthetic positions are created on the Injective exchange module between your subaccount and the taker's, with real margin, PnL, and liquidation.
  </Accordion>

  <Accordion title="Why aren't my quotes reaching takers?">
    Check whitelist registration, auth challenge handling, authz grants,
    signature validity, and quote expiry in that order.

    1. **Whitelisted?** Query `list_makers`.
       **Pagination caveat:** `list_makers` returns at most 20 makers per page.
       If your address sorts past the first 20,
       the default query returns "not registered" even when you are.
       Page through with `start_after`,
       or use `ContractClient.is_maker_registered()` which paginates automatically.
    2. **Auth challenge handled?**
       Stream connects, pings return pongs,
       but no `request` events.
       This means your `MakerChallenge` handler is missing or produced a wrong signature.
       See [Auth handshake](/market-makers/integration/connecting#auth-handshake).
    3. **Authz granted?** See §1.3.
    4. **Signature verifies?**
       Check `evmChainId` field #1 in `SignQuote`, field order,
       and that `rfq_id` is a JSON number.
    5. **Quote expired?** Verify `expiry` (`now + 2s` for live quotes) and clock sync.
  </Accordion>

  <Accordion title="What is the competition window?">
    After a request is sent, the indexer collects quotes for ~2 seconds before delivering the best one to the taker.

    Best price wins: lowest quoted price for long positions, highest for short positions.
  </Accordion>

  <Accordion title="What do I do when my connection drops?">
    Confirm ping every ~1 second, use the `grpc-ws` subprotocol,
    and reconnect with exponential backoff.

    1. Confirm ping every ~1s.
    2. Use the `grpc-ws` subprotocol.
    3. Reconnect with exponential backoff.
  </Accordion>

  <Accordion title="What changes for TP/SL orders?">
    Nothing changes on your send path for live RFQs;
    TP/SL participation is opt-in and carries no reputational cost for non-response.

    If another maker has coverage at an acceptable price,
    the protocol settles without you.
    Not responding carries no reputational cost for TP/SL events.
    To participate in pre-posted blind quotes,
    see [TP/SL liquidity](/market-makers/integration/rfq-quotes-blind).
  </Accordion>

  <Accordion title="What causes the worst_price exceeds limit error?">
    Use the value directly from the v2 endpoint.
    The v2 indexer returns mark price in human-readable scale (e.g. `14.85`),
    not 1e18-scaled.

    If you read mark price as `14850000000000000000` and use that as your reference,
    your quoted price will be far outside the taker's `worst_price`.
    Use the mark price value directly from the v2 endpoint.
  </Accordion>

  <Accordion title="How do I submit reduce-only quotes?">
    Pass `margin: "0"` on the quote.
    No separate flag is needed;
    the protocol enforces that only fills closing or shrinking existing exposure are valid.
  </Accordion>
</AccordionGroup>

{/* TODO: add related docs - e2e testnet run book, TPSL flow, RFQ/orderbook/arb flow, admin ops of RFQ contract */}
