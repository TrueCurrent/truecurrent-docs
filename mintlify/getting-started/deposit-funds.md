---
title: "Deposit funds"
description: "Three ways to fund your TrueCurrent account: send USDC directly to your Injective address, use the wallet-connected CCTP bridge, or use the QR-code deposit flow that bridges from any chain via account abstraction."
updatedAt: "2026-05-05"
---

To trade on TrueCurrent, you need USDC in your Injective subaccount. There are three ways to get it there. Pick the one that matches where your USDC currently lives.

<Note>
Your TrueCurrent address **is** your Injective address (`inj1…`). There is no separate "TrueCurrent deposit address" for direct sends — once USDC is on Injective in your wallet, it is on TrueCurrent.

The QR-code flow ([Option C](#option-c-qr-code-bridge-deposit-from-any-chain)) is different: it generates a per-source-network deposit address that account abstraction uses to bridge USDC into your Injective wallet. That address is not your TC address — it is a one-time-use bridge endpoint.
</Note>

---

## Option A: Send USDC to your Injective address

If you already hold USDC on Injective, send it to your `inj1…` address from any wallet you control. It will appear in your TrueCurrent account immediately, with no bridging step needed.

This is the fastest path. It is also the only one that works if your starting USDC is already on Injective (e.g. moved from Helix, IBC'd in from another Cosmos chain, or already in a Cosmos wallet).

Find your Injective address in the **Portfolio** panel after connecting your wallet.

---

## Option B: Connect-wallet bridge (CCTP)

If your USDC is on Ethereum, Arbitrum, Base, or another EVM chain that Circle's CCTP supports, you can bridge it directly from the source chain into your Injective wallet without leaving TrueCurrent.

1. Go to [`tc.xyz`](https://tc.xyz) and connect the wallet that holds the source USDC
2. Click **Deposit** in the portfolio panel
3. Select the source chain
4. Enter the amount and confirm the transaction in your wallet

CCTP burns the USDC on the source chain and mints native USDC on Injective. Your USDC arrives in your TrueCurrent account within a few minutes. TrueCurrent covers Injective gas — you only pay gas on the source chain.

**When to use Option B:** you hold USDC in a wallet you can connect (MetaMask, Rabby, etc.) on a CCTP-supported chain.

[*Screenshot slot: Option B — Deposit panel after clicking Deposit, showing the source-chain selector and amount input.*]

---

## Option C: QR-code bridge (deposit from any chain)

If the USDC you want to deposit is somewhere you can't connect a wallet to TrueCurrent — for example, sitting on a centralized exchange that doesn't support Injective USDC withdrawals — use the QR-code deposit flow. This is the bridging option that gets the most "wait, that's actually clever" reactions, so it is worth understanding before you need it.

How it works:

1. In the Deposit panel, choose **QR-code deposit** and select the source network (e.g. Ethereum, Arbitrum, Base)
2. TrueCurrent generates a deposit address on that source network — shown as a QR code and a copy-pastable address
3. Send USDC to that address from anywhere — your CEX withdraw page, another wallet, a friend, anywhere that can send USDC on the chosen source network
4. Account abstraction handles the bridging behind the scenes; your USDC arrives in your TrueCurrent account in a few minutes

The deposit address is generated specifically for *your* TrueCurrent account on the source network you selected. Anything sent to it ends up in your Injective wallet. You don't need to connect the source wallet, sign anything from it, or hold a CCTP-supported wallet on the source chain.

**When to use Option C:**
- Your USDC is on a CEX that doesn't natively support Injective USDC withdrawals — paste the QR-code address into the CEX's "withdraw" form and you're done
- Your USDC is in a wallet on a chain TrueCurrent supports for bridging but that doesn't have a connector you want to install
- You want to deposit from cold storage without exposing the private key to a connected dapp

<Warning>
Send only USDC, only on the source network you selected. Sending a different asset, or sending USDC on a different network, will not be recoverable by TrueCurrent.
</Warning>

[*Screenshot slot: Option C — QR-code panel with source-network selector visible and a QR code rendered for one selected network.*]

---

## Withdrawing funds

To move funds out of TrueCurrent:

1. Go to **Portfolio** → **Withdraw**
2. Select the destination chain and amount
3. Confirm — funds arrive in your external wallet promptly

Margin locked in open positions isn't available to withdraw until those positions are closed.

---

## No minimums, no deposit fees

There is no protocol minimum deposit and no fee to move funds in or out of TrueCurrent. You only ever pay source-chain gas for Option B (the connect-wallet CCTP bridge). Option A is whatever your sending wallet charges. Option C is whatever the source venue charges to withdraw.
