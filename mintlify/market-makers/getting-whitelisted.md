---
title: "Getting whitelisted"
description: "Requirements and application process for joining TrueCurrent's maker whitelist, including testnet integration, capital requirements, technical prerequisites, and performance expectations for RFQ liquidity providers."
updatedAt: "2026-05-05"
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
- Implemented the quote signing protocol (see [Signing quotes](/market-makers/signing-quotes))
- Completed the `authz` setup (see [Authorization setup](/market-makers/authz-setup))
- Successfully run the E2E flow on testnet

---

## Testnet testing

Before going live, test your full integration on TrueCurrent's testnet:

1. Set `RFQ_ENV=testnet` in your configuration
2. Fund your testnet wallet with testnet INJ and USDC for the current RFQ market.
   - Use the testnet faucet at `https://testnet-faucet.injective.dev`.
   - It dispenses **500 USDC + 2 INJ** per address, once per 24 hours.
   - The USDC denom on testnet is `erc20:0x0C382e685bbeeFE5d3d9C29e29E341fEE8E84C5d`.
3. Grant `authz` permissions to the testnet contract address
4. Connect your MakerStream to the testnet indexer endpoint
5. Run through the complete quote and acceptance flow to confirm your signing and settlement logic is correct

Once your testnet integration is verified end-to-end, you're ready to apply.

---

## Application process

To apply for maker whitelist access:

1. Reach out via the channels listed on the [Support](/support) page (Discord, Telegram, or `support@tc.xyz`).
2. Provide your Injective maker wallet address.
3. Share your testnet test results or a brief description of your quoting infrastructure.
4. The team will review and add your address to the whitelist.

Once whitelisted, your wallet address can submit quotes to the production MakerStream and those quotes will be routed to traders.

**Verifying whitelist status:**
You can confirm at any point by querying `list_makers` on the contract.
Note that `list_makers` returns at most 20 makers per page.
If your address sorts past page 1,
a simple single-page query will report "not registered" even when you are.
Either paginate with `start_after`,
or use `ContractClient.is_maker_registered()` which handles pagination automatically.

---

## Staying in good standing

Your maker status depends on maintaining reliable, competitive quoting behavior. TrueCurrent monitors maker performance and may remove makers from the whitelist if:

- Response rate falls below an acceptable threshold (consistently failing to quote within 2 seconds)
- Quotes are systematically off-market (quoting prices that never win and appear designed to game metrics)
- Settlements consistently fail due to insufficient margin
- Any behavior that degrades the experience for traders

This is not about penalizing normal maker discretion, such as widening spreads in volatile markets. It is about maintaining a healthy, reliable RFQ pool.
