# Access Control

## Major Real-World Incidents Involving Similar Vulnerabilities

### Audius Governance Takeover (July 2022)

- **Loss:** 18 million AUDIO tokens worth about $6.1 million were drained, then swapped for 705 ETH (~$1 million). [[1]](https://www.bitdefender.com/en-us/blog/hotforsecurity/hacker-exploits-bug-at-decentralized-music-platform-audius-steals-6-million-worth-of-tokens)
- **Method:** The attacker exploited an uninitialized UUPS-proxy implementation. By calling the initialize function twice, they seized the "guardian" role that controls upgrades and proposals, then executed a malicious proposal to transfer treasury funds. [[2]](https://hashdex.com/fr-EU/insights/three-lessons-from-the-audius-hack) [[3]](https://blog.audius.co/article/audius-governance-takeover-post-mortem-7-23-22)
- **Connection:** Mirrors the NativeVault issue's root cause—missing initialization / role validation in upgradeable contracts allowed a rogue upgrade that redirected funds.

### PAID Network Infinite-Mint Attack (March 2021)

- **Loss:** 59.4 million PAID tokens minted (initial paper value ≈ $166 million); attacker realized ~2,040 ETH (~$3 million) before liquidity was pulled. [[4]](https://cryptobriefing.com/hacker-peforms-3-million-attack-on-paid-network/)
- **Method:** A compromised proxy-admin key upgraded the token contract to a malicious implementation that added unrestricted mint and burn, then printed tokens and dumped them. [[5]](https://www.certik.com/resources/blog/paid-network-post-mortem)
- **Connection:** Shows the danger of unrestricted upgrade rights. As with NativeVault's changeNodeImplementation, a single privileged role could deploy malicious logic instantly.
