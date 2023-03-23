# Original link
https://github.com/code-423n4/2022-12-tigris-findings/issues/259
# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L952
https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L163
https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L255
https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L480
https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L689


# Vulnerability details

## Impact

The ```Trading.initiateMarketOrder()```, ```Trading.addPosition()``` and ```Trading.executeLimitOrder()``` pay fees to the DAO, Referrer and Bot with help of the [_handleOpenFees()](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L689) function. The [setFees()](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L952) help to set the fees by the owner.

The ```initiateMarketOrder()```, ```addPosition()``` and ```executeLimitOrder()``` transactions can be frontrun with a fee increase. Furthermore, the user can [create a limitOrder](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L314) with X fee in mind and when the LimitOrder is executed by ```executeLimitOrder()``` function, the fee could have changed in that period of time

This Admin Privilege issue can be used to raise up the fees and the protocol users could get stronger security guarantees by having the fee increases behind governance timelock.

## Proof of Concept

The [setFees](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L952) function can be called by the owner at any time.
```solidity
function setFees(bool _open, uint _daoFees, uint _burnFees, uint _referralFees, uint _botFees, uint _percent) external onlyOwner {
    unchecked {
        require(_daoFees >= _botFees+_referralFees*2);
        if (_open) {
            openFees.daoFees = _daoFees;
            openFees.burnFees = _burnFees;
            openFees.referralFees = _referralFees;
            openFees.botFees = _botFees;
        } else {
            closeFees.daoFees = _daoFees;
            closeFees.burnFees = _burnFees;
            closeFees.referralFees = _referralFees;
            closeFees.botFees = _botFees;                
        }
        require(_percent <= DIVISION_CONSTANT);
        vaultFundingPercent = _percent;
    }
}
```

## Tools used
VsCode

## Recommended Mitigation Steps

The ```setFees()``` should be behind a TimeLock process and add a variable "MAX_FEE" to ensure fees can not go up above the threshold.