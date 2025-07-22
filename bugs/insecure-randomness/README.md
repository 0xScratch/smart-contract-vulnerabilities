# Insecure Randomness

## Major Real-World Incidents Involving Similar Vulnerabilities

### Larva Labs Meebits Exploit (2021)

- **Loss**: Attacker obtained rare Meebit #16647 worth ~$750,000 after spending ~$20,000/hour in gas fees for 6+ hours [[1]](https://news.bit2me.com/en/meebits-exploit-to-choose-to-cradle-nft-at-will)
- **Method**: Contract used predictable randomness based on `keccak256(abi.encodePacked(nonce, msg.sender, block.difficulty, block.timestamp))`. Attacker deployed contract with `onERC721Received` hook that reverted unless rare traits were obtained, then retried 300+ times [[2]](https://blog.mycrypto.com/nft-smart-contract-bugs-exploits) [[3]](https://iphelix.medium.com/meebit-nft-exploit-analysis-c9417b804f89)
- **Connection**: Prime example of same-transaction trait assignment vulnerability where users could inspect and selectively reject NFTs based on attributes

### DEOS Games EOS Gambling Exploit (2018)

- **Loss**: Attacker won jackpot 24 times consecutively, netting ~$23,640 from initial $1,695 deposit [[4]](https://news.sophos.com/en-us/2018/09/14/blockchain-hustler-beats-the-house-with-smart-contract-hack/) [[5]](https://thenextweb.com/news/eos-betting-platform-hacked)
- **Method**: Exploited predictable randomness in EOS smart contract by manipulating timing and using custom contract to only execute winning transactions, reverting losing ones [[6]](https://peckshield.medium.com/defeating-eos-gambling-games-the-tech-behind-random-number-loophole-cf701c616dc0) [[7]](https://www.apriorit.com/dev-blog/588-eos-smart-contract-vulnerabilities)
- **Connection**: Demonstrates how predictable on-chain randomness in gambling applications can be systematically exploited through transaction reversion techniques

### The Idols NFT Self-Transfer Exploit (2025)

- **Loss**: ~$340,000 in stETH stolen through reward manipulation [[8]](https://quadrigainitiative.com/cryptocurrencyhackscamfraudwiki/index.php?title=The_Idols_NFT_Self_Reflection_Rewards_Vulnerability)
- **Method**: Attacker exploited flawed _beforeTokenTransfer logic by repeatedly self-transferring NFT #940, resetting reward snapshots while claiming inflated stETH rewards each time [[9]](https://blog.bunzz.dev/the-idols-nft-340k-exploit-hack-analysis/) [[10]](https://www.quillaudits.com/blog/hack-analysis/idols-nft-exploit-self-transfer-bug)
- **Connection**: While not pure randomness, shows how predictable contract state changes can be manipulated through repeated function calls to drain rewards
