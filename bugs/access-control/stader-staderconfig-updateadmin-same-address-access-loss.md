# Loss of Admin via Self-Assignment in `updateAdmin` (Role Revocation Bug)

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-06-stader-findings/issues/390) / [One Bug Per Day](https://www.onebugperday.com/v1/269)
* **Affected Contract**: [StaderConfig.sol](https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol)
* **Vulnerability Type**: Access Control / Governance

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol#L176-L183](https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol#L176-L183)
>
>## Vulnerability details
>
>>N.B : This bug is different that the other one titled "The admin address used in initialize function, can behave maliciously". Both issues are related to access control, but the impact, root cause and bug fix are different, so DO NOT mark it as dupliate of the other one.
>
>Current admin will lose DEFAULT_ADMIN_ROLE role if updateAdmin issued with same address.
>
>## Impact
>
>The is a possibility of loss of protocol admin access to this critical StaderConfig.sol contract, if updateAdmin() is set with same current admin address by mistake.
>
>## Proof of Concept
>
>Contract : StaderConfig.sol
>
>Function : function updateAdmin(address _admin)
>
>Using Brownie python automation framework commands in below examples.
>
>1. Step#1 After initialization, admin-A is the admin which has the DEFAULT_ADMIN_ROLE
>
>2. Step#2 update new Admin
>StaderConfig.updateAdmin(admin-B, {'from':admin-A})
>The value of StaderConfig.getAdmin() is admin-B
>
>3. Step#3 admin-B updates admin to itself again
>StaderConfig.updateAdmin(admin-B, {'from':admin-B})
>The value of StaderConfig.getAdmin() is admin-B, but the DEFAULT_ADMIN_ROLE is revoked due to _revokeRole(DEFAULT_ADMIN_ROLE, oldAdmin);
>Now the protocol admin control is lost for StaderConfig contract
>
>## Recommended Mitigation Steps
>
>Ref : [https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol#L177](https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol#L177)
>In the updateAdmin() function, add a check for oldAdmin != _admin , like below
>
>```diff
>    address oldAdmin = accountsMap[ADMIN];
>+   require(oldAdmin != _admin, "Already set to admin");
>```
>
>## Assessed type
>
>Access Control

## Summary

`StaderConfig.updateAdmin(address _admin)` intends to transfer the `DEFAULT_ADMIN_ROLE` from the current admin to a new one. If it's called with the **same** address as the current admin, the function **revokes the role from that address**, leaving the protocol **without any account holding `DEFAULT_ADMIN_ROLE`**. Because `StaderConfig` is the configuration hub for the protocol, losing this role **bricks governance and operations** until an external recovery path (if any) is used.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* The function should *move* admin powers from `oldAdmin` → `newAdmin`:

  1. grant role to `newAdmin`
  2. update stored admin account
  3. revoke role from `oldAdmin`
* This keeps exactly one admin at all times.

### What Actually Happens (Bug)

* The function **doesn't** guard against `newAdmin == oldAdmin`.
* When `newAdmin == oldAdmin`, the last step "revoke oldAdmin" becomes **"revoke newAdmin"** (the same address).
* Net effect: the single admin loses the role; **no account** has `DEFAULT_ADMIN_ROLE`.

### Minimal Walkthrough (Step-by-Step)

Assume `B` is currently the admin and has `DEFAULT_ADMIN_ROLE`.

1. **Call**: `updateAdmin(B)` (i.e., set admin to itself by mistake)
2. Inside the function:

   * `oldAdmin = B` (from internal mapping)
   * Grant role to `B` → a no-op (B already has it)
   * Store `ADMIN = B` → unchanged
   * **Revoke role from `oldAdmin` (which is `B`)** → `B` loses `DEFAULT_ADMIN_ROLE`
3. **Result**: No address holds `DEFAULT_ADMIN_ROLE`.

   * No one can change parameters, rotate addresses, or recover via normal admin operations.
   * Protocol governance is **stuck** (high-impact liveness failure).

> Practical note: A separate initialization quirk (not setting `accountsMap[ADMIN]` during `initialize`) can make self-calls more likely in practice and can also lead to confusing "who actually has the role?" states. But regardless of initialize details, **self-assignment must never revoke the only admin role**.

### Why This Matters

* `StaderConfig` coordinates addresses, limits, and roles across the protocol.
* Losing `DEFAULT_ADMIN_ROLE` means:

  * You can't rotate or upgrade critical contract addresses.
  * You can't adjust operational parameters (deposit/withdraw limits, oracle batch size, etc.).
  * Emergency response and governance actions become impossible.
* Even if this requires a privileged caller, **operator mistakes happen**; when they do, impact is catastrophic.

## Vulnerable Code Reference

```solidity
function updateAdmin(address _admin) external onlyRole(DEFAULT_ADMIN_ROLE) {
    address oldAdmin = accountsMap[ADMIN];

    _grantRole(DEFAULT_ADMIN_ROLE, _admin);
    setAccount(ADMIN, _admin);

    _revokeRole(DEFAULT_ADMIN_ROLE, oldAdmin);
}
```

* Missing guard that `oldAdmin != _admin`.
* Calling with the same address **revokes** the only admin's role.

## Recommended Mitigation

1. **Reject self-assignment** (minimal, targeted fix)

    ```solidity
    function updateAdmin(address _admin) external onlyRole(DEFAULT_ADMIN_ROLE) {
        address oldAdmin = accountsMap[ADMIN];
        require(_admin != address(0), "zero address");
        require(oldAdmin != _admin, "already admin");

        _grantRole(DEFAULT_ADMIN_ROLE, _admin);
        setAccount(ADMIN, _admin);
        _revokeRole(DEFAULT_ADMIN_ROLE, oldAdmin);
    }
    ```

2. **(Additionally) Make initialization consistent**

    * In `initialize`, set the admin mapping to keep storage aligned with roles:

      ```solidity
      _grantRole(DEFAULT_ADMIN_ROLE, _admin);
      setAccount(ADMIN, _admin);
      ```

    * This avoids confusing states and reduces foot-guns around first admin rotation.

3. **(Optional defense-in-depth) Safer transfer pattern**

    * If you ever expect multi-admin transitions, consider a **two-step handover**:
      * Step 1: `proposeAdmin(newAdmin)`
      * Step 2: `acceptAdmin()` (called by `newAdmin`)
    * Or use a temporary "guardian"/timelock/escape-hatch able to restore admin in emergencies.

4. **Tests**

    * Add tests that:
      * `updateAdmin(sameAddress)` **reverts**.
      * Role and mapping move in lock-step for distinct addresses.
      * Post-initialize, `accountsMap[ADMIN]` and `hasRole(DEFAULT_ADMIN_ROLE, admin)` match.

## Pattern Recognition Notes

* **Self-assignment pitfalls**: Any "transfer role/owner" function must guard against `new == old`. Otherwise, revocation steps can nuke the only privileged account.
* **Order & symmetry**: When migrating privileges, maintain a clear sequence and verify final invariants: "exactly one admin" and "mapping matches role membership."
* **Initialization alignment**: Ensure storage mirrors the actual role state from day one (`initialize`). Divergence leads to surprising revocations or lingering privileges.
* **Zero-address & idempotency**: Always forbid `address(0)` and self-reassign; functions should be intentionally non-idempotent (revert on no-op transitions) to prevent accidental bricking.
* **Privileged-only ≠ low severity**: Even if only an admin can trigger it, consider **impact** and **likelihood of operator error**. Here, a single mistaken call can brick governance → High.
