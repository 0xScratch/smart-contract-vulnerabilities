# `withdraw()` Locks Stakers With Zero Bonus When Registry Skips `UNDER_ATTACK`

- **Severity**: Medium
- **Source**: [Codehawks](https://codehawks.cyfrin.io/c/2026-07-battlechain-confidence-pools/s/725)
- **Affected Contract**: [ConfidencePool.sol](https://github.com/CodeHawks-Contests/2026-07-bc-confidence-pools/blob/main/src/ConfidencePool.sol)
- **Vulnerability Type**: State Machine / Incorrect State Assumption / Locked Capital

## Summary

`ConfidencePool.withdraw()` permanently disables a staker's ability to exit whenever the associated registry reports a state other than `NOT_DEPLOYED`, `NEW_DEPLOYMENT`, or `ATTACK_REQUESTED`.

This implicitly assumes that reaching a terminal state such as `PRODUCTION` means the pool must previously have observed an active-risk state such as `UNDER_ATTACK`.

However, the underlying `AttackRegistry` allows the agreement to transition to `PRODUCTION` without ever passing through `UNDER_ATTACK`. This can happen either when the promotion window expires without DAO approval or when the agreement owner explicitly calls `goToProduction()`.

In such a case, `riskWindowStart` remains `0`, meaning the pool never observed risk materialization and therefore awards no risk bonus. Despite this, `withdraw()` sees `PRODUCTION` as a non-withdrawable state and permanently locks the staker.

As a result, the protocol can end up in the following inconsistent state:

```text
Risk never materialized
        ↓
riskWindowStart == 0
        ↓
Bonus = 0
        ↓
PRODUCTION
        ↓
withdraw() disabled
        ↓
Staker remains locked without earning a premium
```

This violates the intended economic guarantee that a staker only loses the ability to exit once risk has actually materialized, at which point they begin earning the corresponding risk premium.

## Intended Behavior

The ConfidencePool's withdrawal mechanism is designed around a simple economic tradeoff:

> Before risk materializes, a staker can freely exit. Once risk materializes, the exit closes, but the staker earns the risk premium for remaining exposed.

For example:

```text
Alice stakes 100
      ↓
No risk yet
      ↓
Alice can withdraw 100
```

If risk subsequently materializes:

```text
Alice stakes 100
      ↓
UNDER_ATTACK
      ↓
riskWindowStart is recorded
      ↓
withdraw() becomes disabled
      ↓
Alice remains exposed
      ↓
Protocol survives
      ↓
Alice receives 100 + bonus
```

The withdrawal lock is therefore justified by the fact that the staker has actually entered the risk-bearing period.

## Vulnerable Behavior

The withdrawal condition is effectively:

```solidity
IAttackRegistry.ContractState state = _observePoolState();

if (
    riskWindowStart != 0
    || (
        state != IAttackRegistry.ContractState.NOT_DEPLOYED
        && state != IAttackRegistry.ContractState.NEW_DEPLOYMENT
        && state != IAttackRegistry.ContractState.ATTACK_REQUESTED
    )
) {
    revert WithdrawsDisabled();
}
```

There are two relevant conditions here.

First:

```solidity
riskWindowStart != 0
```

correctly prevents withdrawal once an actual risk window has been observed.

The problem is the second condition.

`PRODUCTION` is not one of the allowed states, so reaching `PRODUCTION` is enough to disable withdrawal even when:

```solidity
riskWindowStart == 0
```

That means the implementation treats:

```text
PRODUCTION after actual risk
```

and:

```text
PRODUCTION without ever observing risk
```

as equivalent.

They are not equivalent from the pool's perspective.

## Why `PRODUCTION` Can Be Reached Without Risk

The underlying `AttackRegistry` does not guarantee that every path to `PRODUCTION` passes through `UNDER_ATTACK`.

### Route A — Promotion Window Expires

When an agreement is registered, its promotion deadline is established using the promotion window.

If the DAO does not approve the attack request before the deadline, the registry can return:

```solidity
if (block.timestamp >= info.deadlineTimestamp) {
    return ContractState.PRODUCTION;
}
```

Therefore the state can evolve as:

```text
ATTACK_REQUESTED
       ↓
14 days pass without DAO approval
       ↓
PRODUCTION
```

No `UNDER_ATTACK` state is observed.

Consequently:

```text
riskWindowStart == 0
```

remains unchanged.

Importantly, this route does not require a malicious actor. The transition can occur simply through the passage of time.

---

### Route B — Direct Promotion

The agreement owner can also call:

```solidity
goToProduction()
```

to transition directly to `PRODUCTION`.

The resulting state transition is:

```text
NEW_DEPLOYMENT
       ↓
goToProduction()
       ↓
PRODUCTION
```

Again, `UNDER_ATTACK` is never reached.

The pool therefore observes:

```text
state = PRODUCTION
riskWindowStart = 0
```

The result is identical from `ConfidencePool`'s perspective.

## Concrete Example

Consider a pool where:

```text
Alice stakes:       100 tokens
Bonus contribution: 50 tokens
```

Alice initially retains the ability to exit.

### Step 1 — Alice stakes

```text
Alice
  ↓
stake(100)
  ↓
ConfidencePool
```

At this point:

```text
riskWindowStart = 0
```

No risk has materialized.

---

### Step 2 — Registry reaches `PRODUCTION` without `UNDER_ATTACK`

For example:

```text
ATTACK_REQUESTED
        ↓
promotion deadline expires
        ↓
PRODUCTION
```

The pool has never observed an active-risk state.

Therefore:

```text
riskWindowStart = 0
```

---

### Step 3 — Alice tries to withdraw

Alice calls:

```solidity
pool.withdraw();
```

The pool observes:

```solidity
state == PRODUCTION
```

But `PRODUCTION` is not included in the permitted withdrawal states.

Therefore:

```solidity
revert WithdrawsDisabled();
```

Alice is now locked.

### Step 4 — No bonus is earned

Because:

```text
riskWindowStart == 0
```

the pool considers that no risk window was observed, so the bonus calculation produces no risk premium for Alice.

Eventually, if the outcome is `SURVIVED`, Alice can recover her principal, but not the bonus:

```text
Alice receives: 100
Bonus:            0
```

The bonus pot can subsequently be swept to the recovery address.

Thus Alice has been denied her exit without receiving the economic compensation that is supposed to justify the lock.

## Intended vs Vulnerable State Transition

### Intended risk-bearing path

```text
             Alice stakes
                  │
                  ▼
          Pre-risk state
                  │
                  ▼
             UNDER_ATTACK
                  │
                  │ riskWindowStart != 0
                  ▼
          Withdrawal locked
                  │
                  ▼
             PRODUCTION
                  │
                  ▼
          Principal + bonus
```

Here the lock is justified because risk actually materialized.

---

### Vulnerable no-risk path

```text
             Alice stakes
                  │
                  ▼
          ATTACK_REQUESTED
                  │
                  │ no risk materializes
                  ▼
             PRODUCTION
                  │
                  │ riskWindowStart == 0
                  ▼
          Withdrawal locked  ← BUG
                  │
                  ▼
          Principal only
                  │
                  ▼
             Bonus = 0
```

The critical difference is that both paths end in `PRODUCTION`, but only one of them actually experienced risk.

## Root Cause

The root cause is an incorrect assumption about the registry's state machine.

`ConfidencePool` effectively treats the registry states as if the lifecycle were:

```text
PRE-RISK
    ↓
UNDER_ATTACK
    ↓
PRODUCTION
```

and therefore assumes that:

```text
PRODUCTION
```

implicitly means:

```text
risk previously materialized
```

However, the actual registry permits multiple paths:

```text
                         ┌──────────────→ PRODUCTION
                         │
ATTACK_REQUESTED ────────┤
                         │
                         └──────────────→ UNDER_ATTACK → ...
                         
NEW_DEPLOYMENT ─────────────────────────→ PRODUCTION
```

Therefore, the current registry state alone does not tell the pool whether risk was ever materialized.

The historical variable:

```solidity
riskWindowStart
```

is what captures that distinction.

The implementation already uses this variable as a permanent withdrawal latch, but the additional current-state check incorrectly closes the exit whenever the registry reaches `PRODUCTION`, even when:

```solidity
riskWindowStart == 0
```

## The Broken Invariant

The intended property is:

> A staker should lose the ability to withdraw only after risk has actually materialized.

This can be expressed as:

```text
riskWindowStart == 0
    ⇒
the staker must not be locked merely because the registry is in
terminal PRODUCTION without observed risk
```

Conversely:

```text
riskWindowStart != 0
    ⇒
withdrawal remains disabled
```

The implementation currently violates the first property.

It effectively uses:

```text
current registry state
```

as a proxy for:

```text
whether risk previously materialized
```

But the two pieces of information are not equivalent.

## Why This Is More Than "PRODUCTION Is Missing"

The superficial bug is that `PRODUCTION` is not included in the withdrawal whitelist.

However, simply looking at the missing enum value hides the actual issue.

There are two semantically different situations:

### `PRODUCTION` after risk

```text
UNDER_ATTACK
      ↓
PRODUCTION

riskWindowStart != 0
```

Withdrawal should remain disabled.

### `PRODUCTION` without risk

```text
ATTACK_REQUESTED
      ↓
PRODUCTION

riskWindowStart == 0
```

Withdrawal should remain available.

Both situations expose the same registry state:

```text
PRODUCTION
```

Therefore, the pool must use historical risk information to distinguish them.

The existing `riskWindowStart` variable already provides exactly that information.

## Impact

A staker can become locked in the pool after the registry reaches `PRODUCTION` even though the pool never observed an active-risk period.

The staker:

- cannot withdraw after the `PRODUCTION` transition;
- earns no risk premium because `riskWindowStart` remains `0`;
- remains locked until the pool reaches a later resolution path such as `SURVIVED` or expiry;
- has their principal ultimately recoverable, but their capital is unnecessarily locked and their expected premium is denied.

On the direct-promotion route, the bonus pot can subsequently be reclaimed through:

```solidity
sweepUnclaimedBonus()
```

meaning the bonus that was intended to compensate stakers for accepting risk can instead end up at the configured recovery address.

The principal is not permanently stolen, which limits the severity to Medium. The primary harm is forced capital lockup combined with the loss of the corresponding risk premium.

## Why `pokeRiskWindow()` Does Not Fix the Issue

`pokeRiskWindow()` cannot retroactively create a risk window.

It can seal the relevant timestamps when an active-risk state is observed, but `PRODUCTION` is already a terminal state and does not constitute an active-risk interval.

Therefore, on the vulnerable path:

```text
PRODUCTION
    ↓
pokeRiskWindow()
```

does not change the fundamental fact that:

```text
riskWindowStart == 0
```

There was never an observed `UNDER_ATTACK` interval to record.

Consequently, the bonus calculation still treats the pool as having observed no risk.

## Recommended Mitigation

Allow withdrawal from `PRODUCTION` only when no risk window has ever been observed.

The existing:

```solidity
riskWindowStart != 0
```

condition should remain the primary permanent lock.

Conceptually, the withdrawal rule should become:

```text
riskWindowStart == 0
AND
(
    state is a normal pre-risk state
    OR
    state == PRODUCTION
)
```

This permits the missing case:

```text
PRODUCTION + no observed risk
```

while preserving the intended lock when risk has actually materialized:

```text
PRODUCTION + riskWindowStart != 0
```

A corresponding change is:

```solidity
IAttackRegistry.ContractState state = _observePoolState();

if (
    riskWindowStart != 0
    || (
        state != IAttackRegistry.ContractState.NOT_DEPLOYED
        && state != IAttackRegistry.ContractState.NEW_DEPLOYMENT
        && state != IAttackRegistry.ContractState.ATTACK_REQUESTED
        && state != IAttackRegistry.ContractState.PRODUCTION
    )
) {
    revert WithdrawsDisabled();
}
```

The important part is **not** simply adding `PRODUCTION` to the whitelist.

The safety property comes from the combination of:

```solidity
state == PRODUCTION
```

and:

```solidity
riskWindowStart == 0
```

If a risk window was previously observed, the first guard still prevents withdrawal regardless of the current registry state.

## Why `CORRUPTED` Should Remain Excluded

`CORRUPTED` should not be treated the same way as no-risk `PRODUCTION`.

A pool may fail to observe an active-risk state while the underlying agreement can still eventually report a real breach.

Therefore:

```text
PRODUCTION + riskWindowStart == 0
        ↓
No risk to settle
        ↓
Withdrawal may remain open
```

is fundamentally different from:

```text
CORRUPTED + riskWindowStart == 0
        ↓
Agreement reports an actual breach
        ↓
Withdrawal must remain protected
```

The mitigation should therefore reopen only the no-risk `PRODUCTION` path and should not reopen a `CORRUPTED` escape route.

## Pattern Recognition

### 1. Current State vs Historical State

Do not assume that a current enum value tells you everything that happened previously.

For state-machine-dependent logic, ask:

```text
What is the current state?
```

and separately:

```text
What states have previously been observed?
```

A terminal state may be reachable through multiple paths with completely different security or economic meanings.

---

### 2. State Enum as a Proxy for a Semantic Property

Be suspicious when code uses:

```text
currentState == X
```

as a proxy for something like:

```text
risk has occurred
funds have been exposed
liquidation has happened
authorization has been consumed
a settlement period has started
```

Those are historical properties and may not be recoverable from the current enum alone.

---

### 3. Multiple Paths Into Terminal States

Whenever a contract has a terminal state such as:

```text
PRODUCTION
COMPLETED
SETTLED
EXPIRED
FINALIZED
```

enumerate **every path** that can reach it.

Do not assume:

```text
A → B → C
```

just because the intended lifecycle looks that way.

Look for:

```text
A → C
A → D → C
B → C
timeout → C
admin action → C
```

Different paths into the same terminal state can require different handling.

---

### 4. Economic Coupling Between Lock and Reward

Whenever a protocol says:

```text
you lose an escape hatch
        ↓
but receive compensation
```

treat those as a coupled invariant.

Here:

```text
withdrawal disabled
        ↔
risk premium earned
```

If you find a path where:

```text
withdrawal disabled
        +
premium == 0
```

that is immediately worth investigating.

## Quick Recall

**Bug:** `withdraw()` disables exits whenever the registry reaches `PRODUCTION`, even if `UNDER_ATTACK` was never observed.

**Root cause:** The pool assumes `PRODUCTION` implies prior risk materialization, but the registry can reach `PRODUCTION` directly or through timeout.

**Key state:** `riskWindowStart == 0` proves that the pool never observed an active-risk window.

**Exploit path:**

```text
STAKE
  ↓
ATTACK_REQUESTED
  ↓
PRODUCTION without UNDER_ATTACK
  ↓
withdraw() reverts
  ↓
riskWindowStart == 0
  ↓
bonus == 0
  ↓
staker remains locked without premium
```

**Impact:** Locked capital + denied risk premium; principal remains recoverable.

**Fix:** Allow `PRODUCTION` withdrawals when `riskWindowStart == 0`, while retaining `riskWindowStart != 0` as the permanent lock once risk has actually materialized.

**Core lesson:**

> **Never use the current state as a substitute for historical state when different paths can reach the same state.**
