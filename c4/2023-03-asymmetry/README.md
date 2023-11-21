# Findings for 2023-03-asymmetry 

- [[MEDIUM]]([MEDIUM]-1646610675/README.md) - ```Reth.sol::poolCanDeposit()``` does not implement the ```rocketDAOProtocolSettingsDeposit.getDepositEnabled()``` check
- [[MEDIUM]]([MEDIUM]-1644587378/README.md) - The Uniswap ```deadline``` parameter is not used in the swap actions causing perform bad trades
- [[MEDIUM]]([MEDIUM]-1646654110/README.md) - ```frxETHMinter._submit()``` function can be paused by the FRX Protocol causing malfunction in the ```SafEth.sol::staking()```
- [[HIGH]]([HIGH]-1644687200/README.md) - Spot price of a decentralized exchange can be manipulated affecting the protocol's internal price
