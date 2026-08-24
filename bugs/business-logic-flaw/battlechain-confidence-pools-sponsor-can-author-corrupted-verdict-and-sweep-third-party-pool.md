# Sponsor Can Author `CORRUPTED` Verdict and Sweep the Third-Party Pool

- **Severity**: Medium
- **Source**: [Codehawks](https://codehawks.cyfrin.io/c/2026-07-battlechain-confidence-pools/s/405)
- **Affected Contracts**:
    - [ConfidencePoolFactory.sol](https://github.com/CodeHawks-Contests/2026-07-bc-confidence-pools/blob/main/src/ConfidencePoolFactory.sol)
    - [ConfidencePool.sol](https://github.com/CodeHawks-Contests/2026-07-bc-confidence-pools/blob/main/src/ConfidencePool.sol)
- **Vulnerability Type**: Business Logic / Broken Separation of Powers / Privilege Composition

## Summary

The Confidence Pool is designed to separate the authority that determines a pool's outcome from the sponsor who benefits from the pool's `recoveryAddress`.

The pool has an independent `outcomeModerator` that is supposed to decide whether the pool survived or was corrupted. Separately, the sponsor controls the pool's recovery destination.

However, in the pinned BattleChain integration, the agreement's `attackModerator` is initialized as the agreement owner. `ConfidencePoolFactory.createPool()` requires the pool sponsor to also be the agreement owner.

This creates the following role collision:

```text
agreement owner
      =
pool sponsor
      =
attackModerator
      =
party controlling recoveryAddress
```

The sponsor can therefore call `markCorrupted()` on the agreement and make the trusted registry report `CORRUPTED` without the independent Confidence Pool moderator making the corresponding outcome decision.

If the pool has already observed an active-risk window and the independent moderator does not intervene during the 180-day corruption grace period, `claimExpired()` automatically converts the upstream `CORRUPTED` state into a terminal pool `CORRUPTED` outcome. This sets `claimsStarted`, preventing the independent moderator from later correcting the outcome.

A subsequent `claimCorrupted()` then transfers the entire pool balance — including third-party staker principal and bonus — to the sponsor-controlled `recoveryAddress`.

The resulting flow is:

```text
Sponsor
   ↓
attackModerator
   ↓
markCorrupted()
   ↓
registry = CORRUPTED
   ↓
180-day moderator grace expires
   ↓
claimExpired()
   ↓
pool outcome = CORRUPTED
   ↓
claimCorrupted()
   ↓
entire third-party pool → sponsor recoveryAddress
```

The sponsor does not need to have any capital at risk.

## Intended Behavior

The Confidence Pool intentionally separates two powers.

The sponsor is responsible for creating the pool and choosing the recovery destination:

```text
Sponsor
   ├── creates pool
   └── controls recoveryAddress
```

An independent factory-configured `outcomeModerator` is responsible for determining the pool's final outcome:

```text
Independent moderator
   └── flagOutcome(SURVIVED / CORRUPTED)
```

The intended corruption path is therefore conceptually:

```text
Agreement becomes genuinely corrupted
          ↓
Independent pool moderator
          ↓
flagOutcome(CORRUPTED)
          ↓
claimCorrupted()
          ↓
recoveryAddress
```

The 180-day auto-`CORRUPTED` mechanism exists as a backstop when the independent moderator is unavailable.

The important assumption behind that fallback is that the upstream `CORRUPTED` state is sufficiently trustworthy to justify eventually sweeping the pool.

## The Role Collision

The vulnerability begins in `ConfidencePoolFactory.createPool()`.

The factory requires:

```solidity
if (IAgreement(agreement).owner() != msg.sender) {
    revert UnauthorizedCreator();
}
```

Therefore:

```text
msg.sender
    =
pool sponsor
    =
agreement owner
```

The pool is then initialized using the sponsor as its owner while the recovery address is sponsor-controlled.

At the same time, the pinned Safe Harbor integration initializes the agreement-level `attackModerator` to the agreement owner.

Consequently, the same account occupies multiple roles:

```text
                    Sponsor
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
   Agreement Owner  Attack      Recovery
                    Moderator    Beneficiary
          │
          ▼
     Pool Sponsor
```

The independent `outcomeModerator` is a different account, but it does not control the upstream `CORRUPTED` state.

This is the critical trust-boundary mismatch.

## What the Attack Moderator Can Do

The agreement-level `attackModerator` can call:

```solidity
registry.markCorrupted(agreement);
```

This changes the agreement's registry state to:

```text
CORRUPTED
```

Crucially, `markCorrupted()` is an authority-based state transition. The relevant issue is not that the registry itself is malfunctioning.

The problem is that the account authorized to produce this state can be the same economic actor that benefits from the pool's subsequent corruption sweep.

The resulting relationship is:

```text
Sponsor
   │
   ├── controls attackModerator
   │
   │   markCorrupted()
   │
   ▼
registry = CORRUPTED
```

while simultaneously:

```text
Sponsor
   │
   └── controls recoveryAddress
```

Thus the sponsor can author the state that eventually causes funds to be sent to its own recovery destination.

## Concrete Example

Consider:

```text
Sponsor / attackModerator: Alice
Independent pool moderator: Bob
Recovery address: Alice-controlled address
```

Alice creates the pool.

Third parties then provide:

```text
Staker A:          100 tokens
Staker B:           50 tokens
Bonus contributor:  50 tokens
--------------------------------
Total pool:        200 tokens
```

Alice contributes:

```text
0 tokens
```

So the entire pool consists of third-party capital.

---

### Step 1 — Risk Materializes

The agreement reaches:

```text
UNDER_ATTACK
```

and the pool observes this state.

Therefore:

```text
riskWindowStart != 0
```

This is important because the corruption fallback requires the pool to have observed an active-risk window.

Stakers are now committed and cannot simply withdraw:

```solidity
pool.withdraw();
```

reverts with:

```text
WithdrawsDisabled
```

The third-party capital is therefore available for the later corruption settlement.

---

### Step 2 — Sponsor Authors `CORRUPTED`

Alice, who is also the agreement's `attackModerator`, calls:

```solidity
registry.markCorrupted(agreement);
```

The registry now reports:

```text
CORRUPTED
```

No independent `ConfidencePool.outcomeModerator` decision has occurred.

The important distinction is:

```text
Registry decision:
    agreement = CORRUPTED

Pool decision:
    pool should sweep as CORRUPTED
```

The first has been authored by Alice.

The second is supposed to belong to Bob.

---

### Step 3 — Independent Moderator Does Not Act

The pool does not immediately auto-finalize the corruption.

Instead, the independent moderator receives the defined grace period in which they can still resolve the pool.

During this period:

```text
registry = CORRUPTED
pool outcome = not yet finalized
```

Bob could still call:

```solidity
flagOutcome(...)
```

and determine the actual pool outcome.

The attack therefore depends on Bob remaining inactive until the 180-day corruption grace period expires.

---

### Step 4 — Auto-CORRUPTED Backstop Activates

After the grace period, the following conditions hold:

```text
state == CORRUPTED
riskWindowStart != 0
block.timestamp >= expiry + MODERATOR_CORRUPTED_GRACE
```

`ConfidencePool.claimExpired()` then executes the automatic fallback:

```solidity
outcome = PoolStates.Outcome.CORRUPTED;
claimsStarted = true;
```

The pool has now accepted the sponsor-authored registry state as its terminal outcome.

No independent moderator decision was required.

## The Moderator Is Now Locked Out

The `claimsStarted` latch makes the outcome final.

If the independent moderator subsequently attempts:

```solidity
pool.flagOutcome(
    PoolStates.Outcome.SURVIVED,
    false,
    address(0)
);
```

the call reverts with:

```text
OutcomeAlreadySet
```

Therefore the sequence is:

```text
Sponsor authors CORRUPTED
          ↓
Moderator gets 180-day opportunity
          ↓
Moderator remains silent
          ↓
claimExpired()
          ↓
claimsStarted = true
          ↓
Pool outcome permanently becomes CORRUPTED
          ↓
Moderator can no longer correct it
```

The independent moderator has therefore been structurally bypassed by the fallback.

## Step 5 — Entire Pool Is Swept

Once the pool's outcome is:

```text
CORRUPTED
```

a caller can execute:

```solidity
pool.claimCorrupted();
```

The pool transfers the corruption balance to:

```text
recoveryAddress
```

which is controlled by the sponsor.

With the example balances:

```text
Staker A       100
Staker B        50
Bonus           50
-------------------
Total           200
```

the recovery address receives:

```text
200 tokens
```

while:

```text
Sponsor stake = 0
```

The sponsor therefore does not need to risk capital in order to obtain the third-party pool.

## Complete Attack Flow

```text
                 Sponsor
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
   attackModerator       recoveryAddress
          │                   │
          │                   │
          ▼                   │
   markCorrupted()            │
          │                   │
          ▼                   │
 registry = CORRUPTED         │
          │                   │
          ▼                   │
  180-day grace               │
          │                   │
          │ moderator silent  │
          ▼                   │
    claimExpired()            │
          │                   │
          ▼                   │
 outcome = CORRUPTED          │
 claimsStarted = true         │
          │                   │
          ▼                   │
   claimCorrupted() ──────────┘
          │
          ▼
  entire third-party pool
```

The independent pool moderator is absent from the finalization path.

## Why the Existing Separation of Powers Fails

At first glance, the system appears to have proper separation:

```text
Sponsor
    ≠
outcomeModerator
```

But that is not enough.

The actual authority chain is:

```text
                    CORRUPTED
                       ↑
                       │
                attackModerator
                       ↑
                       │
                    Sponsor
                       │
                       │
                       ▼
              recoveryAddress
```

The sponsor controls both:

1. the authority capable of creating the upstream `CORRUPTED` state; and
2. the destination that receives the pool after the fallback accepts that state.

The independent moderator only controls the normal pool-level outcome path.

The vulnerability therefore arises from **composing otherwise-valid roles across the external registry and the pool**.

## Root Cause

The root cause is an unsafe trust relationship between the upstream `CORRUPTED` state and the pool's automatic settlement mechanism.

`ConfidencePool` eventually treats:

```text
registry.getAgreementState(agreement) == CORRUPTED
```

as sufficient evidence to finalize the pool as:

```text
PoolStates.Outcome.CORRUPTED
```

after the moderator grace period.

That assumption is unsafe because, in this integration:

```text
registry CORRUPTED authority
        =
agreement owner
        =
pool sponsor
```

while:

```text
CORRUPTED settlement beneficiary
        =
sponsor-controlled recoveryAddress
```

Therefore the party capable of authoring the state that triggers the fallback is economically aligned with the beneficiary of the fallback.

The independent `outcomeModerator` is not part of that chain once the grace period expires.

## The Broken Security Property

The important property is not simply:

> "Only the moderator should be able to call `flagOutcome()`."

That property already holds.

The deeper property is:

> **A party that can unilaterally manufacture the upstream condition used by the corruption fallback must not also control the beneficiary of the resulting full-pool sweep.**

The current architecture violates this:

```text
Can author CORRUPTED
        +
Controls recoveryAddress
        +
Moderator can time out
        ↓
Can eventually cause full-pool sweep
```

The system therefore has a broken separation of powers even though the pool's explicit moderator role remains independent.

## Why This Is Not a Malicious Registry Issue

This vulnerability does not require the trusted BattleChain registry itself to be compromised.

The registry can behave exactly as designed:

```text
attackModerator calls markCorrupted()
        ↓
registry reports CORRUPTED
```

The issue is that the integration allows:

```text
attackModerator == sponsor
```

and therefore the sponsor can legitimately invoke an authority that the Confidence Pool subsequently treats as an autonomous settlement signal.

The problem is **role composition**, not corruption of the registry's own trust model.

## Why This Is Not Simply the Scope-Blind Fallback

The 180-day auto-`CORRUPTED` mechanism is intentionally pessimistic.

Its purpose is to prevent a genuine corruption from remaining unresolved forever when the independent moderator is unavailable.

The intended fallback is:

```text
genuine CORRUPTED
        ↓
moderator unavailable
        ↓
auto-CORRUPTED
```

The vulnerable situation is different:

```text
sponsor-controlled attackModerator
        ↓
fabricated CORRUPTED
        ↓
moderator unavailable
        ↓
auto-CORRUPTED
```

The problem is therefore not merely that the fallback is scope-blind.

The problem is that the **beneficiary can manufacture the input that activates the fallback**.

## Impact

A successful attack transfers the entire unresolved pool balance to the sponsor-controlled recovery address.

This can include:

```text
third-party staker principal
+
third-party bonus contributions
```

The sponsor does not need to contribute capital.

For example:

```text
Third-party principal: 150
Bonus:                  50
Sponsor capital:         0
--------------------------------
Amount swept:           200
```

The pool size is not inherently limited by the sponsor's own stake.

Victims cannot withdraw after the pool has observed the active-risk state, and their `claimExpired()` attempts remain blocked during the corruption grace period.

Once `claimExpired()` finalizes the pool as `CORRUPTED`, `claimsStarted` prevents the independent moderator from correcting the outcome.

The principal is therefore exposed to a full unauthorized sweep.

## Likelihood

The path requires several conditions to align:

```text
1. Sponsor retains the agreement-level attackModerator role
2. Agreement enters active risk
3. Pool observes the risk window
4. Sponsor calls markCorrupted()
5. Independent moderator does not resolve the pool
6. 180-day corruption grace expires
7. Someone calls claimExpired()
8. claimCorrupted() is executed
```

Therefore the likelihood is relatively low under the stated assumptions.

However, once these conditions are satisfied, the resulting loss is the entire third-party pool.

## Recommended Mitigation

The automatic `CORRUPTED` fallback should not finalize a pool solely because:

```solidity
getAgreementState(agreement) == CORRUPTED
```

when the authority capable of producing that state may also be the pool sponsor or recovery beneficiary.

After the corruption grace period, the pool should require an independent authority to authorize the full-principal corruption settlement.

Conceptually, the current fallback:

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
    claimsStarted = true;
    return;
}
```

should no longer autonomously finalize the pool based solely on the registry state.

Instead:

```text
CORRUPTED
   +
moderator unavailable
        ↓
independent fallback authority
        ↓
verify/authorize corruption
        ↓
pool becomes CORRUPTED
```

The fallback authority should be disjoint from:

```text
agreement owner
pool sponsor
recovery beneficiary
```

A factory/DAO-appointed settler or an appropriate registry attestation that the sponsor cannot manufacture would preserve the liveness property without allowing the beneficiary to author its own payout condition.

## Pattern Recognition

### 1. Authority → Trigger → Beneficiary

Whenever a protocol has:

```text
Authority A
    ↓
creates condition X
    ↓
condition X triggers payout
    ↓
beneficiary B
```

ask:

```text
Can A == B?
```

If yes, investigate immediately.

The dangerous composition here is:

```text
Sponsor
  ↓
can create CORRUPTED
  ↓
CORRUPTED triggers sweep
  ↓
sponsor receives sweep
```

---

### 2. External State as a Settlement Oracle

Be especially careful when a contract uses an external state as the basis for irreversible value movement:

```text
external state
     ↓
automatic settlement
     ↓
fund transfer
```

Ask:

> **Who can cause the external state to become the value-moving condition?**

The answer may be different from the authority that normally controls the final settlement.

---

### 3. Timeout Fallbacks Can Bypass Normal Authorization

A timeout often looks harmless:

```text
Moderator normally decides
        ↓
if unavailable for 180 days
        ↓
fallback decides
```

But the fallback may introduce a completely different trust model.

Always analyze:

```text
normal path
vs
timeout path
```

and ask:

> Does the fallback still preserve the same separation of powers as the normal path?

Here, the normal path requires the independent moderator, while the timeout path ultimately requires only the sponsor-authored registry state.

---

### 4. Role Collisions Across Contracts

Do not analyze roles only inside one contract.

A safe-looking arrangement such as:

```text
pool.owner != outcomeModerator
```

does not guarantee separation if:

```text
pool.owner
    =
externalAgreement.owner
    =
externalAgreement.attackModerator
```

Cross-contract role composition can collapse supposedly independent authorities.

## Quick Recall

**Bug:** The pool's automatic `CORRUPTED` fallback trusts an upstream state that the sponsor can author through the agreement's `attackModerator` role.

**Role collision:**

```text
agreement owner
    =
pool sponsor
    =
attackModerator
    =
recovery beneficiary
```

**Exploit:**

```text
UNDER_ATTACK
      ↓
riskWindowStart != 0
      ↓
sponsor calls markCorrupted()
      ↓
registry = CORRUPTED
      ↓
moderator remains silent for 180 days
      ↓
claimExpired()
      ↓
pool outcome = CORRUPTED
      ↓
claimsStarted = true
      ↓
claimCorrupted()
      ↓
entire third-party pool → sponsor recoveryAddress
```

**Impact:** Full third-party pool balance can be swept without sponsor capital being at risk.

**Likelihood:** Low because the attack requires active risk, sponsor-controlled `attackModerator`, moderator inactivity through the grace period, and finalization.

**Core lesson:**

> **When an external state can trigger an irreversible payout, verify that the party capable of creating that state cannot also be the beneficiary of the payout.**

The vulnerability is fundamentally a **broken separation of powers caused by cross-contract role composition**.
