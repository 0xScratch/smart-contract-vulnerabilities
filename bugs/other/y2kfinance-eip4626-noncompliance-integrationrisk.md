# EIP-4626 Interface Mismatch Causing Potential Integration Breakage in SemiFungibleVault

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/47)
* **Affected Contract**: [Vault.sol](https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/Vault.sol), [SemiFungibleVault.sol](https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/SemiFungibleVault.sol)
* **Vulnerability Type**: Standards Non-Compliance / Composability Risk / Integration Inconsistency

## Summary

`SemiFungibleVault` is presented in the README as an **ERC-4626 vault adaptation** (replacing ERC-20 shares with ERC-1155 tokens).
However, the contract **deviates from the ERC-4626 specification** in multiple places:

* Missing required interface functions (`mint()` and `redeem()`).
* `previewDeposit()` ignores deposit fees and therefore misreports share conversion.
* `totalAssets()` does not always return the true amount of managed assets (e.g., after de-peg events or epoch payouts).
* `max*()` functions always return unlimited values, failing to reflect deposit/withdraw locking states.

Because the project explicitly advertises ERC-4626 compliance, other DeFi integrators may assume standard behavior and break or lose funds when those assumptions fail.

This makes the issue **a high-risk integration vulnerability** rather than an isolated logic bug.

## A Better Explanation (With Simplified Example)

### Intended Behavior (under full ERC-4626 compliance)

An ERC-4626 vault must:

1. Offer both **asset-first** (`deposit`, `withdraw`) and **share-first** (`mint`, `redeem`) entry points.
2. Provide *accurate previews* that match on-chain results (`previewDeposit`, `previewWithdraw`).
3. Return the **true amount of assets under management** in `totalAssets()`.
4. Use `maxDeposit`, `maxWithdraw`, etc. to indicate when vault operations are **temporarily disabled or capped**.

Integrators (like yield routers or aggregators) can then safely:

* Query these functions to compute expected results.
* Route deposits or withdrawals automatically without reverts or mispricing.
* Treat the vault as a "plug-and-play" yield source.

### What Actually Happens (Bug)

`SemiFungibleVault` implements only a subset of ERC-4626 behavior:

| Function                        | Issue                                                                             |
| ------------------------------- | --------------------------------------------------------------------------------- |
| `mint()` / `redeem()`           | Completely missing. External protocols expecting them will revert.                |
| `previewDeposit()`              | Ignores protocol-level deposit fees → overestimates shares minted.                |
| `totalAssets()`                 | Excludes distributed or pending assets during epoch transitions → inaccurate AUM. |
| `maxDeposit()`, `maxWithdraw()` | Always return `type(uint256).max` even if vault should be closed/locked.          |

When an integrator or router uses automated ERC-4626 logic (as they would for any "ERC-4626 vault"), it may compute wrong deposit amounts, fail calls, or mis-account total TVL.

### Why This Matters

EIP-4626 was created to standardize **tokenized vault interfaces** and guarantee safe composability.
Deviating from it while still claiming compliance creates **silent interoperability failures**.

Consequences include:

* **Integrator breakage:** DApps like aggregators, yield optimizers, or accounting systems revert or misreport yields.
* **User confusion:** UI previews show wrong share amounts; users receive fewer shares than expected.
* **Economic desync:** Strategies relying on `totalAssets()` for portfolio balancing misallocate funds.
* **Potential fund loss:** Cross-protocol integrations performing atomic routing (deposit + swap + redeem) may revert mid-transaction, locking or mispricing assets.

### Concrete Walkthrough (Example Scenario)

1. **Assumption:** An aggregator identifies Y2K's `SemiFungibleVault` as an ERC-4626 vault (it matches most of the interface and is described as such).
2. **Router Action:** The aggregator calls `mint(shares, receiver)` to deposit assets.
3. **Reality:** `mint()` is not implemented → transaction reverts.

Alternatively:

1. A front-end calls `previewDeposit(1000e6)` expecting to get 1000 shares (1:1).
2. The contract charges a 5% deposit fee internally, but `previewDeposit` ignores this.
3. User actually receives only 950 shares but the UI shows 1000 → user confusion / integration mis-accounting.

Or:

1. `totalAssets()` fails to include tokens already distributed after a de-peg event.
2. External TVL dashboards report inflated asset values → incorrect protocol analytics or liquidity misrouting.

### Why It's Classified as High Risk

The vulnerability's severity comes **not from direct fund theft**, but from **breaking the standard's guarantees**.
Since Y2K explicitly states ERC-4626 compliance, other protocols will trust it as such.
That mismatch can **cascade across integrations**, leading to:

* Failed deposits in routers (DoS for integrators).
* Incorrect price or TVL readings.
* Mis-accounted yield distributions.

This exact reasoning was also upheld in earlier audits (e.g., Notional Coop contest), where deviations from ERC-4626 were judged **High Risk** due to composability and fund-safety implications.

## Vulnerable Code Reference

### 1) Missing Required ERC-4626 Entry Points

```solidity
// No functions:
// mint(uint256, address) returns (uint256)
// redeem(uint256, address, address) returns (uint256)
```

### 2) Non-Compliant Previews

```solidity
function previewDeposit(uint256 id, uint256 assets)
    public
    view
    virtual
    returns (uint256)
{
    return convertToShares(id, assets); // ignores deposit fees
}
```

### 3) Incomplete Accounting

```solidity
function totalAssets(uint256 _id) public view virtual returns (uint256);
// Does not include distributed assets after epoch settlement.
```

### 4) Unlimited Gating Functions

```solidity
function maxDeposit(address) public view virtual returns (uint256) {
    return type(uint256).max; // should be 0 if deposits disabled
}
```

## Recommended Mitigation

1. **Implement missing EIP-4626 functions**

    ```solidity
    function mint(uint256 id, uint256 shares, address receiver)
        external
        virtual
        returns (uint256 assets)
    {
        assets = previewMint(id, shares);
        asset.safeTransferFrom(msg.sender, address(this), assets);
        _mint(receiver, id, shares, EMPTY);
        emit Deposit(msg.sender, receiver, id, assets, shares);
        afterDeposit(id, assets, shares);
    }

    function redeem(uint256 id, uint256 shares, address receiver, address owner)
        external
        virtual
        returns (uint256 assets)
    {
        require(msg.sender == owner || isApprovedForAll(owner, msg.sender));
        assets = previewRedeem(id, shares);
        beforeWithdraw(id, assets, shares);
        _burn(owner, id, shares);
        emit Withdraw(msg.sender, receiver, owner, id, assets, shares);
        asset.safeTransfer(receiver, assets);
    }
    ```

2. **Update `preview*` functions to include fee logic**

    ```solidity
    uint256 fee = assets.mulDivDown(depositFeeBps, 10_000);
    uint256 netAssets = assets - fee;
    return convertToShares(id, netAssets);
    ```

3. **Ensure `totalAssets()` returns the actual managed balance**
   Include or subtract tokens distributed during epoch transitions or held elsewhere.

4. **Override gating functions in child vaults**
   If an epoch or vault is closed, `maxDeposit` / `maxWithdraw` should return `0`.

5. **Document partial compliance**
   If full ERC-4626 semantics aren't desired, explicitly state *"ERC-4626-like, non-standard"* to avoid misleading integrators.

## Pattern Recognition Notes

* **Standards Claim vs. Implementation Drift** — claiming adherence to a standard while omitting key methods breaks downstream expectations and composability.
* **Preview Accuracy Risk** — inaccurate previews lead to UI, accounting, or automated routing errors.
* **Interface Incompleteness** — missing required functions (e.g., `mint`, `redeem`) create runtime reverts in integrations that assume their presence.
* **False Signaling** — declaring ERC-4626 compliance without full conformity is a **social-layer vulnerability**; it propagates assumptions through the ecosystem.
* **Composability Dependence** — DeFi composability means a small spec deviation can trigger large ecosystem failures, even without direct exploitability.

### Quick Recall (TL;DR)

* **Bug:** Contract claims ERC-4626 compliance but misses critical functions and deviates from spec.
* **Impact:** Integrators relying on ERC-4626 assumptions may break, miscalculate, or misprice assets → potential fund loss or DoS.
* **Fix:** Implement missing API methods, update previews and `totalAssets()` for accuracy, and ensure `max*()` reflect vault state — or clearly stop claiming ERC-4626 compliance.
