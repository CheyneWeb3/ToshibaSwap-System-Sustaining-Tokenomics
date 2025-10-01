# PotatoToken — Setup & Rewards Guide

## What this token does

* **Fees on buys/sells/transfers** are taken into the token contract.
* When selling pressure triggers a swap, the contract **swaps collected tokens → ETH**.
* That ETH is split into:

  * **Dividends** → sent to the **DividendTracker** (pays holders)
  * **Marketing** and **Donation** payouts → sent to their wallets
* The **DividendTracker** pays **ETH by default**, or **any ERC-20** you choose later by setting a **payout token** (it swaps ETH→token via the router before distributing).

---

## 0) Prereqs & constants you shipped

* **Router (Base)**: `0x4752ba5DBc23f44D87826276BF6Fd6b1C372aD24` (already wired in constructor)
* **Initial wallets:**

  * Marketing: `0x51a3AE19db386ea16CEa8D5452E8374f8d5eE775`
  * Donation:  `0x784590d1553aA23152411442d0eCE23AF02Ac8b3`
* **Supply**: 100,000,000 × 1e18 minted to owner
* **Pair**: created in the constructor against `WETH()` of the router
* **Trading**: initially **disabled** (must call `enableTrading()` once ready)
* **Dividends**: default **ETH** (on Base) because `defaultToken == address(0)`

---

## 1) Deploy

You can deploy straight from a script, Foundry, or Hardhat. After deploy, **you own the main token** (EOA owner). The main token created the **DividendTracker**, and is set as **tracker owner** internally, so all tracker admin calls must go **through the token**.

> Tip: Verify both contracts on Basescan after deployment (the tracker is created by the token; verify it via “Similar contracts” or by publishing the flattened source with correct constructor args).

---

## 2) One-time initialization checklist

Run these (where applicable) **before turning trading on**:

1. **(Optional) Update wallets**

```solidity
setmarketingWallet(<marketingWallet>);
setdonationWallet(<donationWallet>);
```

2. **(Optional) Exclusions**

* From **fees**: marketing, donation, the token itself, tracker are already excluded.
* You can exclude others:

```solidity
setExcludeFees(<addr>, true);
```

* From **dividends** (never receive rewards):

```solidity
setExcludeDividends(<addr>);
```

* Include back later (re-enable rewards and set balance snapshot):

```solidity
setIncludeDividends(<addr>);
// sync their balance into tracker
dividendTracker.setBalance(<addr>, balanceOf(<addr>));
```

3. **(Optional) Max wallet**

> Minimum allowed is 0.1% of supply. The setter takes “whole tokens” and multiplies by 1e18 for you.

```solidity
setmaxWallet(1_000_000); // example: 1,000,000 tokens
```

4. **(Optional) Rewards trigger threshold**

> Also in “whole tokens” (multiplied by 1e18 internally).

```solidity
setSwapTriggerAmount(50_000); // example: 50k tokens to trigger swap/distribute on sells
```

5. **(Optional) Fee schedule**

> Caps: **Buy total ≤ 8%**, **Sell total ≤ 8%**, **Transfer ≤ 5%**.

```solidity
updateFees(
  /* marketingBuy   */ 2,
  /* marketingSell  */ 2,
  /* rewardsBuy     */ 2,
  /* rewardsSell    */ 2,
  /* donationBuy    */ 2,
  /* donationSell   */ 2
);

updateTransferFee(0); // 0–5, applies only to wallet→wallet (non-AMM) transfers
```

6. **(Optional) Gas & automation knobs**

```solidity
updateGasForProcessing(300000);        // 200k–2M allowed
setAllowCustomTokens(true);            // tracker toggle (kept for compatibility)
setAllowAutoReinvest(false);           // allow tracker auto-reinvest if you want that feature
```

7. **Enable trading (irrevocable)**

```solidity
enableTrading();
```

---

## 3) Funding liquidity & going live

* The pair was already created in the constructor. Provide liquidity with the router (V2-style).
* Once there’s trading, fees accrue; on **sells**, if the contract balance ≥ `swapTokensAtAmount`, it will:

  * swap tokens for ETH,
  * send ETH portions to **DividendTracker** + **marketing** + **donation**,
  * and **process holders** (auto-claim loop subject to `gasForProcessing`).

> If you need to **force** a distribution without waiting for sells:

```solidity
forceSwapAndSendDividends(100_000); // takes “whole tokens”; internally *1e18
```

---

## 4) Choosing what asset holders receive (rewards token)

### Default (no action): **ETH dividends**

* `defaultToken == address(0)` (already set in tracker), so claims pay **ETH**.

### Switch to an ERC-20 rewards token (any time, post-deploy)

* Call ***main token*** method (it forwards to tracker):

```solidity
updatePayoutToken(<ERC20_TOKEN_ADDRESS>);
```

**What happens afterward:**

* When tracker pays a user, it swaps **ETH → ERC-20** via the router path `[WETH, TOKEN]` and sends the **token** to the user.
* If the swap fails for any reason, the claim reverts internally and is retried later (user keeps accruals).

### Switch back to ETH later

```solidity
updatePayoutToken(address(0));
```

> **Router path note:** Your tracker uses a 2-hop path `[WETH, token]`. For tokens that require a different path (e.g., via USDbC), keep the rewards token to something liquid on the router with a **WETH direct pool** for smooth payouts.

---

## 5) Post-deploy tuning (dividend thresholds & user toggles)

Owner-controlled minimums (in **whole tokens**, multiplied by 1e18 inside tracker):

```solidity
setMinimumTokenBalanceForAutoDividends(1); // need ≥ this to be in the auto-claim map
setMinimumTokenBalanceForDividends(1);     // need ≥ this to be eligible at all
```

End-user toggles (any holder can call through the main token; the main token relays as tracker-owner):

```solidity
setAutoClaim(true | false); // opts the caller out/in of auto-claim (gas loop)
setReinvest(true | false);  // opts the caller to auto-reinvest into MASH (if allowAutoReinvest=true)
claim();                    // manual claim now
```

Admin pause:

```solidity
setDividendsPaused(true | false);
```

---

## 6) Reading state (useful quick checks)

```solidity
getPayoutToken()                              // address(0) = ETH, else ERC-20 address
getTotalDividendsDistributed()                // cumulative ETH sent to tracker
withdrawableDividendOf(<account>)             // claimable now
dividendTokenBalanceOf(<account>)             // tracker-side balance snapshot
getNumberOfDividendTokenHolders()             // count currently in auto-claim map
getLastProcessedIndex()                       // where the loop is up to
getAccountDividendsInfo(<account>)            // (acct, index, itersUntilProcessed, withdrawable, total, lastClaimTs)
```

---

## 7) Examples — calling from scripts

### Ethers v6 (Node)

```ts
import { ethers } from "ethers";

const provider = new ethers.JsonRpcProvider(process.env.RPC);
const wallet   = new ethers.Wallet(process.env.PRIVKEY!, provider);
const token    = new ethers.Contract(process.env.TOKEN!, [
  "function enableTrading() external",
  "function updateFees(uint256,uint256,uint256,uint256,uint256,uint256) external",
  "function updateTransferFee(uint256) external",
  "function setSwapTriggerAmount(uint256) external",
  "function setmaxWallet(uint256) external",
  "function updatePayoutToken(address) external",
  "function forceSwapAndSendDividends(uint256) external",
  "function claim() external",
  "function setAutoClaim(bool) external",
  "function setReinvest(bool) external",
], wallet);

await token.updateFees(2,2,2,2,2,2);
await token.setSwapTriggerAmount(50_000);
await token.setmaxWallet(1_000_000);
await token.updatePayoutToken("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"); // example
await token.enableTrading();

// later:
await token.updatePayoutToken(ethers.ZeroAddress); // back to ETH
```

### Foundry (cast)

```bash
cast send $TOKEN "updatePayoutToken(address)" 0x0000000000000000000000000000000000000000 --private-key $PK
cast call $TOKEN "getPayoutToken()(address)"
```

---

## 8) Common gotchas & fixes

* **Trading is off:** Any non-owner wallet cannot transfer until `enableTrading()`.
* **No distributions happening:**

  * Ensure sells occur **after** contract balance ≥ `swapTokensAtAmount`.
  * Or call `forceSwapAndSendDividends(tokens)` to push one through.
* **Max wallet reverts on buys:** Increase `setmaxWallet(...)` or buy smaller sizes.
* **Fees exceed limits:** `updateFees` enforces **≤8%** per side; `updateTransferFee` enforces **≤5%**.
* **Dead address burns:** Sending to `DEAD` triggers `_burn` (no fees/dividends on that transfer).
* **Router liquidity path:** Rewards token must have a **WETH pool** on the configured router.

---

## 9) Role & security notes

* **onlyOwner** on main token is your EOA (or a multisig you hand ownership to).
* The **DividendTracker owner** is **the token contract**, so you always configure tracker via the **main token’s** functions.
* You can hand the token’s ownership to a multisig with:

```solidity
transferAdmin(<newOwner>);  // sets fees/dividend exclusions for new owner then transferOwnership(newOwner)
```

---

## 10) Minimal runbook (TL;DR)

1. (Optional) `setmarketingWallet`, `setdonationWallet`
2. (Optional) `setmaxWallet`, `setSwapTriggerAmount`, `updateFees`, `updateTransferFee`
3. (Optional) `updatePayoutToken(<ERC20> or address(0))`
4. **`enableTrading()`**
5. Provide LP, start trading
6. Monitor: `getTotalDividendsDistributed`, `getPayoutToken`, `withdrawableDividendOf(addr)`
7. If needed: `forceSwapAndSendDividends(n)`

---
