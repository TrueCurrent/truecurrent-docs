---
title: "Quick start checklist"
description: "Step-by-step onboarding checklist for market makers integrating with TrueCurrent's RFQ system, from wallet generation to sending your first signed quote."
updatedAt: "2026-05-01"
---

| Step | Who | Description |
|---|---|---|
| 1 | You | Generate an Injective wallet (secp256k1). |
| 2 | You | Share your `inj1...` address with the TrueCurrent team. |
| 3 | Admin | Whitelists your address on the RFQ contract (`register_maker`). |
| 4 | You | Grant authz permissions to the RFQ contract (one-time tx). |
| 5 | You | Fund your exchange subaccount with quote-asset margin. On the current testnet RFQ market, this is USDC. |
| 6 | You | Connect to MakerStream. |
| 7 | You | Receive requests, sign quotes, send quotes. |
