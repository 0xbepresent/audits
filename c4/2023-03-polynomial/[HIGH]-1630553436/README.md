# Original link
https://github.com/code-423n4/2023-03-polynomial-findings/issues/62
# Lines of code

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L494
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L511


# Vulnerability details

## Impact

The protocol does not adequately increase the calculated fees to the ```totalFunds``` storage variable in the open short operations consequently the liquidity providers receive less yields and the protocol receive less funds. Those calculated fees are lost.

## Proof of Concept

The fees are calculated in the 511 code line but those are not increased to the ```totalFunds``` storage variable:

```solidity
File: LiquidityPool.sol
491:     /// @notice Called by exchange when a new short position is created
492:     /// @param amount Amount of square perp being shorted
493:     /// @param user Address of the user
494:     function openShort(uint256 amount, address user, bytes32 referralCode)
495:         external
496:         override
497:         onlyExchange
498:         nonReentrant
499:         returns (uint256 totalCost)
500:     {
501:         (uint256 markPrice, bool isInvalid) = getMarkPrice();
502:         require(!isInvalid);
503: 
504:         uint256 tradeCost = amount.mulWadDown(markPrice);
505:         uint256 fees = orderFee(-int256(amount));
506:         totalCost = tradeCost - fees;
507: 
508:         SUSD.safeTransfer(user, totalCost);
509: 
510:         uint256 hedgingFees = _hedge(-int256(amount), false);
511:         uint256 feesCollected = fees - hedgingFees;
512:         uint256 externalFee = feesCollected.mulWadDown(devFee);
513: 
514:         SUSD.safeTransfer(feeReceipient, externalFee);
515: 
516:         usedFunds += int256(totalCost + hedgingFees + externalFee);
517:         require(usedFunds <= 0 || totalFunds >= uint256(usedFunds));
518: 
519:         emit RegisterTrade(referralCode, feesCollected, externalFee);
520:         emit OpenShort(markPrice, amount, fees);
521:     }
```

## Tools used

VScode

## Recommended Mitigation Steps

Increment the ```feesCollected``` to the ```totalFunds``` storage variable in the ```openShort()``` function.