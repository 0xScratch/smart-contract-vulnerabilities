# Denial-of-Service via Flow-Limit Exhaustion in Axelar TokenManager

- **Severity**: Medium (Quality Assurance)
- **Source**: [Code4rena](https://github.com/code-423n4/2023-07-axelar-findings/issues/484) / [One Bug Per Day](https://www.onebugperday.com/v1/306)
- **Affected Contracts**: [TokenManager.sol](https://github.com/code-423n4/2023-07-axelar/blob/2f9b234bb8222d5fbe934beafede56bfb4522641/contracts/its/token-manager/TokenManager.sol) / [InterchainToken.sol](https://github.com/code-423n4/2023-07-axelar/blob/2f9b234bb8222d5fbe934beafede56bfb4522641/contracts/its/interchain-token/InterchainToken.sol)
- **Vulnerability Type**: Denial-of-Service / Quality Assurance as **determined by sponsors**

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-07-axelar/blob/2f9b234bb8222d5fbe934beafede56bfb4522641/contracts/its/token-manager/TokenManager.sol#L83-L173](https://github.com/code-423n4/2023-07-axelar/blob/2f9b234bb8222d5fbe934beafede56bfb4522641/contracts/its/token-manager/TokenManager.sol#L83-L173)
>[https://github.com/code-423n4/2023-07-axelar/blob/2f9b234bb8222d5fbe934beafede56bfb4522641/contracts/its/interchain-token/InterchainToken.sol#L1-L106](https://github.com/code-423n4/2023-07-axelar/blob/2f9b234bb8222d5fbe934beafede56bfb4522641/contracts/its/interchain-token/InterchainToken.sol#L1-L106)
>
>## Vulnerability details
>
>## Impact
>
>A large token holder can send back and forth tokens, using the flow limit to the capacity in start of every epoch making the system unusable for everyone else.
>
>## Proof of Concept
>
>Interchain tokens can be transferred from one chain to another via the token manager and interchain token service.
>
>And there is a limit imposed, for both the flow out and flow in.
>
>Flow out happens when we send the token from one chain to another. Lets say arbitrum to optimism and we are sending USDC. So in this case, in context of arbitrum it will be flow out and in context of optimism it will be flow in and and receiver on optimism will get the tokens via the token manager 'giveToken()' callable by the inter chain token service.
>
>But there is a flow limit impose per epoch.
>
>One Epoch = 6 hours long.
>
>So there cannot be more than certain amount of tokens sent between the chain per 6 hours. This is done to protect from the uncertain conditions like a security breach and to secure as much of tokens as possible.
>
>But the problem with such design is big token holder or whale could easily exploit it to DOS the other users.
>
>Consider the following scenerio:
>
>1. Epoch starts.
>2. Limit imposed for the flow is 10 million USDC (considering usdc to be interchain token for ease of understanding).
>3. A big whale transfer 10 million USDC in start of the epoch and those are there and may or may not receive them on other end right away.
>4. But the limit have been reached for the specific epoch. Now no other user can use the axelar interchain token service to transfer that particular token on the Dossed lane.
>5. Now attacker can repeat the process across multiple lanes on multiple chain or one, in start of every epoch making it unusable for every one with very minimum cost.
>
>This attack is pretty simple and easy to acheive and also very cheap to do, specifically on the L2's or other cheap chains due to low gas price.
>
>Function using the flow limit utility in `tokenManager.sol` are following
>
>```solidity
>    function sendToken(
>        string calldata destinationChain,
>        bytes calldata destinationAddress,
>        uint256 amount,
>        bytes calldata metadata
>    ) external payable virtual {
>        address sender = msg.sender;
>        amount = _takeToken(sender, amount);
>        _addFlowOut(amount);
>        interchainTokenService.transmitSendToken{ value: msg.value }(
>            _getTokenId(),
>            sender,
>            destinationChain,
>            destinationAddress,
>            amount,
>            metadata
>        );
>    }
>
>    /**
>     * @notice Calls the service to initiate the a cross-chain transfer with data after taking the appropriate amount of tokens from the user.
>     * @param destinationChain the name of the chain to send tokens to.
>     * @param destinationAddress the address of the user to send tokens to.
>     * @param amount the amount of tokens to take from msg.sender.
>     * @param data the data to pass to the destination contract.
>     */
>    function callContractWithInterchainToken(
>        string calldata destinationChain,
>        bytes calldata destinationAddress,
>        uint256 amount,
>        bytes calldata data
>    ) external payable virtual {
>        address sender = msg.sender;
>        amount = _takeToken(sender, amount);
>        _addFlowOut(amount);
>        uint32 version = 0;
>        interchainTokenService.transmitSendToken{ value: msg.value }(
>            _getTokenId(),
>            sender,
>            destinationChain,
>            destinationAddress,
>            amount,
>            abi.encodePacked(version, data)
>        );
>    }
>
>    /**
>     * @notice Calls the service to initiate the a cross-chain transfer after taking the appropriate amount of tokens from the user. This can only be called by the token itself.
>     * @param sender the address of the user paying for the cross chain transfer.
>     * @param destinationChain the name of the chain to send tokens to.
>     * @param destinationAddress the address of the user to send tokens to.
>     * @param amount the amount of tokens to take from msg.sender.
>     */
>    function transmitInterchainTransfer(
>        address sender,
>        string calldata destinationChain,
>        bytes calldata destinationAddress,
>        uint256 amount,
>        bytes calldata metadata
>    ) external payable virtual onlyToken {
>        amount = _takeToken(sender, amount);
>        _addFlowOut(amount);
>        interchainTokenService.transmitSendToken{ value: msg.value }(
>            _getTokenId(),
>            sender,
>            destinationChain,
>            destinationAddress,
>            amount,
>            metadata
>        );
>    }
>
>    /**
>     * @notice This function gives token to a specified address. Can only be called by the service.
>     * @param destinationAddress the address to give tokens to.
>     * @param amount the amount of token to give.
>     * @return the amount of token actually given, which will onle be differen than `amount` in cases where the token takes some on-transfer fee.
>     */
>    function giveToken(address destinationAddress, uint256 amount) external onlyService returns (uint256) {
>        amount = _giveToken(destinationAddress, amount);
>        _addFlowIn(amount);
>        return amount;
>    }
>
>    /**
>     * @notice This function sets the flow limit for this TokenManager. Can only be called by the operator.
>     * @param flowLimit the maximum difference between the tokens flowing in and/or out at any given interval of time (6h)
>     */
>    function setFlowLimit(uint256 flowLimit) external onlyOperator {
>        _setFlowLimit(flowLimit);
>    }
>```
>
>## Tools Used
>
>Manual review
>
>## Recommended Mitigation Steps
>
>There could be many solution for this one. But two solutions from top of my head are:
>
>1. Do the chainlink way CCIP way, chainlink recently launched cross chain service solved the similar problem by imposing the token bps fee, by imposing such fee along with gas fee, cost of attack becomes way higher and system can be protected from such attack.
>2. Introduce the mechanism of limit per account, instead of whole limit. But that can be exploited too by doing it through multiple accounts.
>
>Chainlink's way would be the better solution to go with IMO.
>
>## Assessed type
>
>DoS

## Summary

Axelar's TokenManager enforces a **global flow limit** that caps the total amount of tokens that can be transferred cross-chain in or out during a fixed time period ("epoch") of 6 hours. This is intended as a safety feature to prevent large, rapid, or potentially malicious token movements that could harm the system or users.

However, this design introduces a risk where a **single large token holder ("whale") can repeatedly consume the entire transfer capacity at the very start of each epoch**. By sending close to or the entire flow limit amount cross-chain right when the epoch resets, the whale fills the "transfer bucket" and **temporarily blocks all other users from transferring tokens on that chain for that epoch duration** (up to 6 hours). This creates a Denial-of-Service (DoS) against legitimate users.

## Detailed Explanation

### What Are Flow Out and Flow In?

- **Flow Out:**
When tokens are moved _from_ a source chain _to_ another chain, the TokenManager on the source chain records this as Flow Out. It increments an internal counter tracking how many tokens have left the chain during the current epoch.
- **Flow In:**
Conversely, when those tokens arrive _on the destination chain_, the TokenManager there records a Flow In of the same amount.
- **Epoch:**
The TokenManager tracks Flow In and Out in 6-hour rolling windows called epochs to limit token movement within each interval.
- **Flow Limit:**
Operators set a maximum allowed difference (usually a numeric token amount) between Flow In and Flow Out per token manager during each epoch. If a transfer would exceed this limit, the transaction reverts.

### How Are the Flow Limits Enforced?

- Each time a user initiates a cross-chain token transfer from chain A (e.g., by calling `sendToken()`), the TokenManager on chain A calls `_addFlowOut(amount)` to increase the Flow Out counter.
- When tokens arrive on chain B, the TokenManager there calls `_addFlowIn(amount)` to increase Flow In.
- The flow limit is enforced by ensuring the net transfer does not exceed the allowed threshold in either direction over the current epoch.

### The Attack Scenario

1. **Epoch begins on Chain A; flow limit = 10 million tokens**
2. Large whale sends 10 million tokens immediately via `sendToken(...)` on chain A:
    - `_addFlowOut(10 million)` sets the Flow Out counter to max limit.
3. Since the flow limit is now fully consumed, **any subsequent token transfer attempts by other users revert** because the contract enforces that transfers cannot exceed this limit.
4. Honest users are **blocked from transferring any tokens across chains on Chain A until the epoch resets**.
5. The whale repeats sending the full limit at each new epoch start, **causing persistent denial of cross-chain token transfers for others**.

Meanwhile, tokens may or may not have arrived on the destination chain yet, and the system doesn't distinguish whether the Flow In has caught up to this Flow Out.

### Why Does This Matter?

- **Temporary but impactful denial:** Legitimate users cannot move tokens across chains, hampering usability and degrading user experience.
- **Low-cost exploit:** Whale only pays gas fees and token amounts, which is affordable on low-gas or Layer 2 chains, making the attack trivial to execute repeatedly.
- **No direct loss of funds:** Assets remain safe but the protocol functionality is degraded.
- **Repeated across multiple chains or tokens:** The whale can multiply the impact by targeting multiple bridges or tokens.

## Vulnerable Code References

```solidity
function sendToken(...) public payable {
    amount = _takeToken(sender, amount);
    _addFlowOut(amount);  // Increase flow out counter globally
    interchainTokenService.transmitSendToken{ value: msg.value }(...);
}

function callContractWithInterchainToken(...) public payable {
    amount = _takeToken(sender, amount);
    _addFlowOut(amount);
    interchainTokenService.transmitSendToken{ value: msg.value }(...);
}

function transmitInterchainTransfer(...) external payable onlyToken {
    amount = _takeToken(sender, amount);
    _addFlowOut(amount);
    interchainTokenService.transmitSendToken{ value: msg.value }(...);
}

function giveToken(...) external onlyService returns (uint256) {
    amount = _giveToken(destinationAddress, amount);
    _addFlowIn(amount);  // Increase flow in counter on destination chain
    return amount;
}
```

## Recommended Mitigation \& Protocol Context

### Understanding the Design Choice

- The **flow limit mechanism is an intentional safety feature** to prevent rapid large-value token movements that could risk theft or bugs.
- Axelar operators **have control to adjust or disable the flow limit at any time** by calling `setFlowLimit()` — it can be increased or set to zero (no limit) if needed to respond to demand spikes or abuse.
- Sponsors of the protocol explicitly prefer **not to introduce additional fees (like basis-point fees)** on cross-chain transfers, as they want to keep user costs minimal.
- Per-account limits or throttling were considered but rejected due to ineffectiveness against Sybil-style circumvention and complexity.

### Suggestions for Improvement

Given the sponsors' stance and the protocol's design goals, here are potential mitigation ideas that align with their philosophy:

1. **Operator Proactive Monitoring \& Dynamic FlowLimits**
    - Operators should monitor usage patterns and suspicious large transfers.
    - They can dynamically increase or remove flow limits temporarily during high legitimate demand or suspected attacks.
    - Automated alerting or governance mechanisms can help ensure rapid operator response.
2. **Shorter Epochs or Sliding Window**
    - Instead of rigid 6-hour epochs, implement sliding windows or smaller intervals to minimize the impact time a whale can hold the system hostage.
    - This reduces the window during which no other transfers can occur.
3. **Anti-Whale Policies at the Protocol or dApp Level**
    - Encourage or enforce limits on maximum transfer sizes at the user or dApp level.
    - This doesn't solve systemic risk fully but reduces risk of large single transfers exhausting flow.
4. **Introduce Economic Disincentives Only if Absolutely Necessary**
    - While sponsors prefer feeless transfers, a minimal fee in exceptional cases could be a deterrent if abuse becomes problematic.

## Severity \& Real-World Implications

- This issue is categorized primarily as a **Quality Assurance (QA) / Medium severity** because:
  - It does **not cause asset theft or loss**.
  - It **can impact availability and user experience** temporarily.
  - It **is a known and accepted trade-off** by the protocol developers and operators.
- Operators hold the responsibility to configure flow limits balancing security against availability.
- The protocol provides **flexibility, but requires vigilant operational management** to prevent prolonged denial-of-service situations.

## Pattern Recognition Notes

- **Global Limits Without Per-User Controls**
  - Shared limits across all users can be abused by a single actor, leading to denial-of-service.

- **Coarse Time Windows for Rate Limiting**
  - Large fixed epochs enable attackers to monopolize resources until reset; finer or sliding windows reduce impact.

- **Lack of Economic Deterrents**
  - Absence of fees or slashing makes repeated abuse cheap and easy.

- **Unilateral Control Over Limits**
  - Single-party control over critical parameters can lead to instability or exploitation if misused.

- **Race Conditions in Multi-Step Processes**
  - Asynchronous updates with separate checks and state changes open attack windows.

- **Dependence on Manual Recovery**
  - Relying on operator action to fix or adjust parameters prolongs downtime during attacks.

- **Sybil Resistance Concerns**
  - Per-account limits risk circumvention without robust identity or Sybil defenses.

- **Insufficient Monitoring & Alerts**
  - Lack of observability hampers timely detection and response to abuses.
