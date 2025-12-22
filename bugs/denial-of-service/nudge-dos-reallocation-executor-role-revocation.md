# DoS on Reallocation Processing via Privileged Executor Role Revocation

* **Severity**: Medium
* **Source**: [Code4rena](https://code4rena.com/audits/2025-03-nudgexyz/submissions/F-14)
* **Affected Contract**:
  * [NudgeCampaign.sol](https://github.com/code-423n4/2025-03-nudgexyz/blob/main/src/campaign/NudgeCampaign.sol#L164-L233)
  * [NudgePointsCampaigns.sol](https://github.com/code-423n4/2025-03-nudgexyz/blob/main/src/campaign/NudgePointsCampaigns.sol#L126-L178)
  * [NudgeCampaignFactory.sol](https://github.com/code-423n4/2025-03-nudgexyz/blob/main/src/campaign/NudgeCampaignFactory.sol#L4)
  * [Executor.sol](https://github.com/lifinance/contracts/blob/b8c966aad30407b3f579723847057729549fd353/src/Periphery/Executor.sol#L105-L126)
* **Vulnerability Type**: Denial of Service (DoS) / Access Control / Third-Party Integration Risk

## Summary

The Nudge protocol integrates Li.Fi's **Executor** contract to facilitate asset swaps during user reallocations. The Executor is granted the `SWAP_CALLER_ROLE` in the NudgeCampaignFactory, allowing it to call the restricted `handleReallocation` function in campaign contracts to record user movements and trigger rewards.

However, the Executor's `swapAndExecute` function is **public and unrestricted**—anyone can call it and instruct the Executor to perform arbitrary external calls (including to the NudgeCampaignFactory).

An attacker can exploit this by directing the Executor to call `renounceRole(SWAP_CALLER_ROLE, executorAddress)` on the Factory. Since the call originates from the Executor itself, it succeeds in revoking its own role.

Once revoked, all future legitimate calls from the Executor to `handleReallocation` revert due to missing permissions. This halts reallocation processing across **all campaigns**, preventing users from claiming rewards. The attack is repeatable: even if admins re-grant the role, the attacker can revoke it again instantly and cheaply.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Reallocation Flow**: A user swaps assets (possibly via Li.Fi) to meet campaign criteria (e.g., acquire and hold target token).
2. **Executor Action**: After the swap, the Executor (as a trusted caller) invokes `handleReallocation` on the relevant NudgeCampaign contract.
3. **Permission Check**: `handleReallocation` is guarded by `hasRole(SWAP_CALLER_ROLE, msg.sender)`. Since the Executor holds this role, the call succeeds → user's reallocation is recorded → rewards become claimable.

The `SWAP_CALLER_ROLE` is managed via OpenZeppelin's AccessControl in the Factory contract, with standard functions like `grantRole` and `renounceRole`.

### What Actually Happens (Bug)

* Li.Fi's Executor is designed to be **permissionless**—its `swapAndExecute` allows anyone to specify arbitrary post-swap calls.
* When Nudge grants the Executor elevated privileges (`SWAP_CALLER_ROLE`), it inadvertently turns this public interface into a **privileged self-destruct mechanism**.
* An attacker crafts a dummy swap (e.g., using a self-deployed token) and embeds a call to `renounceRole` targeting the Executor's own role.
* The Executor executes the renounce → loses `SWAP_CALLER_ROLE` → future `handleReallocation` calls fail → protocol-wide reallocation freeze.

### Why This Matters

* Core protocol functionality (reward distribution for reallocations) becomes unavailable indefinitely.
* No funds are stolen, but user trust erodes quickly—campaigns stall, protocols can't attract liquidity/activity via Nudge.
* Attack is low-cost, griefing-resistant, and repeatable, forcing constant admin intervention.
* Highlights risks of integrating permissionless third-party contracts into privileged flows without isolating or restricting their capabilities.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: Executor has `SWAP_CALLER_ROLE`; campaigns are active and funded.
* **Mallory attack**: Mallory deploys a dummy ERC20, approves/mints minimal amounts, then calls `executor.swapAndExecute` with:
  * A fake swap (self-to-self, amount=1).
  * Post-swap call: `factory.renounceRole(SWAP_CALLER_ROLE, address(executor))`.
* **Result**: Call succeeds → Executor revokes its own role.
* **Alice reallocates**: Alice performs a legitimate asset move. Executor attempts `handleReallocation` → reverts (no role) → Alice's reallocation isn't recorded → no rewards.
* **Global Effect**: Every subsequent reallocation fails the same way. Admins can re-grant the role, but Mallory repeats the attack immediately → permanent DoS unless code is upgraded.

> **Analogy**: Imagine a secure factory where only a trusted robot (Executor) can press the "approve production" button. The robot has a public remote control that anyone can use to make it do extra tasks. An attacker uses the remote to make the robot punch itself in the off-switch—disabling itself forever. Even if you turn it back on, the attacker can just do it again.

## Vulnerable Code Reference

**1) Public arbitrary execution in Li.Fi Executor**:

```solidity
// Executor.sol (Li.Fi)
function swapAndExecute(
    bytes32 _transactionId,
    LibSwap.SwapData[] calldata _swapData,
    address _token,
    address payable _receiver,
    uint256 _amount
) external {
    // ... performs swaps ...
    // Then executes arbitrary calls
    _executeCall(_swapData[i].callTo, _swapData[i].callData);
}
```

**2) Role-based guard on reallocation (relies on Executor having role)**:

```solidity
// NudgeCampaign.sol / NudgePointsCampaign.sol
modifier onlySwapCaller() {
    require(hasRole(SWAP_CALLER_ROLE, msg.sender), "Not authorized");
    _;
}

function handleReallocation(...) external onlySwapCaller {
    // Records reallocation and updates rewards
}
```

**3) Standard renounceRole allows self-revocation**:

```solidity
// NudgeCampaignFactory.sol (inherits AccessControl)
function renounceRole(bytes32 role, address account) public virtual override {
    require(msg.sender == account, "AccessControl: can only renounce roles for self");
    _revokeRole(role, account);
}
```

## Recommended Mitigation

1. **Prevent Executor from renouncing its role** (quick fix):

    ```solidity
    // In NudgeCampaignFactory.sol
    address public immutable EXECUTOR;

    function renounceRole(bytes32 role, address account) public virtual override {
        require(!(role == SWAP_CALLER_ROLE && account == EXECUTOR), "Executor cannot renounce SWAP_CALLER_ROLE");
        super.renounceRole(role, account);
    }
    ```

2. **Replace role check with direct address verification** (preferred for immutability):

    Eliminates role entirely for this permission → no way to "renounce".

    ```solidity
    // In NudgeCampaign.sol / NudgePointsCampaign.sol
    address public immutable SWAP_CALLER;  // Set to Executor in factory/deploy
    
    modifier onlySwapCaller() {
        require(msg.sender == SWAP_CALLER, "Not authorized");
        _;
    }
    ```

3. **Additional defenses**: If keeping roles, make Executor non-renounceable or use a proxy/wrapper that filters calls.
4. **Testing**: Add PoC tests simulating the attack; ensure integration tests cover third-party permissionless behaviors.
5. **General**: When integrating permissionless external contracts into privileged paths, audit their full capabilities and add wrappers/restrictions if needed.

## Pattern Recognition Notes

* **Privileged Third-Party Integration Risk**: Granting roles/permissions to permissionless contracts can expose internal functions to anyone via the external contract's public interface.
* **Self-Revocation Vectors**: Standard AccessControl allows easy self-renunciation—dangerous for critical roles held by contracts with public entrypoints.
* **Arbitrary Call Exposure**: Functions like `swapAndExecute` that allow user-specified calls are powerful but toxic when the caller holds privileges elsewhere.
* **DoS via Griefing**: Low-cost, repeatable attacks that don't steal funds but disrupt service are increasingly common—design for recoverability.
* **Immutable Critical Permissions**: For fixed trusted callers (like external integrators), prefer hardcoded address checks over revocable roles to avoid revocation vectors.

## Quick Recall (TL;DR)

* **Bug**: Public Executor can be tricked into renouncing its own `SWAP_CALLER_ROLE`.
* **Impact**: Loses permission → all `handleReallocation` calls fail → protocol-wide freeze on reward processing.
* **Fix**: Block renunciation for Executor or switch to immutable address check (`msg.sender == executor`); test third-party integrations carefully.
