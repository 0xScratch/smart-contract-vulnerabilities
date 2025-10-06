# Rental Stop DoS via Disabled `onStop` Hook in reNFT Guard Policy

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-01-renft-findings/issues/501) / [One Bug Per Day](https://www.onebugperday.com/v1/881)
* **Affected Contracts**:
  * [Guard.sol](https://github.com/re-nft/_v3_.smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Guard.sol)
  * [Stop.sol](https://github.com/re-nft/_v3_.smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Stop.sol)
* **Vulnerability Type**: Denial of Service (DoS) / Logic Misconfiguration / Lifecycle Dependency Break

## Summary

The **Guard policy** manages activation status for protocol hooks such as `onStart` and `onStop`, which define custom logic that executes during rental creation and termination.

A **Denial-of-Service condition** arises when an administrator **disables the `onStop` hook** for a specific hook address **after rentals have already been created** using that hook.

Since the rental's hook address is **immutably bound** at creation, any subsequent attempt to stop the rental (`stopRent`) will **revert** because the system checks if the hook is active before execution.
As a result, active rentals associated with that hook become **unstoppable**, freezing both NFT assets and associated ERC20 escrowed payments.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Rental Creation**

   * The lender and renter agree on a hook contract implementing both `onStart` and `onStop` logic.
   * During rental creation, `onStart` is called successfully, and the hook address is stored in the rental order.
   * The rental order now permanently references this hook address.

2. **Rental Stop**

   * When the renter finishes using the NFT, they call `stopRent`.
   * The system looks up the same hook address and executes its `onStop` function.
   * After successful execution, assets are returned to the lender and payments are finalized.

### What Actually Happens (Bug)

1. The administrator later disables the `onStop` hook for that hook address using `Guard::updateHookStatus`.
2. The rental that used this hook is still active and must call the same hook address when stopping.
3. Inside `Stop.sol`, the call flow performs:

   ```solidity
   if (!STORE.hookOnStop(target)) {
       revert Errors.Shared_DisabledHook(target);
   }
   ```

   Since the hook is now disabled, this always **reverts**.
4. As a result, `stopRent()` can **never complete**, permanently locking:

   * The rented NFT inside the renter's safe.
   * The rental payment inside the escrow contract.

### Why This Matters

* All active rentals using a now-disabled hook address **cannot be stopped**.
* Lenders' NFTs and payment tokens are **stuck indefinitely**.
* This effectively **freezes a segment of the protocol** due to an admin configuration change.
* The issue arises not from malicious users, but from **legitimate admin actions**, making it subtle but dangerous.

## Concrete Walkthrough (Alice & Admin Example)

* **Setup:**

  * The Guard contract has `onStart` and `onStop` hooks enabled for a hook contract `HookA`.
  * Alice (renter) and Bob (lender) create a rental using `HookA`.

* **During Rental Creation:**

  * `onStart(HookA)` executes successfully.
  * The rental order now records `hookAddress = HookA`.

* **Admin Action:**

  * The protocol admin later disables the `onStop` hook for `HookA` by calling:

    ```solidity
    guard.updateHookStatus(address(HookA), uint8(3)); // disables onStop
    ```

* **Rental Termination Attempt:**

  * Alice tries to call `stopRent()`.
  * The system checks `STORE.hookOnStop(HookA)` which now returns false.
  * It reverts with `Errors.Shared_DisabledHook(HookA)`.
  * The rental never completes — **assets remain locked forever**.

> **Analogy**: Imagine you rent a car where the company later disables the "return vehicle" function in the system after you've started using it. Now, even though your rental is valid, the system won't accept your return — you're stuck with the car indefinitely.

## Vulnerable Code Reference

### 1) Hook status check in Stop.sol

```solidity
if (!STORE.hookOnStop(target)) {
    revert Errors.Shared_DisabledHook(target);
}
```

### 2) Admin can disable hooks post-rental creation

```solidity
function updateHookStatus(address hook, uint8 status) external onlyAdmin {
    // allows toggling of onStart / onStop hooks for an address
    STORE.setHookStatus(hook, status);
}
```

### 3) Hook address is immutable post-rental creation

```solidity
// during creation
rental.hook = providedHook; // stored permanently for this rental
```

Once recorded, this hook cannot be replaced or updated, so disabling it mid-rental irreversibly breaks `stopRent`.

## Recommended Mitigation

1. **Do not enforce `onStop` status for active rentals**

   * If `onStart` has been executed for a given hook during creation, ensure the corresponding `onStop` hook always executes regardless of its current status flag.

   Example:

   ```solidity
   if (!STORE.hookOnStop(target) && !isActiveRental(target)) {
       revert Errors.Shared_DisabledHook(target);
   }
   ```

2. **Restrict admin disabling logic**

   * Prevent disabling of hooks that are actively referenced by ongoing rentals.
   * Maintain a reference count of active rentals per hook and only allow disabling when count = 0.

3. **Event-based safeguard**

   * Emit events when a hook is disabled while still in use, helping off-chain monitoring systems alert admins before disruption.

4. **Policy reconsideration**

   * Rethink whether `onStop` needs independent toggling. In most cases, if `onStart` ran for a rental, its `onStop` counterpart should always run — the system lifecycle depends on it.

## Pattern Recognition Notes

* **Lifecycle Dependency Breaks** - When two actions (start/stop) are logically paired, disabling one mid-cycle creates unrecoverable states.
* **Admin-Induced DoS** - Even trusted admin actions can cause irreversible disruption if not context-aware.
* **Immutable Hook References** - Systems binding user-specified addresses for future execution must account for state changes in those addresses or their permissions.
* **Fail-Closed vs Fail-Open Trade-Offs** - The check `if (!STORE.hookOnStop(target)) revert;` is a *fail-closed* pattern, safe but potentially harmful when administrative toggling isn't lifecycle-aware.

### Quick Recall (TL;DR)

* **Bug**: Admin disables `onStop` hook after rentals using it are already active.
* **Impact**: Any attempt to call `stopRent()` reverts, locking assets and payments permanently.
* **Fix**: Ensure that once `onStart` runs, the paired `onStop` is always allowed — or prevent disabling hooks in use.
