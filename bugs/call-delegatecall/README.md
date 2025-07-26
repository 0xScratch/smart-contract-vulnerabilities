# Call and Delegatecall Vulnerability

## Major Real-World Incidents Involving Similar Vulnerabilities

### SpankChain Reentrancy Attack (October 2018)

- **Loss**: 165.38 ETH stolen (~$38,000) and 4,000 BOOTY tokens frozen [[1]](https://www.coindesk.com/markets/2018/10/12/spankchain-says-hacker-returned-stolen-crypto-funds/)
- **Method**: While primarily a reentrancy attack, it involved unchecked external calls where the malicious contract's transfer function repeatedly called back into the payment channel contract without proper return value validation [[2]](https://medium.com/swlh/how-spankchain-got-hacked-af65b933393c)
- **Connection**: The attack exploited both reentrancy and unchecked call return values, where failed external calls weren't properly handled, allowing the attacker to repeatedly extract funds

### Parity MultiSig Wallet Library Bug (November 2017)

- **Loss**: 513,774 ETH and tokens permanently frozen (~$165 million) [[3]](https://web.archive.org/web/20210227150251/https:/www.parity.io/the-multi-sig-hack-a-postmortem/)
- **Method**: While not directly an unchecked call issue, the vulnerability involved improper delegatecall usage without adequate return value validation and access control, allowing a user to accidentally destroy the library contract [[4]](https://blog.openzeppelin.com/parity-wallet-hack-reloaded)
- **Connection**: Related to call-delegatecall vulnerability category where external calls weren't properly secured or validated, demonstrating the broader risks of unchecked external interactions
