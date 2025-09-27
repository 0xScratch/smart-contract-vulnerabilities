# Signature Replay in Split-Voting Governor Elections

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2023-08-arbitrum-findings/issues/252) / [One Bug Per Day](https://www.onebugperday.com/v1/448)
* **Affected Contracts**: [GovernorUpgradeable.sol](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol) (Ain't afffected, but the reason behind this vulnerability)
* **Vulnerability Type**: Signature Replay / Missing Nonce / Authorization Bypass

## Summary

Arbitrum's Security Council election contracts extend OpenZeppelin's `GovernorUpgradeable` to support **split voting**, where delegates can distribute their voting power across multiple candidates.

The function `castVoteWithReasonAndParamsBySig()` lets users authorize votes via off-chain signatures. However, the signature data **does not include a nonce or replay protection**.

This allows an attacker to **reuse the same signature multiple times**, forcing a delegate's entire voting power onto a single candidate, overriding the delegate's intended allocation.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. Delegate Bob has **1000 votes**.
2. Bob signs:

   * Signature A → 500 votes for Alice
   * Signature B → 500 votes for Charlie
3. Relayers submit both signatures.

   * Alice gets 500
   * Charlie gets 500
   * Total = 1000, as intended.

### What Actually Happens (Bug)

1. Bob signs Signature A (500 → Alice).
2. Relayer submits Signature A once → Alice gets 500.
3. Attacker sees Signature A on-chain and **replays it** → Alice gets **another 500**.
4. Now Alice has all 1000 votes.
5. When Signature B (for Charlie) is submitted, it reverts (Bob has no remaining votes).

👉 Bob's intent (split votes) is overridden. An attacker can redirect all votes to a single candidate.

### Why This Matters

* Delegates **lose control** of how their votes are split.
* Attackers can manipulate election outcomes by replaying signatures.
* Especially critical for **Security Council governance**, where a few thousand votes can decide key protocol upgrades.

> **Analogy**: Think of a signed blank cheque with no serial number. If you give one to pay $500, anyone who sees it can photocopy it and cash it multiple times until your account is drained.

## Vulnerable Code Reference

### 1) Signature Recovery (missing nonce)

```solidity
address voter = ECDSAUpgradeable.recover(
    _hashTypedDataV4(
        keccak256(
            abi.encode(
                EXTENDED_BALLOT_TYPEHASH,
                proposalId,
                support,
                keccak256(bytes(reason)),
                keccak256(params)
            )
        )
    ),
    v,
    r,
    s
);
```

* The signed payload = `(proposalId, support, reason, params)`
* **No nonce / unique marker** → replayable.

### 2) Vote Casting

```solidity
return _castVote(proposalId, voter, support, reason, params);
```

* Accepts same signature repeatedly.
* Each call consumes more votes until exhausted.

## Recommended Mitigation

1. **Nonce in Signatures**

   * Extend signature domain to include `nonce`.
   * Track `usedNonces[voter]` and increment after use.

   ```solidity
   bytes32 structHash = keccak256(
       abi.encode(
           EXTENDED_BALLOT_TYPEHASH,
           proposalId,
           support,
           keccak256(bytes(reason)),
           keccak256(params),
           nonces[voter]++  // replay protection
       )
   );
   ```

2. **Replay Tracking**

   * Track `usedSignature[hash] = true`.
   * Reject any reused signature hash.

3. **Election-Specific Override**

   * Override `castVoteWithReasonAndParamsBySig()` in Security Council governors.
   * Ensure each vote authorization can only be executed **once**.

## Pattern Recognition Notes

* **Missing Nonce in Off-Chain Signatures**: Replay attacks are a classic pitfall when using `ecrecover`. Compare with ERC-20 `permit()`, which always includes a nonce.
* **Extension Mismatch**: OpenZeppelin Governor assumes **1 vote per address per proposal** → natural replay protection. Arbitrum's **split voting** breaks this assumption, making nonce omission exploitable.
* **Authorization Reuse**: Any contract where off-chain authorization leads to **repeated state change** must include a **nonce, deadline, or hash registry**.
* **Critical Governance Context**: Bugs in voting mechanisms amplify risk since they affect **system-level decisions** rather than just user balances.

### Quick Recall (TL;DR)

* **Bug**: Signatures for votes lack nonce → replayable.
* **Impact**: Attackers can force a delegate's entire vote onto one candidate.
* **Fix**: Add nonce or signature replay protection (as in ERC-20 permit).
