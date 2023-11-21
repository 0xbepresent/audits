# Original link
https://github.com/code-423n4/2023-03-asymmetry-findings/issues/544
# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L120
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L146-L149
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L170


# Vulnerability details

## Impact

The [Reth.sol::poolCanDeposit()](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L120) function is used to [determine](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L170) if the users deposits are going to be via Rocket deposit pool or via swapping. The ```poolCanDeposit()``` function [checks](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/Reth.sol#L146-L149) if the deposit amount is in the min/max ranges.

The problem is that [rocketDAOProtocolSettingsDeposit.getDepositEnabled()](https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/dao/protocol/settings/RocketDAOProtocolSettingsDeposit.sol#L31) is not used to determine if the deposits to the Rocket pool are enabled. The user staking will [revert](https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/deposit/RocketDepositPool.sol#L77) if the Rocket deposits are not enabled by the Rocket protocol, causing a bad experience to the stakers because the ```stake()``` function will not be available due the reverts.

## Proof of Concept

The ```poolCanDeposit()``` checks in the 147-149 code lines that the deposit amount is in the min/max range but it doesn't check if the deposits are enabled via the ```rocketDAOProtocolSettingsDeposit.getDepositEnabled()``` function.

```solidity
File: Reth.sol
120:     function poolCanDeposit(uint256 _amount) private view returns (bool) {
121:         address rocketDepositPoolAddress = RocketStorageInterface(
122:             ROCKET_STORAGE_ADDRESS
123:         ).getAddress(
124:                 keccak256(
125:                     abi.encodePacked("contract.address", "rocketDepositPool")
126:                 )
127:             );
128:         RocketDepositPoolInterface rocketDepositPool = RocketDepositPoolInterface(
129:                 rocketDepositPoolAddress
130:             );
131: 
132:         address rocketProtocolSettingsAddress = RocketStorageInterface(
133:             ROCKET_STORAGE_ADDRESS
134:         ).getAddress(
135:                 keccak256(
136:                     abi.encodePacked(
137:                         "contract.address",
138:                         "rocketDAOProtocolSettingsDeposit"
139:                     )
140:                 )
141:             );
142:         RocketDAOProtocolSettingsDepositInterface rocketDAOProtocolSettingsDeposit = RocketDAOProtocolSettingsDepositInterface(
143:                 rocketProtocolSettingsAddress
144:             );
145: 
146:         return
147:             rocketDepositPool.getBalance() + _amount <=
148:             rocketDAOProtocolSettingsDeposit.getMaximumDepositPoolSize() &&
149:             _amount >= rocketDAOProtocolSettingsDeposit.getMinimumDeposit();
150:     }
```

## Tools used

VScode

## Recommended Mitigation Steps

Add a verification via ```rocketDAOProtocolSettingsDeposit.getDepositEnabled()``` if the deposits are enabled:

```solidity
    return
        rocketDepositPool.getBalance() + _amount <=
        rocketDAOProtocolSettingsDeposit.getMaximumDepositPoolSize() &&
        _amount >= rocketDAOProtocolSettingsDeposit.getMinimumDeposit() &&
        rocketDAOProtocolSettingsDeposit.getDepositEnabled();
```