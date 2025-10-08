# Permanent Lock via Expired-Lock Undelegation Restriction in VotingEscrow

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2023-08-verwa-findings/issues/268) / [One Bug Per Day](https://www.onebugperday.com/v1/514)
* **Affected Contract**: [VotingEscrow.sol](https://github.com/code-423n4/2023-08-verwa/blob/a693b4db05b9e202816346a6f9cada94f28a2698/src/VotingEscrow.sol)
* **Vulnerability Type**: Logic Flaw / Denial of Service (DoS) / Timing Constraint

## Summary

The **`VotingEscrow`** contract implements delegation for vote-escrowed locks (`veRWA`).
A user may **delegate** their locked voting power to another address and later **undelegate** (delegate back to themselves).
However, undelegation requires that the user's **lock has not yet expired**.

If a user's lock expires before they undelegate, the protocol **blocks both undelegation and withdrawal**.
Because withdrawal is only permitted when the lock is not delegated, the user's tokens become **permanently locked**, creating a **non-recoverable denial of service** for the affected account.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Lock**: User deposits tokens via `createLock()`.

   * Lock is valid for 5 years.
   * User receives voting power proportional to lock time and amount.

2. **Delegate**: User delegates voting power to another user.

   ```solidity
   ve.delegate(bob);
   ```

   * Voting rights transfer to Bob.
   * Alice (the delegator) cannot withdraw until she undelegates.

3. **Undelegate**: When ready to withdraw, user re-delegates to themselves.

   ```solidity
   ve.delegate(alice);
   ```

   * This restores control of voting power to Alice.
   * She can now call `withdraw()` after lock expiry.

4. **Withdraw**: After the lock expires, the user retrieves their tokens.

### What Actually Happens (Bug)

When a user attempts to **undelegate after lock expiry**, this line reverts:

```solidity
require(toLocked.end > block.timestamp, "Delegatee lock expired");
```

Let's walk through an example with **Alice and Bob**:

#### 1. Lock and Delegate

```solidity
ve.createLock(100 ether);
ve.delegate(bob);
```

* Alice's `delegatee = Bob`.
* Her lock ends in 5 years.

#### 2. Time Passes

After 5 years:

```solidity
block.timestamp > locked.end
```

→ Alice's lock is expired.

#### 3. Alice Tries to Undelegate

```solidity
ve.delegate(alice);
```

Inside the function:

```solidity
else if (_addr == msg.sender) {
    // Undelegate
    fromLocked = locked[delegatee]; // Bob's lock
    toLocked = locked_;             // Alice's lock
}
require(toLocked.end > block.timestamp, "Delegatee lock expired");
```

* Here, `toLocked` = Alice's expired lock.
* The `require()` fails → **"Delegatee lock expired"**.
* Alice cannot undelegate.

#### 4. Alice Tries to Withdraw

```solidity
ve.withdraw();
```

But withdrawal requires:

```solidity
require(locked_.delegatee == msg.sender, "Lock delegated");
```

* Still delegated to Bob → reverts again.
  💀 **Alice's tokens are stuck forever.**

### Why This Matters

This is a **permanent account-level denial of service**:

* User cannot undelegate (due to expired lock).
* User cannot withdraw (because still delegated).
* Tokens remain **irrecoverable**.

No admin or protocol function exists to recover the lock.
Even a minor oversight (forgetting to undelegate before expiry) results in total loss.

### Concrete Walkthrough (Alice & Bob)

| Step | Action                        | `delegatee` | Lock Expired? | Can Undelegate? | Can Withdraw? | Result                                |
| ---- | ----------------------------- | ----------- | ------------- | --------------- | ------------- | ------------------------------------- |
| 1    | Alice locks 100 tokens        | Alice       | ❌             | ✅               | ✅             | Works                                 |
| 2    | Alice delegates to Bob        | Bob         | ❌             | ✅               | ❌             | Works                                 |
| 3    | 5 years later                 | Bob         | ✅             | ❌               | ❌             | **Stuck**                             |
| 4    | Alice tries `delegate(alice)` | Bob         | ✅             | ❌               | ❌             | **Reverts: "Delegatee lock expired"** |

## Vulnerable Code Reference

### 1) Withdraw requires undelegation

```solidity
function withdraw() external nonReentrant {
    LockedBalance memory locked_ = locked[msg.sender];
    require(locked_.delegatee == msg.sender, "Lock delegated");
    ...
}
```

### 2) Undelegation blocked by expiry check

```solidity
function delegate(address _addr) external nonReentrant {
    LockedBalance memory locked_ = locked[msg.sender];
    ...
    else if (_addr == msg.sender) {
        fromLocked = locked[delegatee];
        toLocked = locked_;
    }
    require(toLocked.end > block.timestamp, "Delegatee lock expired");
}
```

## Recommended Mitigation

1. **Skip expiry check for self-undelegation**

   ```solidity
   if (_addr != msg.sender) {
       require(toLocked.end > block.timestamp, "Delegatee lock expired");
   }
   ```

   → allows undelegation even after expiry.

2. **Alternative**: Add a small expiry grace window (e.g., 1 second)

   ```solidity
   require(
       _addr == msg.sender || toLocked.end > block.timestamp,
       "Delegatee lock expired"
   );
   ```

3. **Add tests for undelegation edge cases**

   * Test undelegation immediately before and after expiry.
   * Ensure withdrawal succeeds after self-undelegation even when lock expired.

4. **UX-level safeguard**

   * Frontend should remind users to undelegate before expiry or automate undelegation on unlock.

## Pattern Recognition Notes

* **Self-Reference Check Oversight** — Condition meant for delegation also triggered on undelegation because no branch excluded `_addr == msg.sender`.
* **Expiry Time-Lock Misapplied** — Timestamp guards often apply to others' locks, not self-owned ones.
* **Permanent DoS Risk** — Withdraw paths gated by delegation state require safe reversal mechanisms.
* **Mitigation Pattern** — When an invariant (`delegatee == msg.sender`) gates withdrawal, always ensure users can re-enter that invariant under all timing conditions.
* **Preventive Lesson** — Explicitly separate *delegation logic* and *undelegation logic* in control flow to avoid shared conditions that create irreversible states.

### Quick Recall (TL;DR)

* **Bug**: Expired locks can't undelegate because of strict `require(toLocked.end > now)` check.
* **Impact**: User permanently loses access to funds.
* **Fix**: Skip or relax time check for self-undelegation.
* **Severity**: High — permanent account lock.
