# Findings for 2022-12-gogopool 

- [[MEDIUM]]([MEDIUM]-1515216228/README.md) - The Staking.sol::restakeGGP() function does not have the whenNotPaused modifier.
- [[MEDIUM]]([MEDIUM]-1515207705/README.md) - The minipools creation could be compromised if is not possible to register more multisigs and all of them are disabled
- [[MEDIUM]]([MEDIUM]-1514768272/README.md) - Users may not be able to redeem their shares due to underflow
- [[MEDIUM]]([MEDIUM]-1515190085/README.md) - MinipoolManager.sol::recordStakingEnd() reverts when there are no rewards and duration is zero causing funds not to be returned
- [[HIGH]]([HIGH]-1514850762/README.md) - Initial ```ERC4626x.sol::deposit()``` can manipulate share prices
- [[MEDIUM]]([MEDIUM]-1516712534/README.md) - NodeOp can get rewards even if there was an error in registering the node as a validator
- [[MEDIUM]]([MEDIUM]-1515198928/README.md) - MinipoolManager.sol::recreateMinipool() uses a wrong reward for the avaxLiquidStakerAmt causing to re-stake more Avax than it should be
- [[MEDIUM]]([MEDIUM]-1517684234/README.md) - NodeOp funds may be trapped by a invalid state transition
- [[MEDIUM]]([MEDIUM]-1516700790/README.md) - NodeOp with error in registering the node as a validator can withdraw his funds and still have an active minipool.
- [[MEDIUM]]([MEDIUM]-1515166334/README.md) - MinipoolManager.sol::recreateMinipool() should verify if the multisig is still enabled
