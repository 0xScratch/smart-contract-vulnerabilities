# Unslashable NativeVault via Unvalidated `extraData` and Unrestricted Manager Upgrade

- **Severity**: High
- **Source**: [Code4rena](https://github.com/code-423n4/2024-09-karak-mitigation-findings/issues/8) / [One Bug Per Day](https://www.onebugperday.com/v1/1536)
- **Affected Contract**: [NativeVault.sol](https://github.com/code-423n4/2024-07-karak/blob/f5e52fdcb4c20c4318d532a9f08f7876e9afb321/src/NativeVault.sol) / [SlasherLib.sol](https://github.com/code-423n4/2024-07-karak/blob/f5e52fdcb4c20c4318d532a9f08f7876e9afb321/src/entities/SlasherLib.sol)
- **Vulnerability Type**: Access Control - Unprotected Upgradeability / Role Abuse

## Original Bug Description

>## Lines of code
>
>[https://github.com/karak-network/karak-arena-mitigations/blob/475cfd73744cabe239720feec4a227a739910119/src/NativeVault.sol#L81](https://github.com/karak-network/karak-arena-mitigations/blob/475cfd73744cabe239720feec4a227a739910119/src/NativeVault.sol#L81)
>
>## Vulnerability details
>
>### **H-02:The operator can create a `NativeVault` that can be silently unslashable.**
>
>[Link to issue](https://github.com/code-423n4/2024-07-karak-findings/issues/55)
>
>## Comments
>
>The original implementation allows operators to set `manager`, `slashStore`, and `nodeImplementation` to arbitrary values. This flexibility enables operators to create an unslashable vault by setting the `slashStore` to an address other than the whitelisted slashing handler for ETH
>
>## Mitigation
>
>The mitigation does not fully address all potential risks:
>
>- The `slashStore` issue has been correctly handled by removing the component.
>- The initial `nodeImplementation` appears acceptable, as users and the DSS owner can review it before joining the vault.
>
>However, a vulnerability remains with the `manager` role. The manager, designated by the operator during deployment, retains the ability to call the `changeNodeImplementation` function and change the implementation to anything without delay.
>
>This setup could lead to a scenario where a vault, initially functioning correctly and trusted by users, becomes rogue as the operator or manager updates the implementation to a malicious version without warning.
>
>```solidity
>    /// @notice Allows the owner to change the NativeNode implementation and upgrade all NativeNodes
>    /// @param newNodeImplementation: The address of the new node implementation
>    function changeNodeImplementation(address newNodeImplementation)
>        external
>        onlyOwnerOrRoles(Constants.MANAGER_ROLE)
>        whenFunctionNotPaused(Constants.PAUSE_NATIVEVAULT_NODE_IMPLEMENTATION)
>    {
>        if (newNodeImplementation == address(0)) revert ZeroAddress();
>
>        _state().nodeImpl = newNodeImplementation;
>        emit UpgradedAllNodes(newNodeImplementation);
>    }
>```
>
>## Recommended additional mitigation
>
>A viable solution to further protect users is to modify the `changeNodeImplementation` function to incorporate a request-and-finalize change process, where any proposed changes to the node implementation must go through a time-locked approval phase. This approach allows users who disagree with the proposed changes or detect a potentially malicious implementation to withdraw their funds before the update is finalized.
>
>- The time lock should exceed the withdrawal time lock, providing users sufficient time to exit safely.
>
>This solution allows operators to update the node implementation while protecting users from unexpected and potentially harmful changes, maintaining trust and security within the system.

## Summary

Initially, Karak's **NativeVault** let the **operator** specify three critical addresses—`manager`, `slashStore`, and `nodeImplementation`—via an unvalidated `extraData` field. A mismatch between the protocol's registered slashing handler for ETH and the vault's `slashStore` made that vault impossible to slash, breaking security guarantees. The mitigation removed the customizable `slashStore`, but **did not** restrict the `manager` role: the chosen manager can still instantly swap the vault's node implementation to any malicious contract, risking user funds without warning.

## A Better Explanation (With Simplified Example)

### 1. How Slashing Was Meant to Work

1. **Core** maps each asset (e.g., ETH) to a single, trusted **slashing handler** contract.
2. When misbehavior is detected, **SlasherLib** calls:

    ```solidity
    vault.slashAssets(amount, core.assetSlashingHandlers[asset]);
    ```

3. Inside **NativeVault**, `slashAssets` immediately checks:

    ```solidity
    if (passedHandler != self.slashStore) revert NotSlashStore();
    ```

    Only if both addresses match does it proceed to deduct funds.

### 2. Original Bug: Unslashable Vault via Bad `slashStore`

- At deployment, **operator** runs:

    ```solidity
    deployVaults({
      extraData: abi.encode(manager, badSlashStore, nodeImpl)
    });
    ```

- Because `badSlashStore` was never checked against the protocol's handler, the vault stored `self.slashStore = badSlashStore`.
- Later, when slashing is executed, `core` passes in the *correct* handler, which != `badSlashStore`, causing an immediate revert and **no slashing ever occurring**.

**Result**: A vault immune to slashing—even for valid misbehavior—undermining the protocol's core security.

### 3. Partial Mitigation: Removing `slashStore` Input

The Karak team's fix removed `slashStore` from operator-controlled inputs. All slashes now go to a single, protocol‐managed handler, restoring slashing capability.

### 4. Remaining Risk: Unrestricted `manager` Role Upgrade

Even after removing `slashStore`, the **`manager`** address (still chosen by the operator at deployment) retains the power to call:

```solidity
function changeNodeImplementation(address newImpl)
    external onlyOwnerOrRoles(MANAGER_ROLE) { … }
```

- **Who**: The manager, an address operator designated when deploying the vault.
- **What**: Can instantly switch *all* user node proxies to any `newImpl` contract.
- **Why Dangerous**: A malicious implementation could seize or freeze user ETH, block withdrawals, or disable slashing, with zero notification and no time for users to react.

### 5. Simple End-to-End Illustration

1. **Deployment**: Operator deploys a vault, setting **Mallory** as manager and `GoodNodeImpl` as node logic.
2. **User Stakes**: Alice deposits ETH, relies on `GoodNodeImpl` for proof-based reward accounting and safe withdrawals.
3. **Manager Attack**: Mallory immediately calls `changeNodeImplementation(MaliciousNodeImpl)`.
4. **Compromise**:
    - Any further `startWithdrawal` or snapshot operations now invoke `MaliciousNodeImpl`.
    - That contract can divert funds to Mallory or block genuine withdrawals.

Alice has **no warning** or time window to withdraw beforehand.

## Impact

- **Security Guarantee Broken**: The core mechanism for penalizing misbehaving operators—slashing—can be completely bypassed if an operator deploys a NativeVault with an incorrect slashing handler, or later upgrades node implementations maliciously. This allows malicious operators to avoid economic penalties.
- **Protocol Integrity Risk**: Unsplashable vaults undermine trust in the Karak protocol's economic security model, as operators can misbehave with impunity.
- **User Funds at Risk**: Users staking in these vaults may lose funds due to malicious upgrades or disabled slashing mechanisms, with no ability to recover via slashing.
- **Loss of Decentralization and Accountability**: If operators or managers can instantly change critical contract logic without delay or oversight, the system becomes vulnerable to centralized attacks and rug pulls.

## Vulnerable Code Reference

```solidity
// SlasherLib.sol snippet invoking slashAssets on vaults
for (uint256 i = 0; i < queuedSlashing.vaults.length; i++) {
    IKarakBaseVault(queuedSlashing.vaults[i]).slashAssets(
        queuedSlashing.earmarkedStakes[i],
        self.assetSlashingHandlers[IKarakBaseVault(queuedSlashing.vaults[i]).asset()]
    );
}

// NativeVault.sol snippet checking slashing handler
function slashAssets(uint256 totalAssetsToSlash, address slashingHandler)
    external
    onlyOwner
    nonReentrant
    returns (uint256 transferAmount)
{
    if (slashingHandler != self.slashStore) revert NotSlashStore();

    // Slashing logic removing funds from the vault
    if (totalAssetsToSlash > self.totalAssets) {
        totalAssetsToSlash = self.totalAssets;
    }
    self.totalAssets -= totalAssetsToSlash;

    emit Slashed(totalAssetsToSlash);
    return totalAssetsToSlash;
}

// NativeVault.sol snippet for manager-controlled upgrade
function changeNodeImplementation(address newNodeImplementation)
    external
    onlyOwnerOrRoles(Constants.MANAGER_ROLE)
{
    if (newNodeImplementation == address(0)) revert ZeroAddress();
    _state().nodeImpl = newNodeImplementation;
    emit UpgradedAllNodes(newNodeImplementation);
}
```

## Recommended Mitigation

Combine the original fix with a **time-locked manager upgrade**:

1. **Remove Operator Input for Critical Roles**: Already done for `slashStore`.
2. **Time-Lock Manager Actions**:
    - Replace `changeNodeImplementation` with a two-step process:
        - `proposeChangeNodeImplementation(newImpl)` records the change and timestamp.
        - Enforce a delay longer than the vault's withdrawal delay (e.g., 3×), giving users time to exit.
        - `finalizeChangeNodeImplementation()` executes the upgrade after the waiting period.
3. **Optional Governance Review**: Emit an event on proposal so off-chain watchers or a governance contract can veto.

This combined approach ensures:

- **Slashing** cannot be bypassed at deployment.
- **Upgrades** occur only after public notice and sufficient time for user reaction, preserving trust and security.

## Pattern Recognition Notes

- **Watch for Initialization Unchecked Inputs for Privileged Roles**
  - Look for contracts that accept user- or operator-controlled inputs during `initialize`, `constructor`, or factory deployment which assign critical roles (e.g., `manager`, `admin`, `owner`) without validation or whitelisting.
  - Example heuristic: usage of `abi.decode(extraData)` or large dynamic inputs feeding directly into role assignment or key config variables.
- **Identify Privileged Functions Guarded Only by Roles Given at Deployment**
  - If upgrade or sensitive functions use modifiers like `onlyOwnerOrRoles(someRole)`, verify who and how these roles are assigned.
  - If roles can be assigned arbitrarily or persist without on-chain checks, that creates an attack surface for malicious actors to gain control.
- **Check For Immediate and Unrestricted Upgradability**
  - Look for upgrade functions (`changeImplementation()`, `setLogic()`, `upgradeTo()`, etc.) callable immediately by privileged roles without any **timelock**, **multi-sig**, or **governance delay**.
  - Instant upgrades allow stealthy or rug-pull style attacks with no notice.
- **Cross-Contract Invariants on Shared Parameters**
  - In multi-contract systems, if one contract (e.g., core) expects certain parameters/handlers to be fixed or registered, ensure other contracts (e.g., vaults, proxies) use the same trusted source and validate externally supplied inputs.
  - Mismatches between who sets and who verifies parameters often cause logic bypass or trust failures.
- **Unstructured or Dynamic Storage Initializations Are High Risk**
  - Contracts that store config/state through unstructured storage slots, or dynamically decode initialization data, are more error-prone.
  - These patterns often lack rigorous validation and can lead to corrupted state or privilege escalation.
- **Delegation of Responsibility Without Enforcement**
  - If critical security properties depend on an external party (e.g., another contract, service, or module) to enforce constraints (like slashing capability or operator revocation), auditors should check how those guarantees are enforced or validated.
  - Over-delegation without protocol-level checks risks allowing bad actors to exploit assumptions.
- **Heuristic Checks to Use**
  - Search for all instances of role assignments derived from input data (e.g., factory parameters).
  - Enumerate all "upgrade" or "change implementation" functions and verify presence of explicit delay mechanisms or multi-sig patterns.
  - Flag cases where sensitive addresses or modules' addresses can be set/changed by unverified parties.
- **Look for Event Emissions and Transparency Mechanisms**
  - Good upgrade patterns emit events on proposals and finalizations, enabling off-chain monitoring.
  - Absence of these or silent changes suggest risk.
- **Common Vulnerable Patterns**
  - Factory patterns creating user-deployable contracts passing privileged roles blindly.
  - Role assignments not restricted to a well-known registry or governance set.
  - Upgrade functions triggered by single privileged entity without timelocks.
  - Cross-contract inconsistencies between expected and actually stored privileged addresses or handlers.
