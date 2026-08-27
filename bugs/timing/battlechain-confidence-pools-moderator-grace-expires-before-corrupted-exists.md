# Moderator Grace Can Expire Before `CORRUPTED` Exists

- **Severity**: Medium
- **Source**: [Codehawks](https://codehawks.cyfrin.io/c/2026-07-battlechain-confidence-pools/s/201)
- **Affected Contracts**: [ConfidencePool.sol](https://github.com/CodeHawks-Contests/2026-07-bc-confidence-pools/blob/main/src/ConfidencePool.sol)
- **Vulnerability Type**: Timing / State Lifecycle / Incorrect Grace-Period Anchoring

## Summary

`ConfidencePool` provides the pool moderator with a 180-day grace period before a permissionless fallback can automatically finalize an Agreement as `CORRUPTED`.

The purpose of this grace period is to give the moderator an opportunity to classify an actually corrupted Agreement as one of the protocol's supported outcomes:

```text
CORRUPTED + in-scope + good faith
CORRUPTED + bad faith
SURVIVED + out-of-scope corruption
```

However, the fallback's 180-day clock is anchored to **Pool expiry**:

```solidity
expiry + MODERATOR_CORRUPTED_GRACE
```

rather than to the moment when `CORRUPTED` first becomes observable.

An Agreement can remain `UNDER_ATTACK` after the Pool expires and throughout the entire 180-day grace period. While it remains `UNDER_ATTACK`, the moderator cannot classify it as `CORRUPTED` because the `CORRUPTED` state does not yet exist.

If `CORRUPTED` first appears only after the expiry-based grace has already elapsed, the moderator's first opportunity to classify the corruption and the permissionless fallback's opportunity to finalize it become available simultaneously.

A permissionless caller can then execute `claimExpired()` immediately, causing the Pool to mechanically select bad-faith `CORRUPTED`, set `claimsStarted = true`, and permanently prevent the moderator from applying the intended Pool-specific classification.

The core issue is therefore:

```text
Moderator can act
        ↓
only after CORRUPTED exists

Fallback can act
        ↓
after expiry + 180 days

CORRUPTED can appear
        ↓
after expiry + 180 days

Therefore:
CORRUPTED appears → moderator and fallback become actionable simultaneously
```

The moderator receives no protected post-`CORRUPTED` interval.

## Intended Behavior

The 180-day grace period is intended to protect the moderator's ability to classify a corrupted Agreement before a mechanical fallback takes over.

Once an Agreement becomes `CORRUPTED`, the moderator needs to determine what that means for this particular Pool.

The Agreement-level corruption does not by itself determine the Pool outcome.

The moderator may select:

```text
Agreement = CORRUPTED
        │
        ├──→ SURVIVED
        │      corruption was outside Pool scope
        │
        ├──→ CORRUPTED
        │      goodFaith = true
        │      attacker = named attacker
        │
        └──→ CORRUPTED
               goodFaith = false
               attacker = address(0)
```

These classifications have different consequences for the Pool's entire principal and bonus corpus.

Therefore, the intended lifecycle is conceptually:

```text
Agreement becomes CORRUPTED
          ↓
Moderator receives protected window
          ↓
Moderator classifies Pool
          ↓
SURVIVED / good-faith CORRUPTED / bad-faith CORRUPTED
```

Only if the moderator remains unavailable should the mechanical fallback eventually take over.

## Vulnerable Behavior

The fallback in `claimExpired()` uses an absolute threshold based on Pool expiry:

```solidity
if (
    state == IAttackRegistry.ContractState.CORRUPTED
    && riskWindowStart != 0
) {
    if (
        block.timestamp <
        expiry + MODERATOR_CORRUPTED_GRACE
    ) {
        revert AgreementCorruptedAwaitingModerator();
    }

    outcome = PoolStates.Outcome.CORRUPTED;
    outcomeFlaggedAt = riskWindowEnd;
    corruptedReserve = snapshotTotalStaked + snapshotTotalBonus;
    claimsStarted = true;

    return;
}
```

The critical condition is:

```solidity
block.timestamp < expiry + MODERATOR_CORRUPTED_GRACE
```

This measures:

```text
time since Pool expiry
```

It does **not** measure:

```text
time since CORRUPTED became observable
```

That distinction creates the vulnerability.

## Why the Moderator Cannot Act Before `CORRUPTED`

The moderator's `flagOutcome()` path is intentionally restricted by the live Agreement state.

While the Agreement remains:

```text
UNDER_ATTACK
```

the moderator cannot select the `CORRUPTED`-dependent outcomes.

In particular:

```text
flagOutcome(SURVIVED, ...)
        ↓
InvalidOutcome

flagOutcome(CORRUPTED, ...)
        ↓
InvalidOutcome
```

This makes sense because there is no `CORRUPTED` state to classify yet.

The moderator therefore has no way to "use" the 180-day period while the Agreement is still `UNDER_ATTACK`.

This creates the critical mismatch:

```text
Grace clock:
    starts at Pool expiry

Moderator's protected action:
    becomes possible at CORRUPTED
```

Those events are not guaranteed to occur in that order.

## How the Grace Period Can Fully Elapse

Consider:

```text
Pool expiry = Day 90
MODERATOR_CORRUPTED_GRACE = 180 days
```

The mechanical fallback therefore becomes eligible at:

```text
Day 90 + 180 days
= Day 270
```

Now suppose the Agreement enters `UNDER_ATTACK` before Pool expiry but remains there for an extended period.

The resulting lifecycle can be:

```text
Day 80
  ↓
UNDER_ATTACK

Day 90
  ↓
Pool expires
  ↓
Agreement still UNDER_ATTACK

Day 90 → Day 270
  ↓
Agreement remains UNDER_ATTACK
  ↓
Moderator cannot classify CORRUPTED

Day 270
  ↓
Expiry-based grace expires

Day 271
  ↓
Agreement finally becomes CORRUPTED
```

At Day 271, the moderator's first valid opportunity to classify `CORRUPTED` appears.

But the fallback's timer has already expired.

Therefore:

```text
CORRUPTED first appears
        │
        ├──→ Moderator can finally act
        │
        └──→ claimExpired() can immediately finalize
```

There is no additional grace period.

## Concrete Example

Consider a Pool containing:

```text
Staker principal: 100 tokens
Bonus:             50 tokens
--------------------------------
Total corpus:     150 tokens
```

### Step 1 — Risk Materializes

The Agreement enters:

```text
UNDER_ATTACK
```

and the Pool observes the active-risk state.

Therefore:

```text
riskWindowStart != 0
```

The staker's withdrawal is now disabled as intended.

---

### Step 2 — Pool Expires

The Pool reaches its expiry while the Agreement remains:

```text
UNDER_ATTACK
```

The Pool is still unresolved.

At this point, the moderator cannot classify `CORRUPTED` because the Agreement has not become `CORRUPTED`.

The Pool can nevertheless be permissionlessly resolved through the ordinary `claimExpired()` path while the Agreement remains active-risk. If nobody calls it, however, the Pool remains unresolved.

---

### Step 3 — The Entire Grace Period Passes

The Pool reaches:

```text
expiry + MODERATOR_CORRUPTED_GRACE
```

while the Agreement is still:

```text
UNDER_ATTACK
```

Therefore:

```text
Moderator:
    cannot classify CORRUPTED

Fallback:
    corruption grace has expired
```

The entire 180-day protection period has been consumed without the moderator ever having a `CORRUPTED` state to classify.

---

### Step 4 — `CORRUPTED` Finally Appears

After the grace threshold has already elapsed, the Agreement's attack moderator calls:

```solidity
markCorrupted(agreement);
```

The registry now reports:

```text
CORRUPTED
```

This is the first moment at which the moderator can legitimately make the Pool-specific corruption classification.

But the fallback is already unlocked.

No additional time passes.

---

### Step 5 — Permissionless Caller Finalizes the Pool

A permissionless caller executes:

```solidity
pool.claimExpired();
```

Because:

```text
state == CORRUPTED
riskWindowStart != 0
block.timestamp >= expiry + MODERATOR_CORRUPTED_GRACE
```

the mechanical branch executes immediately:

```text
outcome = CORRUPTED
claimsStarted = true
```

The fallback has no ability to determine:

```text
Was the corruption inside Pool scope?
Was it good faith?
Who was the attacker?
```

Therefore it mechanically selects:

```text
bad-faith CORRUPTED
```

## Moderator's Classification Is Permanently Lost

Once `claimExpired()` sets:

```solidity
claimsStarted = true;
```

the Pool outcome becomes final.

The moderator can no longer call:

```solidity
flagOutcome(...)
```

to select the correct Pool-local classification.

For example, the moderator can no longer change the result to:

```text
SURVIVED
```

if the corruption was outside the Pool's scope.

Nor can the moderator select:

```text
CORRUPTED
goodFaith = true
attacker = Alice
```

if the corruption was a legitimate good-faith disclosure.

Instead, the mechanical fallback has already committed the Pool to:

```text
CORRUPTED
goodFaith = false
attacker = address(0)
```

The fallback therefore consumes the moderator's first available decision opportunity.

## Complete Vulnerable Flow

```text
                  Pool expiry
                      │
                      ▼
              180-day grace starts
                      │
                      │
                      │ Agreement remains
                      │ UNDER_ATTACK
                      │
                      ▼
              180 days elapse
                      │
                      │
                      ▼
            CORRUPTED finally appears
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
        Moderator can      claimExpired()
        finally classify      can already
        the Agreement         finalize
             │                 │
             │                 ▼
             │          bad-faith CORRUPTED
             │                 │
             │                 ▼
             │          claimsStarted = true
             │                 │
             ▼                 ▼
          too late       moderator locked out
```

The intended lifecycle should instead be:

```text
Pool expiry
     │
     ▼
Agreement eventually becomes CORRUPTED
     │
     ▼
CORRUPTED observation recorded
     │
     │ 180-day moderator window
     ▼
Moderator classification
     │
     └── if unavailable ──→ mechanical fallback
```

## Why `UNDER_ATTACK` Is Not the Bug

The fact that the Pool can expire while the Agreement remains `UNDER_ATTACK` is not itself incorrect.

The Pool's insured period can legitimately end while the upstream Agreement remains attackable.

The issue arises from combining that valid lifecycle with the expiry-anchored corruption grace:

```text
Pool expires
        +
Agreement remains UNDER_ATTACK
        +
expiry-based grace elapses
        +
Agreement later becomes CORRUPTED
```

The problem is that the fallback's clock can finish before the event that makes the moderator's protected action possible.

## Root Cause

The root cause is that the moderator grace period is anchored to the wrong lifecycle event.

The contract measures:

```text
Pool expiry
    ↓
180 days
    ↓
mechanical CORRUPTED fallback
```

when it should protect the moderator based on:

```text
first observable CORRUPTED
    ↓
moderator grace
    ↓
mechanical CORRUPTED fallback
```

The Pool does not record when `CORRUPTED` first became observable.

As a result, the fallback only knows:

```solidity
block.timestamp >= expiry + MODERATOR_CORRUPTED_GRACE
```

rather than:

```text
CORRUPTED has existed for at least the moderator grace period
```

The entire grace period can therefore elapse before the moderator's classification question even exists.

## The Broken Invariant

The intended security property is:

> **Once `CORRUPTED` becomes observable, the moderator must receive a protected interval to classify the Pool before the permissionless fallback can finalize it.**

Equivalently:

```text
CORRUPTED first observed
        ↓
moderator grace period
        ↓
fallback becomes available
```

The implementation instead permits:

```text
expiry
   ↓
moderator grace period
   ↓
CORRUPTED first observed
   ↓
fallback immediately available
```

The broken invariant can therefore be expressed as:

```text
fallback_available
    ⇒
CORRUPTED has been observable for the required moderator interval
```

This property is not guaranteed by the current implementation.

## Moderator Decisions Lost to the Fallback

Once `CORRUPTED` exists, the moderator has three materially different classifications available.

### 1. `SURVIVED` — Out-of-Scope Corruption

The Agreement can be `CORRUPTED` while the actual breached account lies outside the Pool's fixed scope.

The moderator can therefore classify:

```text
SURVIVED
```

and stakers recover their principal plus bonus.

The mechanical fallback cannot make this determination.

---

### 2. Good-Faith `CORRUPTED`

The moderator can identify a legitimate attacker:

```text
CORRUPTED
goodFaith = true
attacker = attackerAddress
```

This activates the corresponding bounty path.

Again, the permissionless fallback cannot identify or name a good-faith attacker.

---

### 3. Bad-Faith `CORRUPTED`

The moderator can also deliberately classify:

```text
CORRUPTED
goodFaith = false
attacker = address(0)
```

which sends the pool through the recovery path.

The mechanical fallback can only safely choose this pessimistic branch because it has no Pool-local information with which to select the other two classifications.

Therefore, once the fallback wins the race:

```text
SURVIVED
```

and:

```text
good-faith CORRUPTED
```

are permanently lost.

## Control Scenario

The vulnerability is especially clear when the order of the two decisive operations is reversed.

### Vulnerable Ordering

```text
1. Agreement becomes CORRUPTED
2. claimExpired()
```

Because the expiry-based grace has already elapsed:

```text
CORRUPTED
→ automatic bad-faith CORRUPTED
→ claimsStarted = true
→ recovery path
```

### Control Ordering

```text
1. claimExpired()
2. Agreement becomes CORRUPTED
```

At the time of `claimExpired()` the Agreement is still:

```text
UNDER_ATTACK
```

Therefore the Pool resolves through the normal:

```text
EXPIRED
```

branch.

The later `CORRUPTED` transition cannot change the already-resolved Pool.

Thus the same Pool corpus can follow different destinations solely because of the order in which:

```text
claimExpired()
```

and:

```text
markCorrupted()
```

occur.

This demonstrates that the live Agreement state at the exact resolution moment is security-critical.

## Impact

The mechanical fallback can irreversibly determine the destination of the Pool's entire principal and bonus corpus before the independent moderator has a meaningful opportunity to classify the newly-created `CORRUPTED` state.

For example:

```text
Principal: 100 tokens
Bonus:      50 tokens
----------------------
Corpus:    150 tokens
```

The late-`CORRUPTED` path can cause:

```text
150 tokens → recoveryAddress
```

whereas the moderator might otherwise have determined that:

```text
150 tokens → staker
```

through `SURVIVED`, or directed the corpus through the appropriate good-faith corruption path.

Once:

```solidity
claimsStarted = true;
```

the moderator's classification authority is permanently lost.

The impact is therefore the irreversible loss of the moderator's Pool-specific outcome decision over the entire principal-and-bonus corpus.

## Likelihood

Low.

The Pool must remain unresolved through the entire 180-day expiry-based grace period, the Agreement must remain `UNDER_ATTACK` during that time, and `CORRUPTED` must only appear after the fallback threshold has already elapsed.

A permissionless caller must then invoke `claimExpired()` before the moderator can classify the newly-created `CORRUPTED` state.

## Recommended Mitigation

Record the first observation of `CORRUPTED` in a dedicated timestamp.

For example:

```text
corruptedObservedAt
```

The timestamp should be distinct from the existing risk-window markers.

The first observation of:

```text
state == CORRUPTED
```

should record the timestamp but **must not simultaneously finalize the Pool**.

The mechanical fallback should then require both:

```text
1. Existing expiry-based minimum has elapsed
```

and:

```text
2. Moderator grace period has elapsed since
   corruptedObservedAt
```

Conceptually:

```text
Pool expiry
    │
    │ existing minimum
    ▼
minimum threshold

Agreement becomes CORRUPTED
    │
    │ corruptedObservedAt
    │
    │ moderator grace
    ▼
mechanical fallback
```

This guarantees that the moderator receives an actual post-`CORRUPTED` interval in which to make the Pool-local classification.

The observation transaction must not also finalize the bad-faith `CORRUPTED` branch, otherwise a permissionless caller could still consume the newly-created observation and finalization opportunity in the same transaction.

## Why `riskWindowEnd` Should Not Be Reused

The existing `riskWindowEnd` should not be repurposed as the moderator-classification timestamp.

`riskWindowEnd` is associated with the Pool's risk-window and bonus-distribution lifecycle and is capped at Pool expiry.

A late `CORRUPTED` observation would therefore inherit an already-expired timestamp.

The moderator-classification clock represents a different semantic event:

```text
riskWindowEnd
    =
end of risk/bonus timing

corruptedObservedAt
    =
first time CORRUPTED became available for moderator classification
```

These should remain separate.

## Pattern Recognition

### 1. Grace Period Anchoring

Whenever a protocol provides a grace period for a privileged/manual action, ask:

> **What event makes the protected action possible, and does the grace period start from that same event?**

For this vulnerability:

```text
Protected action:
    flagOutcome(CORRUPTED)

Action becomes possible:
    Agreement → CORRUPTED

Grace starts:
    Pool expiry
```

The mismatch is the bug.

---

### 2. Manual Path vs Permissionless Fallback

Whenever you see:

```text
manual resolution
        +
permissionless fallback
```

analyze both paths together.

Ask:

```text
When does the manual path become available?
When does the fallback become available?
Can the fallback become available before the manual path?
```

If yes, investigate whether the fallback can consume the state before the privileged actor gets its intended opportunity.

---

### 3. Timeout Based on an Earlier Lifecycle Event

Do not assume that a timeout is safe merely because the duration itself looks generous.

A 180-day timeout is meaningless if:

```text
the protected event happens after those 180 days
```

The important property is not:

```text
"there is a 180-day delay"
```

but:

```text
"there is a 180-day delay after the protected decision becomes possible."
```

---

### 4. Multiple Clocks in One State Machine

When a protocol contains multiple timestamps such as:

```text
expiry
riskWindowStart
riskWindowEnd
corruptedObservedAt
deadline
gracePeriod
```

map each timestamp to the event it represents.

Then ask:

```text
Which permission does this timestamp gate?
What event starts it?
Can that event happen before or after the event that makes the permission relevant?
```

Many lifecycle bugs arise from using a valid timestamp for the wrong semantic clock.

## Quick Recall

**Bug:** The 180-day `CORRUPTED` grace is measured from Pool expiry rather than from the first observation of `CORRUPTED`.

**Exploit path:**

```text
Pool expires
      ↓
Agreement remains UNDER_ATTACK
      ↓
180-day grace elapses
      ↓
Moderator still cannot classify CORRUPTED
      ↓
Agreement finally becomes CORRUPTED
      ↓
claimExpired() is already unlocked
      ↓
bad-faith CORRUPTED finalized immediately
      ↓
claimsStarted = true
      ↓
moderator permanently loses classification opportunity
```

**Impact:** The fallback can irreversibly select the wrong Pool outcome and determine the destination of the entire principal-and-bonus corpus before the moderator gets a protected post-`CORRUPTED` window.

**Fix:** Record `corruptedObservedAt` and require the moderator grace to elapse from that observation, while retaining the existing expiry threshold as a separate minimum.

**Core lesson:**

> **A grace period must be anchored to the event that makes the protected action possible, not merely to an earlier lifecycle deadline.**
