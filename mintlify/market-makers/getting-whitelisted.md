---
title: "Getting whitelisted"
description: "Requirements and application process for joining TrueCurrent's market maker whitelist, including testnet integration, capital requirements, technical prerequisites, and performance expectations for RFQ liquidity providers."
updatedAt: "2026-04-06"
---

TrueCurrent uses an approved market maker whitelist. Only whitelisted addresses can submit quotes to the MakerStream. This section explains the requirements and process for getting added.

---

## Why a whitelist?

The whitelist ensures that every market maker on TrueCurrent is:

- Technically integrated and able to respond reliably within the MM response window (a few hundred milliseconds)
- Capitalized sufficiently to back their quotes
- Committed to quoting in good faith (avoiding quote manipulation or gaming)

A reliable maker pool produces better outcomes for traders – consistent fills, tight spreads, and no quote degradation.

---

## Requirements

Before applying, you should have:

- An Injective wallet with adequate USDC margin
- A functional automated quoting system connected to TrueCurrent's MakerStream
- Implemented the quote signing protocol (see [Signing quotes](/market-makers/signing-quotes))
- Completed the `authz` setup (see [Authorization setup](/market-makers/authz-setup))
- Successfully run the E2E flow on testnet

---

## Testnet testing

Before going live, test your full integration on TrueCurrent's testnet:

1. Set `RFQ_ENV=testnet` in your configuration
2. Fund your testnet wallet with testnet INJ and USDC (available from Injective testnet faucet)
3. Grant `authz` permissions to the testnet contract address
4. Connect your MakerStream to the testnet indexer endpoint
5. Run through the complete quote and acceptance flow to confirm your signing and settlement logic is correct

Once your testnet integration is verified end-to-end, you're ready to apply.

---

## Application process

To apply for market maker whitelist access:

1. Contact the TrueCurrent team at *(contact info TBD)* or via Discord/Telegram *(links TBD)*
2. Provide your Injective maker wallet address
3. Share your testnet test results or a brief description of your quoting infrastructure
4. The team will review and add your address to the whitelist

Once whitelisted, your wallet address can submit quotes to the production MakerStream and those quotes will be routed to traders.

---

## Staying in good standing

Your market maker status depends on maintaining reliable, competitive quoting behavior. TrueCurrent monitors maker performance and may remove makers from the whitelist if:

- Response rate falls below an acceptable threshold (consistently failing to quote within the response window)
- Quotes are systematically off-market (quoting prices that never win and appear designed to game metrics)
- Settlements consistently fail due to insufficient margin
- Any behavior that degrades the experience for traders

This is not about penalizing normal market making decisions (like widening spreads in volatile markets) – it's about maintaining a healthy, reliable RFQ pool.
