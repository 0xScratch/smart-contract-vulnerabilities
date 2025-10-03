# Initial Mint Front‑Run Inflation Attack — SelfPeggingAsset (Tapio / NUTS Finance)

* **Severity**: Critical
* **Source**: [NUTS Tapio Security Audit Report](https://github.com/mixbytes/audits_public/blob/master/NUTS%20Finance/Tapio/README.md#1-initial-mint-front-run-inflation-attack)
* **Affected Contract**: [SelfPeggingAsset.sol](https://vscode.blockscan.com/146/0xAACF301E4397171bf838592b9f4b58Ae44cCf285)
* **Vulnerability Type**: Economic attack / Front‑run / Rounding/Integer math / Initialization logic

## Summary

When the pool is empty, an attacker can *front‑run the initial mint* with a tiny transfer (e.g. 1 wei) and mint a tiny number of LP shares before a legitimate first depositor. By then pushing the legitimate depositor's large token transfer into the pool **before** the legitimate `mint()` executes, the attacker inflates the token balance relative to the tiny share supply. Because the `mintShares` calculation uses integer division (floors), the legitimate depositor can receive **0 shares** for their large deposit, effectively donating their funds and giving the attacker near‑total ownership.

This is a classic "first‑mounter" exchange‑rate poisoning attack caused by small initial share supply + pre‑transfer sequencing + integer rounding.

## A Better Explanation (With Simplified Example)

### Intended behavior (what should happen on first deposit)

1. When the pool is empty: the initial depositor seeds the pool and receives LP shares that fairly reflect the token amounts deposited.
2. The initial exchange rate (tokens per share) should be deterministic and not manipulable by other actors in the same block/transaction ordering.
3. `mint()` should not be susceptible to tiny pre‑transfers that change pool balances without corresponding share adjustments.

### What actually happens (bug)

1. Transfers of tokens directly to the pool address are allowed and can happen *before* `mint()` is called.
2. A malicious actor transfers a tiny balance (e.g. 1 wei) and then calls `mint()` to create a tiny `totalShares > 0` while the token balances are tiny.
3. The attacker then transfers the large amounts that the honest depositor intended to deposit (or otherwise arranges balances) — so token balances balloon while `totalShares` remains tiny.
4. The honest depositor's subsequent `mint()` call computes minted shares using a formula that floors the result: `shares = floor(depositTokens * totalShares / totalTokenBalance)`. With tiny `totalShares` and large `totalTokenBalance` the result becomes `0`.
5. The honest depositor receives 0 shares while the attacker retains the tiny share supply that now represents almost all deposited tokens.

## Concrete Walkthrough (Mallory & Alice) — numeric example

**Actors & values**:

* Mallory (attacker) — wants to manipulate first mint
* Alice (legit first provider) — intends to deposit `1000` tokens
* Mallory's tiny pre‑transfer: `1 wei`

**Start**:

* `totalShares = 0`
* `poolTokenBalance = 0`

**Step 1 — Mallory pre‑transfers 1 wei to pool address**:

* `poolTokenBalance = 1` (balance increased by 1, but no shares exist yet)

**Step 2 — Mallory calls `mint()` and mints shares against the 1 wei**:

* Contract mints `1` share (or some tiny non‑zero shares) to Mallory.
* `totalShares = 1`
* `poolTokenBalance = 1` (unchanged)

**Step 3 — Mallory (still) transfers Alice’s intended 1000 tokens into the pool address**:

* `poolTokenBalance = 1 + 1000 = 1001`
* `totalShares = 1` (still tiny; Mallory owns it)

**Step 4 — Alice now executes `mint()` to deposit `1000` tokens**:

* Inside `mint()`, pool calls `poolToken.mintShares` which computes:

```solidity
sharesMinted = floor( depositTokens * totalShares / totalTokenBalance )
            = floor( 1000 * 1 / 1001 )
            = floor(0.999...) = 0
```

* Alice receives `0` shares even though she deposited 1000 tokens.
* Mallory owns `1 / 1 = 100%` of LP shares and therefore controls the pool.

**Result**: Mallory effectively steals Alice's deposit — the initial legitimate liquidity is captured by the attacker.

## Why this happens (root causes)

1. **Pre‑transfer sequencing**: tokens can be moved to the pool address independently from `mint()`, allowing state to be manipulated between balance reads and share minting.
2. **Tiny initial shares**: the contract allowed a tiny non‑zero `totalShares` to exist during pool initialization, creating a manipulable exchange rate when token balances increase afterwards.
3. **Integer division flooring**: `shares = floor(amount * totalShares / totalTokenBalance)` can evaluate to zero when `totalShares` is tiny relative to `totalTokenBalance`.
4. **Lack of special case on first mint**: there was no safe, deterministic logic to set the initial shares↔tokens exchange rate when `totalShares == 0`.

## Vulnerable Code Reference (conceptual)

> The vulnerability lies in the `mint()` flow where the contract converts token deposit amounts into LP shares by calling `poolToken.mintShares(...)` (or equivalent), using the *current* `totalShares` and *current* pool token balances to compute shares.

### Pseudocode of the vulnerable flow

```solidity
// simplified view of the vulnerable path
function mint(uint256[] memory tokenAmounts, uint256 minMintAmount) external {
    // tokenAmounts are pull‑transferred earlier (or assumed present)

    // calculate amount of shares to mint for depositor using current totalShares & balances
    uint256 shares = poolToken.mintShares(tokenAmounts); // internally: floor(amount * totalShares / totalTokenBalance)

    require(shares >= minMintAmount, "Slippage: minted shares too low");

    // mint LP shares to user
    poolToken._mint(msg.sender, shares);
}
```

### Dangerous sequence that makes the math fail

1. Attacker `transfer(token, pool, 1)` (balance becomes 1)
2. Attacker `mint()` → `totalShares` becomes 1
3. Attacker `transfer(token, pool, 1000)` (balance becomes 1001)
4. Honest user `mint(1000)` → shares computed as `floor(1000 * 1 / 1001) = 0`

## Recommended Mitigations (practical & testable)

Multiple safe approaches — choose one or combine:

### 1) **Canonical fix (chosen by project)** — If `totalShares == 0` set initial shares deterministically

When first mint occurs (total LP supply = 0), mint shares equal to the deposit token amount (establish a 1:1 starting exchange rate) or otherwise ensure the minted shares are non‑zero and deterministic:

```solidity
uint256 totalShares = poolToken.totalSupply();
if (totalShares == 0) {
    // Approach A: set initial shares = the deposit token amount (1:1 baseline)
    uint256 sharesToMint = totalTokenDeposit; // or a function of deposits
    poolToken._mint(msg.sender, sharesToMint);
    // set invariants / emit event
    return;
}

// otherwise use the normal proportional mint calculation
uint256 shares = poolToken.mintShares(tokenAmounts);
poolToken._mint(msg.sender, shares);
```

**Pros**: Simple, deterministic, prevents exchange‑rate poisoning. Easy to reason about initial price.

**Cons**: Requires selecting a sensible baseline if deposits contain multiple tokens with different units (use normalized amounts / rate provider to convert) — the implementation must use the same normalization that `mintShares` expects.

### 2) **Mint a reserved "dead" share supply at init**

Reserve a small, non‑trivial number of shares that are minted to a burn address or admin on the first successful initialization, guaranteeing `totalShares` is large enough to avoid 0‑flooring during a single‑block reorder.

```solidity
if (poolToken.totalSupply() == 0) {
    // e.g. mint a small dead supply to address(0) or to the protocol
    poolToken._mint(address(0xdead), 1000);
}
```

**Pros**: Keeps logic uniform and preserves proportional mint math.
**Cons**: Protocol must accept the small vesting/burned shares and choose an amount that is large enough but not harmful.

### 3) **Require mint to be atomic and use transferFrom**

Don't rely on token balances present in the pool address. Require users to `approve` and the contract to `transferFrom` inside the same `mint()` call. This prevents third parties from injecting token transfers between balance reads and minting.

```solidity
// inside mint()
for each token: token.transferFrom(msg.sender, address(this), amount);
// now compute shares based on balances (safe: no pre‑transfers from others in between)
```

**Pros**: Prevents external pre‑transfers from reordering.
**Cons**: Some UX friction, and incompatibilities if the system previously supported direct transfers.

### 4) **Pause initial mint to admin only**

Start pools paused; the admin performs the first seed deposit (trusted). After initialization, unpause and allow public mints.

**Pros**: Very simple and operationally safe.
**Cons**: Centralized initialization, requires trusted party and off‑chain coordination.

### 5) **Add extra checks around minMintAmount and rounding**

Don't rely on caller‑provided `minMintAmount` (which can be bypassed due to how token amounts are mapped to shares). Add protections that detect suspicious minted shares (e.g., if shares == 0 and depositTokens > 0 revert with explicit message).

```solidity
uint256 shares = poolToken.mintShares(tokenAmounts);
if (shares == 0 && totalTokenDeposit > 0) revert("Initial mint rounding protection");
```

**Test both happy path and attack path** to ensure legitimate small deposits don't break while preventing the exploit.

## Suggested Tests to Add

1. **First mint safe baseline** — simulate attacker pre‑transfer of 1 wei then first legit deposit; assert legit deposit mints non‑zero shares and attacker cannot steal the pool.
2. **Atomic transferFrom behavior** — ensure `mint()` with `transferFrom` prevents arbitrary pre‑transfers.
3. **Mint rounding property** — property test over many (random) `totalShares` and `balances` to assert the minted shares are never zero for significant deposits.
4. **Integration & unit tests** — test multi‑token deposits with normalization via rate provider to verify first‑mint logic works for multi‑asset pools.

## Pattern Recognition Notes (how to recognise similar bugs)

* **First‑mounter assaults**: Attacks targeting initialization logic are common wherever total supply or denominators start at zero.
* **Relying on `balanceOf(address)` snapshots**: If code reads balances and then relies on them across multiple externally visible steps, reordering attacks (including direct transfers) are possible.
* **Integer flooring causing zero outcomes**: When `floor(a * b / c)` is used and `b` is tiny, watch for `0` results when `a` is smaller than `c`.
* **Sentinel conflation**: When `0` is used both as a valid numeric state and a sentinel (e.g., "no shares"), injection of tiny values can collapse invariant checks.
* **Missing atomicity on deposit flows**: Best practice is to pull funds inside the function using `transferFrom` so ordered operations are atomic.

## Quick Recall (TL;DR)

* **Bug**: Attacker pre‑transfers tiny tokens and mints tiny shares first, then pushes the real deposit amounts into the pool; rounding math causes the honest depositor to mint `0` shares.
* **Impact**: Attacker captures essentially all LP shares and steals the initial liquidity.
* **Fixes**: Make the first mint deterministic (set initial shares = tokenAmount or mint a dead supply), require atomic `transferFrom`, or restrict initial mint to admin.
