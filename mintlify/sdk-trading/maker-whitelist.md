---
title: "Maker whitelist"
description: "Requirements and application process for joining the TrueCurrent maker whitelist: testnet integration, capital, technical prerequisites, and performance."
updatedAt: "2026-05-06"
---

TrueCurrent uses an approved maker whitelist. Only whitelisted addresses can submit quotes to the MakerStream. This section explains the requirements and process for getting added.

---

## Why a whitelist?

The whitelist ensures that every maker on TrueCurrent is:

- Technically integrated and able to respond reliably within the 2-second window
- Capitalized sufficiently to back their quotes
- Committed to quoting in good faith (avoiding quote manipulation or gaming)

A reliable maker pool produces better outcomes for traders – consistent fills, tight spreads, and no quote degradation.

---

## Requirements

Before applying, you should have:

- An Injective wallet with adequate USDC margin for the current testnet market
- A functional automated quoting system connected to TrueCurrent's MakerStream
- Implemented the quote signing protocol (see [Signing quotes](/sdk-trading/signing-quotes))
- Completed the `authz` setup (see [Authorization setup](/sdk-trading/authz))
- Successfully run the E2E flow on testnet

---

## Testnet testing

Before going live, test your full integration on TrueCurrent's testnet:

1. Set `RFQ_ENV=testnet` in your configuration
2. Fund your testnet wallet with testnet INJ and USDC for the current RFQ market.
   - Use the testnet faucet at `https://testnet-faucet.injective.dev`.
   - It dispenses **500 USDC \+ 2 INJ** per address, once per 24 hours.
   - The USDC denom on testnet is `erc20:0x0C382e685bbeeFE5d3d9C29e29E341fEE8E84C5d`.
3. Grant `authz` permissions to the testnet contract address
4. Connect your MakerStream to the testnet indexer endpoint
5. Run through the complete quote and acceptance flow to confirm your signing and settlement logic is correct

Once your testnet integration is verified end-to-end, you're ready to apply.
