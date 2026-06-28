# Off-by-One Timestamp Validation Allows Post-Draw Ticket Purchase

* **Severity**: Medium
* **Source**: [Code4rena](/bugs/insecure-randomness/wenwin-off-by-one-timestamp-validation-allows-post-draw-ticket-purchase.md)
* **Affected Contracts**:

  * [Lottery.sol](https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/Lottery.sol#L135)
  * [LotterySetup.sol](https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/LotterySetup.sol#L114)
* **Vulnerability Type**: Business Logic / Off-by-One Error / Time-of-Check Time-of-Use (TOCTOU)

## Summary

The lottery relies on **timestamps** to separate two critical phases of every draw:

1. **Ticket registration phase** (users may purchase tickets).
2. **Draw execution phase** (the contract requests randomness and determines the winning ticket).

However, two incorrect comparison operators (`<` instead of `<=`, and `>` instead of `>=`) create a one-second overlap where **both phases are simultaneously valid**.

As a result, an attacker can:

1. Trigger the lottery draw.
2. Wait for the random number to be generated.
3. Read the winning ticket from on-chain storage.
4. Purchase a ticket containing the already-known winning numbers.
5. Immediately claim the jackpot.

The randomness itself remains secure. The vulnerability exists because the protocol **allows ticket purchases after randomness has already been requested and fulfilled.**

## A Better Explanation (With Simplified Example)

### Intended Behavior

Every lottery round should follow this strict sequence:

```
Ticket Sales Open
        │
        ▼
Ticket Sales Close
        │
        ▼
Request Random Number
        │
        ▼
Receive Random Number
        │
        ▼
Determine Winner
        │
        ▼
Claim Rewards
```

The important security invariant is:

> **No one should be able to buy tickets after the winning numbers become knowable.**

### What Actually Happens (Bug)

The protocol uses two timestamp checks.

#### Draw execution

```solidity
if (block.timestamp < drawScheduledAt(currentDraw))
    revert;
```

Because `<` is used instead of `<=`, calling `executeDraw()` is allowed **exactly at** `drawScheduledAt`.

#### Ticket purchases

```solidity
if (block.timestamp > ticketRegistrationDeadline(drawId))
    revert;
```

Because `>` is used instead of `>=`, ticket purchases remain valid **exactly at** the registration deadline.

This means that at one specific timestamp:

```
block.timestamp == drawScheduledAt == ticketRegistrationDeadline
```

both operations are simultaneously allowed.

```
executeDraw()     ✔ Allowed
buyTickets()      ✔ Allowed
```

This should never happen.

The protocol accidentally creates a window where the draw can begin while ticket registration is still open.

## Why This Matters

Normally, lottery systems rely on a very simple guarantee:

> Once randomness is requested (or known), ticket sales must already be closed.

Breaking this ordering completely destroys the fairness of the lottery.

If an attacker can observe the winning numbers before purchasing tickets, they no longer need luck—they simply buy the winning ticket and drain the jackpot.

The vulnerability therefore compromises the core security assumption of the protocol rather than exploiting Chainlink or the randomness source itself.

## Concrete Walkthrough (Alice & Mallory)

Assume:

```
Ticket registration deadline = 100
Draw scheduled at            = 100
Cooldown period              = 0
```

### Honest users

Alice, Bob and Charlie purchase tickets before the deadline.

```
Alice
Ticket: 3, 7, 14, 19

Bob
Ticket: 1, 5, 12, 18

Charlie
Ticket: 2, 9, 16, 20
```

Everything is normal so far.

### Mallory waits

Mallory intentionally buys nothing.

She simply waits until

```
block.timestamp = 100
```

### Step 1 — Start the draw

Mallory calls

```solidity
executeDraw();
```

This succeeds because

```
100 < 100

false
```

The contract now requests a random number.

### Step 2 — Randomness arrives

Chainlink (or the randomness provider) returns

```
Random Number
      │
      ▼
Winning Ticket
```

The contract stores the winning ticket internally.

For example,

```
winningTicket = 0x18A4
```

### Step 3 — Read the winning numbers

Since blockchain storage is public, Mallory simply reads

```solidity
lot.winningTicket(0)
```

She now knows the exact winning ticket.

### Step 4 — Buy the winning ticket

Because ticket registration is still open at timestamp 100,

Mallory submits

```solidity
buyTickets(...winningTicket...)
```

This succeeds because

```
100 > 100

false
```

Registration has not yet closed.

### Step 5 — Claim the jackpot

Since Mallory intentionally purchased the winning ticket after learning the result,

she simply calls

```solidity
claimWinningTickets(...)
```

and receives the jackpot.

> **Analogy:** Imagine a national lottery where ticket counters remain open for one extra minute after the winning numbers are announced on television. Anyone could simply watch the announcement, purchase the winning numbers, and legally claim the jackpot. The randomness wasn't broken—the ordering of events was.

## Vulnerable Code Reference

### 1) Draw execution opens too early

```solidity
function executeDraw() external override whenNotExecutingDraw {
    if (block.timestamp < drawScheduledAt(currentDraw)) {
        revert ExecutingDrawTooEarly();
    }

    requestRandomNumber();
}
```

**Problem**

At exactly

```
block.timestamp == drawScheduledAt(currentDraw)
```

the draw begins.

It should still revert until the ticket registration period has completely ended.

### 2) Ticket registration closes too late

```solidity
modifier beforeTicketRegistrationDeadline(uint128 drawId) {
    if (block.timestamp > ticketRegistrationDeadline(drawId)) {
        revert TicketRegistrationClosed(drawId);
    }
    _;
}
```

**Problem**

At exactly

```
block.timestamp == ticketRegistrationDeadline(drawId)
```

ticket purchases are still accepted.

This overlaps with draw execution.

## Recommended Mitigation

### 1. Close ticket registration exactly at the deadline

```solidity
if (block.timestamp >= ticketRegistrationDeadline(drawId)) {
    revert TicketRegistrationClosed(drawId);
}
```

This ensures no tickets can be purchased once the deadline is reached.

### 2. Delay draw execution until after the scheduled time

```solidity
if (block.timestamp <= drawScheduledAt(currentDraw)) {
    revert ExecutingDrawTooEarly();
}
```

This guarantees the draw cannot begin during the same timestamp in which ticket registration ends.

### 3. Maintain explicit phase separation

Rather than relying solely on timestamps, consider introducing explicit lottery states such as:

```
TicketSales
      │
      ▼
RegistrationClosed
      │
      ▼
RandomnessRequested
      │
      ▼
WinnerDetermined
```

State transitions are generally less error-prone than boundary timestamp comparisons.

### 4. Add boundary-condition tests

Unit tests should verify behavior at:

```
deadline - 1 second
deadline
deadline + 1 second
```

Likewise, draw execution should be tested at:

```
drawTime - 1
drawTime
drawTime + 1
```

Many off-by-one vulnerabilities are discovered only through these edge-case tests.

## Pattern Recognition Notes

* **Off-by-One Validation Errors:** Using `<` instead of `<=` (or `>` instead of `>=`) around timestamp boundaries can unintentionally leave a transition window open.

* **Broken Phase Separation:** Critical protocol phases (registration and draw execution) should never overlap. If two mutually exclusive operations become simultaneously valid, business logic vulnerabilities often emerge.

* **TOCTOU (Time-of-Check Time-of-Use):** The protocol checks whether ticket registration should be closed based on timestamps, but the timing overlap allows users to act after the draw has effectively begun.

* **Ordering Before Randomness:** In any lottery, raffle, auction, or commit-reveal protocol, all user inputs must be finalized before randomness is requested or revealed.

* **Boundary Testing:** Every timestamp comparison should be tested at exactly the boundary (`==`) as well as immediately before and after it. Most off-by-one bugs hide at these exact values.

## Quick Recall (TL;DR)

* **Bug:** Two off-by-one timestamp comparisons (`<` instead of `<=`, `>` instead of `>=`) allow ticket purchases and draw execution during the same timestamp.
* **Impact:** An attacker starts the draw, learns the winning ticket after randomness is fulfilled, purchases that exact ticket while registration is still open, and claims the jackpot.
* **Root Cause:** The protocol violates the invariant that **ticket sales must close before randomness is requested or revealed**.
* **Fix:** Replace the comparison operators with `<=` and `>=`, enforce strict phase separation, and thoroughly test timestamp boundary conditions.
