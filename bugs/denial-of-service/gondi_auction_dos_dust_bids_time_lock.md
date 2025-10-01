# Auction DoS via Minimal Increment Bids in Gondi

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-04-gondi-findings/issues/37)
* **Affected Contract**: [AuctionLoanLiquidator.sol](https://github.com/code-423n4/2024-04-gondi/blob/b9863d73c08fcdd2337dc80a8b5e0917e18b036c/src/lib/AuctionLoanLiquidator.sol)
* **Vulnerability Type**: Denial of Service (DoS) / Auction Manipulation

## Summary

The `placeBid` function in Gondi auctions enforces a **5% increment rule** relative to the last bid. However, when the **initial bid is extremely small**, subsequent increments remain negligible (e.g., 100 → 105 → 110 wei).

Because each bid also enforces a **no-action margin** (`_MIN_NO_ACTION_MARGIN`), an attacker can repeatedly place tiny bids that **temporarily lock other participants out**. By chaining such increments, the attacker effectively monopolizes the auction, preventing fair participation and acquiring the NFT at a very low price.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Auction Start**: Bids must always increase the current highest bid by at least 5%.
2. **No-Action Margin**: Each bid adds a time buffer (`lastBidTime + margin`) during which no further bids are allowed.
3. **Competition**: Users are expected to outbid each other until the auction ends near fair market value.

### What Actually Happens (Bug)

* If the attacker starts at a **tiny bid** (e.g., `100 wei`), then:

  * The next bid requirement is just `105 wei`.
  * Then `110 wei`.
  * Then `115 wei`, and so on.
* Each increment is insignificant, so the **attacker's total cost remains trivial**.
* Because of `_MIN_NO_ACTION_MARGIN`, **other bidders cannot place a competing bid until the attacker's margin passes**.
* By timing their next bid at the end of each margin window, the attacker can **lock everyone else out** and keep raising the bid in tiny steps.

### Why This Matters

* The auction becomes **unfair and non-competitive**.
* An NFT worth multiple ETH can be won for just a few wei increments.
* DoS on competition is as damaging as contract freeze: the system *functions* but only for the attacker's benefit.
* This undermines protocol credibility and directly harms sellers and lenders relying on liquidation auctions.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: An NFT worth ~1 ETH is liquidated via auction.
* **Mallory attack**:

  * Places initial bid: `100 wei`.
  * Margin timer starts → other users locked out.
* **Second round**:

  * Before margin expires, Alice wants to bid 0.5 ETH.
  * She can't → transaction reverts (`AuctionOverError` or lock not expired).
  * Mallory instead places a new bid: `105 wei`.
* **Repeats**:

  * Mallory continues 110 wei → 115 wei → …
  * Each bid cheap, each locks out competitors.
* **Auction end**:

  * No one else could meaningfully bid.
  * Mallory acquires NFT for ~trivial cost.

> **Analogy**: Imagine a public auction where each bid adds a 1-minute pause. One bidder raises by $0.01 every time. Everyone else is forced to wait and never gets a turn before the auction closes.

## Vulnerable Code Reference

### 1) 5% Increment Check

```solidity
uint256 currentHighestBid = _auction.highestBid;
// MIN_INCREMENT_BPS = 10000, _BPS = 500 → +5%
if (_bid == 0 || (currentHighestBid.mulDivDown(_BPS + MIN_INCREMENT_BPS, _BPS) >= _bid)) {
    revert MinBidError(_bid);
}
```

### 2) Time Lock Enforcement

```solidity
uint96 withMargin = _auction.lastBidTime + _MIN_NO_ACTION_MARGIN;
uint96 max = withMargin > expiration ? withMargin : expiration;
if (max < currentTime && currentHighestBid > 0) {
    revert AuctionOverError(max);
}
```

*Problem*: Small starting bids keep increments negligible, and `_MIN_NO_ACTION_MARGIN` locks competitors out after each tiny step.

## Recommended Mitigation

1. **Introduce a Minimum Bid Threshold**
   Prevent auctions from starting at dust values:

   ```solidity
   require(_bid > MIN_BID, "Bid too low");
   ```

   Where `MIN_BID` is protocol-defined (e.g., 0.01 ETH).

2. **Consider Absolute + Relative Increments**
   Require bids to increase by *both*:

   * A percentage (5%) **and**
   * A minimum absolute step (e.g., 0.001 ETH).

3. **Adjust Locking Mechanism**
   Instead of a hard no-action lock, use extensions that allow others to still bid but extend the end time if close to expiration.

## Pattern Recognition Notes

* **Relative-only Validation**: Percentage checks without absolute floors create exploitable "dust" ranges.
* **DoS via Time Locks**: Systems with enforced waiting periods are vulnerable if one actor can cheaply re-trigger the lock.
* **Auction Fairness Principles**: Always combine percentage increments with absolute minimum increments to prevent manipulation.
* **Anti-Sniping vs. Anti-Spam Trade-off**: Mechanisms designed to stop sniping (time buffers) can be inverted into tools for spam/DoS if not bounded.

### Quick Recall (TL;DR)

* **Bug**: Bids can start at tiny amounts → increments remain negligible.
* **Impact**: Attacker monopolizes bids using time locks, wins NFT at very low cost.
* **Fix**: Enforce minimum starting bid + combine absolute + relative increments + review lock logic.
