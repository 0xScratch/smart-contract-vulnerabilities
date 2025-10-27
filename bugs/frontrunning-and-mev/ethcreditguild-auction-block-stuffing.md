# Auction manipulation by block stuffing and reverting on ERC-777 hooks

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2023-12-ethereumcreditguild-findings/issues/685) / [One Bug Per Day](https://www.onebugperday.com/v1/774)
- **Affected Contracts**: [AuctionHouse.sol](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/AuctionHouse.sol#L118-L196) & [GIP_0.sol](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/test/proposals/gips/GIP_0.sol#L175-L179)
- **Vulnerability Type**: MEV / Block Stuffing / Economic Manipulation

## Original Bug Description

> **Auction manipulation by block stuffing and reverting on ERC-777 hooks**
>
> **Summary**
>
> The protocol stated out in the C4 description that the deployment script of the protocol, located in test/proposals/gips/GIP_0.sol is also in scope, as protocol deployment/configuration mistakes could be made. A low immutable auction duration set in this deployment script can lead to profitable block stuffing attacks on the desired L2 chains. This attack vector can be further improved under the condition that the collateral token is ERC-777 compatible.
>
> **Vulnerability Details**
>
> The auction house contract is deployed with the following parameters (auctionDuration and midPoint are immutables):
>
> ```solidity
> AuctionHouse auctionHouse = new AuctionHouse(
>   AddressLib.get("CORE"),
>   650, // midPoint = 10m50s
>   1800 // auctionDuration = 30m
> );
> ```
>
> During the first half of the auction (before midPoint), an increasing amount of the collateral is offered, for the full CREDIT amount.
>
> During the second half of the action (after midPoint), all collateral is offered, for a decreasing CREDIT amount.
>
> The calculation can be seen in the getBidDetail function:
>
> ```solidity
> uint256 public immutable midPoint;
> uint256 public immutable auctionDuration;
>
> function getBidDetail(bytes32 loanId) public view returns (uint256 collateralReceived, uint256 creditAsked) {
>   // check the auction for this loan exists
>   uint256 _startTime = auctions[loanId].startTime;
>  require(_startTime != 0, 'AuctionHouse: invalid auction');
>
>  // check the auction for this loan isn't ended
>  require(auctions[loanId].endTime == 0, 'AuctionHouse: auction ended');
>
>  // assertion should never fail because when an auction is created,
>  // block.timestamp is recorded as the auction start time, and we check in previous
>  // lines that start time != 0, so the auction has started.
>  assert(block.timestamp >= _startTime);
>
>  // first phase of the auction, where more and more collateral is offered
>  if (block.timestamp < _startTime + midPoint) {
>    // ask for the full debt
>    creditAsked = auctions[loanId].callDebt;
>
>    // compute amount of collateral received
>    uint256 elapsed = block.timestamp - _startTime; // [0, midPoint[
>    uint256 _collateralAmount = auctions[loanId].collateralAmount; // SLOAD
>    collateralReceived = (_collateralAmount * elapsed) / midPoint;
>  }
>  // second phase of the auction, where less and less CREDIT is asked
>  else if (block.timestamp < _startTime + auctionDuration) {
>    // receive the full collateral
>    collateralReceived = auctions[loanId].collateralAmount;
>
>    // compute amount of CREDIT to ask
>    uint256 PHASE_2_DURATION = auctionDuration - midPoint;
>    uint256 elapsed = block.timestamp - _startTime - midPoint; // [0, PHASE_2_DURATION[
>    uint256 _callDebt = auctions[loanId].callDebt; // SLOAD
>    creditAsked = _callDebt - (_callDebt * elapsed) / PHASE_2_DURATION;
>  }
>  // second phase fully elapsed, anyone can receive the full collateral and give 0 CREDIT
>  // in practice, somebody should have taken the arb before we reach this condition.
>  else {
>    // receive the full collateral
>    collateralReceived = auctions[loanId].collateralAmount;
>    //creditAsked = 0; // implicit
>  }
>}
>```
>
> This means that as longer the auction goes till a bid is made (which instantly buys the auction), the more profit can be made by executing the auction.
>
> The following conditions allow an attacker to manipulate auctions by stuffing blocks to increase profits:
>
> - auctionDuration is set to 1800 seconds (30min) in the deployment script
> - auction midPoint is set to 650 seconds (10m50s) in the deployment script
> - the protocol will be deployed on L2s let's take optimism for example (as this chain was mentioned by the devs in discord):
>   - blocks in optimism are minted every 2s
>   - the block gas limit of optimism is 30M as in ethereum mainnet
>   - at the time of writting this the cost of stuffing a full block on optimism is around $6.39
>   - therefore prevent a bid for one minute costs $6.39 * 30 blocks = $191.7
>   - therefore prevent a bid for 30 minutes costs $191.7 * 30 minutes = $5751
>   - but the attacker will almost never need to stuff blocks for around 30 minutes as the system of the auction is not profitable in the beginning and it starts to be really profitable after the midpoint, and it also starts to be much more damaging to the system at this point. Which is after 10m50s and the costs to stuff blocks for 10m50s is $191.7 * 10.5 minutes = $2012.85 and as mentioned before the auction is not profitable at the beginning, therefore no one will bid on second one and the attacker does not need to stuff blocks instantly. This means the attack will most likely cost less than $2000.
> - therefore if at any given timestamp the profit of the auction outweighs the cost of stuffing blocks for the time till the deal becomes profitable for the attacker a system damaging incentive appears and manipulating auctions becomes a profitable attack vector.
>
> But the impact increases further in terms of griefing as loss for terms can occur after the midPoint which will instantly lead to slashing and therefore all stakers of the given term will lose all their credit tokens weighted on this term.
>
> The following code snippets showcase the slashing mechanism that lead to a total loss for the stakers if the term receives any loss during these block stuffing attack:
>
> ```solidity
>function bid(bytes32 loanId) external {
>    ...
>
>    LendingTerm(_lendingTerm).onBid(
>        loanId,
>        msg.sender,
>        auctions[loanId].collateralAmount - collateralReceived, // collateralToBorrower
>        collateralReceived, // collateralToBidder
>        creditAsked // creditFromBidder
>    );
>
>    ...
>}
>
>function onBid(
>    bytes32 loanId,
>    address bidder,
>    uint256 collateralToBorrower,
>    uint256 collateralToBidder,
>    uint256 creditFromBidder
>) external {
>    ...
>
>    int256 pnl;
>    uint256 interest;
>    if (creditFromBidder >= principal) {
>        interest = creditFromBidder - principal;
>        pnl = int256(interest);
>    } else {
>        pnl = int256(creditFromBidder) - int256(principal);
>        principal = creditFromBidder;
>        require(
>            collateralToBorrower == 0,
>            "LendingTerm: invalid collateral movement"
>        );
>    }
>
>    ...
>
>    // handle profit & losses
>    if (pnl != 0) {
>        // forward profit, if any
>        if (interest != 0) {
>            CreditToken(refs.creditToken).transfer(
>                refs.profitManager,
>                interest
>            );
>        }
>        ProfitManager(refs.profitManager).notifyPnL(address(this), pnl);
>    }
>
>    ...
>}
>
>function notifyPnL(
>    address gauge,
>    int256 amount
>) external onlyCoreRole(CoreRoles.GAUGE_PNL_NOTIFIER) {
>    ...
>
>    // handling loss
>    if (amount < 0) {
>        uint256 loss = uint256(-amount);
>
>        // save gauge loss
>        GuildToken(guild).notifyGaugeLoss(gauge);
>
>        // deplete the term surplus buffer, if any, and
>        // donate its content to the general surplus buffer
>        if (_termSurplusBuffer != 0) {
>            termSurplusBuffer[gauge] = 0;
>            emit TermSurplusBufferUpdate(block.timestamp, gauge, 0);
>            _surplusBuffer += _termSurplusBuffer;
>        }
>
>        if (loss < _surplusBuffer) {
>            // deplete the surplus buffer
>            surplusBuffer = _surplusBuffer - loss;
>            emit SurplusBufferUpdate(
>                block.timestamp,
>                _surplusBuffer - loss
>            );
>            CreditToken(_credit).burn(loss);
>        }
>    } ...
>}
>
>function notifyGaugeLoss(address gauge) external {
>    require(msg.sender == profitManager, "UNAUTHORIZED");
>
>    // save gauge loss
>    lastGaugeLoss[gauge] = block.timestamp;
>    emit GaugeLoss(gauge, block.timestamp);
>}
>
>/// @notice apply a loss that occurred in a given gauge
>/// anyone can apply the loss on behalf of anyone else
>function applyGaugeLoss(address gauge, address who) external {
>    // check preconditions
>    uint256 _lastGaugeLoss = lastGaugeLoss[gauge];
>    uint256 _lastGaugeLossApplied = lastGaugeLossApplied[gauge][who];
>    require(
>        _lastGaugeLoss != 0 && _lastGaugeLossApplied < _lastGaugeLoss,
>        "GuildToken: no loss to apply"
>    );
>
>    // read user weight allocated to the lossy gauge
>    uint256 _userGaugeWeight = getUserGaugeWeight[who][gauge];
>
>    // remove gauge weight allocation
>    lastGaugeLossApplied[gauge][who] = block.timestamp;
>    _decrementGaugeWeight(who, gauge, _userGaugeWeight);
>    if (!_deprecatedGauges.contains(gauge)) {
>        totalTypeWeight[gaugeType[gauge]] -= _userGaugeWeight;
>        totalWeight -= _userGaugeWeight;
>    }
>
>    // apply loss
>    _burn(who, uint256(_userGaugeWeight));
>    emit GaugeLossApply(
>        gauge,
>        who,
>        uint256(_userGaugeWeight),
>        block.timestamp
>    );
>}
>```
>
> This attack vector can be further improved under the condition that the collateral token is ERC-777 compatible. It is advised to first read the report called `Bad debt can occur if the collateral token blacklists a borrower leading to total loss of stake for all lenders on that term` which showcases how the auction time is increased till the midPoint of the auction if transferring the collateral tokens to the borrower reverts.
>
> The attack path would be as follows:
>
> - attacker borrows a loan (with a ERC-777 compatible collateral token) using a contract that revert on receiving the collateral back if the tx.origin != address(attacker)
> - the attacker receives the credit token from the loan and deposits collateral into the protocol
> - the attacker does not repay the loan after the first partial duration and call it for auction
> - as showcased in the mentioned report every call till the midPoint will revert as trying to transfer the collateral back to the attacker will revert (this will drastically decrease the costs for the attacker as he does not need to stuff blocks till the midPoint is reached)
> - after the midPoint is reached the attacker can start stuffing blocks till bad debt occurs, at this time the attacker can bid on the auction and will therefore buy the full collateral back for less credit tokens than the attacker received for the loan
> - if it is possible for the attacker to stuff blocks and wait till the asked credit amount goes to 0 the attacker was able to steal almost the full loan amount (which can potentially be way bigger than the gas amount for stuffing the blocks as the calculation in the beginning showed)
>
> **Impact**
>
> The attacker can prevent other users from bidding on the auction and therefore manipulate the auction to a point where the attacker would be able to buy the full collateral for almost zero credit tokens. As loss for the term occurs in such an event, all stakers of the given term will lose all their credit tokens weighted on this term. If the given collateral token is ERC-777 compatible, the costs of such an attack can be drastically reduced. And the attack can potentially become a self liquidation attack.
>
> **Recommendations**
>
> Increase the auction duration, as the longer the auction goes the less profitable such an attack would be and implement the mentioned fix in the `Bad debt can occur if the collateral token blacklists a borrower leading to total loss of stake for all lenders on that term` report.

## Real-World Context

This vulnerability is particularly dangerous for:

- **Layer 2 deployments** (Optimism, Arbitrum) where block stuffing costs are lower
- **High-value collateral auctions** where attack costs are justified by potential profits
- **ERC-777 compatible collateral tokens** that enable hook-based bid blocking
- **Lending protocols** using time-based Dutch auctions without censorship resistance

## Key Details

- **Pattern**: Manipulation through auction timing + ERC-777 revert hooks + economic censorship (block stuffing).
- **Root Cause**:
  - Auction logic uses a **push pattern** to send collateral back to borrowers during bidding.
  - Malicious borrowers can use ERC-777 `tokensReceived()` hooks to **revert** those transfers, DoS'ing all bids from others.
  - Short, predictable auction durations (30m total, 10m50s midpoint) make **block stuffing economically viable** on L2s.
- **Cost-Profit Window**:
  - Stuffing blocks post-midpoint (~$2,000 cost) buys attacker exclusive bidding access as **credit asked drops fast**.
  - Attacker acquires full collateral at a discount, causing **protocol loss** and staker slashing.
- **Push vs Pull Note**:
  - This issue is avoidable if collateral transfers are moved to a **pull-based design**, avoiding mid-auction token pushes.

## Push vs Pull Patterns in Protocol Design

### The Push Pattern (and Why It's Dangerous Here)

- In the **push pattern**, the protocol immediately sends (pushes) tokens—like excess collateral—directly to users/borrowers as part of a state-altering process (e.g., during an auction bid).
- With ERC-777 tokens, the receiver can include custom logic in tokensReceived, allowing the borrower (or any contract recipient) to **revert** the entire transaction based on arbitrary criteria.
- In this bug, a malicious borrower can use a contract that always reverts if they're not the one triggering the receive (e.g., `if (tx.origin != attacker) revert()`). Because the protocol tries to push collateral to the borrower during the auction (before midpoint), any legitimate bid that isn't from the attacker fails—the process is "DoS'd" by the borrower's contract.

### The Pull Pattern (The Safer Alternative)

- The **pull pattern** records (in contract state) the tokens a user is owed, but **requires the user to pull/claim** them in a separate, independent transaction.
- The protocol simply updates an internal balance (`collateralOwed[loanId] += amount`) instead of transferring tokens in-line.
- Only when a user (borrower) calls a function like `withdrawCollateral(loanId)`—decoupled from the critical protocol logic—does the transfer occur. If their contract reverts or misbehaves, **it affects only their withdrawal, not the main protocol action.**

### Example Pull Pattern Code

```solidity
function withdrawCollateral(bytes32 loanId) external {
    require(msg.sender == loans[loanId].borrower, "Not borrower");
    uint256 amount = collateralOwed[loanId];
    collateralOwed[loanId] = 0;
    IERC20(collateralToken).safeTransfer(msg.sender, amount);
}
```

### Why This Change Matters

- **Attack Prevention**: Core mechanics (auction bids, liquidations, repayments) can't be blocked by custom logic in user contracts or token hooks.
- **Robust Protocol Flows**: Eliminates dependency on external contract behavior for process to complete.
- **User Safety**: If a borrower's contract malfunctions, only their own withdrawal is impacted—they can't block system state or auction success for others.
- **Industry-Standard Defense**: Using a pull pattern for user payouts/returns is standard in safe DeFi, especially for any external or callback-enabled token.

## Vulnerable Code Reference

### Deployment Configuration (GIP_0.sol)

```solidity
AuctionHouse auctionHouse = new AuctionHouse(
    AddressLib.get("CORE"),
    650, // midPoint = 10m50s - too short for L2 security
    1800 // auctionDuration = 30m - insufficient for block stuffing resistance
);
```

### Auction Logic (AuctionHouse.sol)

```solidity
function getBidDetail(bytes32 loanId) public view returns (uint256 collateralReceived, uint256 creditAsked) {
    // Second phase: all collateral offered for decreasing debt
    if (block.timestamp < _startTime + auctionDuration) {
        collateralReceived = auctions[loanId].collateralAmount;
        uint256 elapsed = block.timestamp - _startTime - midPoint;
        creditAsked = _callDebt - (_callDebt * elapsed) / PHASE_2_DURATION; // Exploitable price decrease
    }
}
```

### ERC-777 Hook Vulnerability

```solidity
function onBid(...) external {
    // Vulnerable: collateral transfer can be blocked by ERC-777 hooks
    if (collateralToBorrower != 0) {
        IERC20(params.collateralToken).safeTransfer(
            loans[loanId].borrower, 
            collateralToBorrower
        );
    }
}
```

## Impact

**Primary Impact**:

- Attackers can acquire valuable collateral for drastically reduced debt payments
- Protocol suffers bad debt that triggers staker slashing mechanism
- All stakers backing the affected lending term lose their tokens proportionally

**Secondary Impact**:

- Economic griefing of honest participants and protocol
- Undermines auction price discovery and liquidation efficiency
- Creates systemic risk if multiple high-value auctions are targeted

## Mitigation

### Immediate Fixes

1. **Increase auction duration** to 2+ hours to make block stuffing economically unviable
2. **Implement pull pattern** for collateral transfers to prevent ERC-777 DoS:

    ```solidity
    // Instead of pushing collateral to borrower during auction
    mapping(bytes32 => uint256) public borrowerCollateralOwed;

    function withdrawCollateral(bytes32 loanId) external {
        require(msg.sender == loans[loanId].borrower);
        uint256 amount = borrowerCollateralOwed[loanId];
        borrowerCollateralOwed[loanId] = 0;
        IERC20(collateralToken).safeTransfer(msg.sender, amount);
    }
    ```

### Long-term Solutions

1. **Randomized auction timing** to prevent predictable attack windows
2. **Commit-reveal bidding schemes** to hide bidding intentions
3. **Multiple auction phases** with different mechanisms
4. **Economic penalties** for failed bids to discourage manipulation

## Pattern Recognition Notes

- **Watch for**:
  - Short, fixed auction durations on L2s (cheap to censor).
  - Push-based token transfers to borrowers inside auctions or liquidations.
- **Red Flags**:
  - Use of `.transfer()` / `.safeTransfer()` to users during auction actions.
  - Lack of ERC-777 awareness in handling collateral.
- **Common Pitfalls**:
  - Returning leftover collateral mid-bid, especially before auctions complete.
  - No allowlist for safe collateral tokens (opens up exotic standards like ERC-777).
- **Testing Tip**:
  - Inject ERC-777 tokens with revert-on-hook logic (`tokensReceived()`), simulate bids from non-attacker, and expected reverts.
- **Push vs Pull Warning**:
  - If smart contract behavior affects core flows (like auctions), you need **pull, not push**—to isolate effects locally.
