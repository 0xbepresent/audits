# Original link
https://github.com/code-423n4/2023-10-wildcat-findings/issues/718
# Lines of code

https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L490
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarket.sol#L142


# Vulnerability details

## Impact

Once the market is closed the [reserveRatioBips is set to zero value](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarket.sol#L146).

```solidity
File: WildcatMarket.sol
142:   function closeMarket() external onlyController nonReentrant {
...
...
146:     state.reserveRatioBips = 0;
...
...
```

The problem is that there could be a [`temporaryExcessReserveRatio[market]` in progress](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L499), returning back the `state.reserveRatioBips` to a non zero value causing that the [liquidityRequired()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/MarketState.sol#L87C12-L87C29) to be positive because there is a non [zero reserve ratio](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/MarketState.sol#L92). This behaivour will cause that the borrower can be in deliquency even [when the market is closed](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L449).

## Proof of Concept

Please consider the next scenario:
1. Borrower calls [setAnnualInterestBips()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L468C12-L468C33) and the [`tmp.reserveRatioBips` is 100](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L478).
2. Borrower closes the market and [reserveRatioBps is zero](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarket.sol#L146).
3. After 2 weeks the function [resetReserveRatio()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L490C12-L490C29) returns the value to a non zero (100).

## Tools used

Manual review

## Recommended Mitigation Steps

Add a restriction in the [resetReserveRatio()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L490C12-L490C29) function that it could not be called when the market is closed.


## Assessed type

Access Control