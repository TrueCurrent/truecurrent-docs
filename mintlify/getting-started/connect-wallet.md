---
title: "Connect your wallet"
description: "Connect a supported wallet to TrueCurrent and complete the one-time authorization required for non-custodial trading."
updatedAt: "2026-05-05"
---

Your wallet is your TrueCurrent account. Connect a supported wallet, complete the one-time authorization, and trade directly from your own address – no account creation.

---

## Supported wallets

- **Rabby** – Recommended for most users
- **Rainbow** – Great mobile-first option
- **MetaMask** – Works with any EVM-compatible wallet
- **Keplr** – Cosmos-native wallet

---

## Connecting

1. Go to [`tc.xyz`](https://tc.xyz)
2. Click **Start here** in the top right
3. Select your wallet from the list
4. Approve the connection in your wallet extension

Your address will appear in the top right when connected.

<video src="/videos/Clipboard-20260505-215724-873-1.mp4" controls />

{/* SCREENSHOT SLOT: connect-wallet — wallet picker modal opened from the top-right "Start here" button, showing the four supported wallets. */}

---

## One-time setup

The first time you trade, TrueCurrent asks you to approve a one-time `authz` authorization. This lets TrueCurrent execute the specific trading actions needed for settlement without taking custody of your funds.

The authorization is scoped to TrueCurrent, only needs to be completed once per wallet, and can be revoked later.

<Info>
  Authorization is performed using `authz`.

  To find out more about `authz`, for programmatic uses:

  - [`authz` transaction example](https://docs.injective.network/developers-native/examples/authz)
  - [`x/authz` native module documentation](https://docs.injective.network/developers-native/core/authz)
</Info>