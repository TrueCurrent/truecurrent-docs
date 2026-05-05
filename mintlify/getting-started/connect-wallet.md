---
title: "Connect your wallet"
description: "Connect supported wallets including Rabby, MetaMask, Keplr, Rainbow, Leap, and Phantom to TrueCurrent's non-custodial perpetuals exchange with one-time authorization setup."
updatedAt: "2026-05-04"
---

TrueCurrent is non-custodial.
You trade directly from your own wallet.
No account creation, no email, no KYC.
Wallet support and supported chains reflect the TrueCurrent integration set as of Q2 2026.

---

## Supported wallets

- **Rabby** – Recommended for most users
- **Rainbow** – Great mobile-first option
- **MetaMask** – Works with any EVM-compatible wallet
- **Keplr** – Cosmos-native wallet
- **Leap** – Cosmos wallet with broad chain support
- **Phantom** – Supported for Solana and EVM users

---

## Connecting

1. Go to [`tc.xyz`](https://tc.xyz)
2. Click **Start here** in the top right
3. Select your wallet from the list
4. Approve the connection in your wallet extension

Your address will appear in the top right when connected.

{/* TODO DR-148 add screen shot
<div class="image-placeholder">
  <img src="/img/connect-wallet.png" alt="Connect wallet screen" />

  <p><em>Connecting a wallet on TrueCurrent</em></p>
</div>
*/}

---

## One-time setup

The first time you trade, TrueCurrent will prompt you to complete a one-time authorization that allows the exchange to execute trades on your behalf.
This takes about 30 seconds and only needs to be done once per wallet.

This authorization is specific to TrueCurrent's contract and can be revoked at any time.

<Info>
Authorization is performed using `authz`.

To find out more about `authz`, for programmatic uses:
- [`authz` transaction example](https://docs.injective.network/developers-native/examples/authz)
- [`x/authz` native module documentation](https://docs.injective.network/developers-native/core/authz)
</Info>
