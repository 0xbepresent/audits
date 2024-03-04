# Original link
https://github.com/code-423n4/2023-10-wildcat-findings/issues/592
# Lines of code

https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarket.sol#L142
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/MarketState.sol#L138
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/FeeMath.sol#L89


# Vulnerability details

# The borrower does not pay all debts when he closes the market

## Lines of code
- https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarket.sol#L142
- https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/MarketState.sol#L138
- https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/FeeMath.sol#L89

## Impact

When borrower closes the market using the [WildMarket::closeMarket()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarket.sol#L142), he [must pay all debts](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarket.sol#L154):

```solidity
File: WildcatMarket.sol
142:   function closeMarket() external onlyController nonReentrant {
...
...
151:     uint256 totalDebts = state.totalDebts();
152:     if (currentlyHeld < totalDebts) {
153:       // Transfer remaining debts from borrower
154:       asset.safeTransferFrom(borrower, address(this), totalDebts - currentlyHeld);
155:     } else if (currentlyHeld > totalDebts) {
...
...
```

The [totalDebts()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/MarketState.sol#L138) function calculates how much the borrower needs to pay back to the market. The problem is that the function does not count all the incurring debt when the borrower was in delinquent process. The penalty is increased when the [user is `delinquent` code line 97-111](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/FeeMath.sol#L97) and the penalty is being decreased when the [time elapsed code line 115](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/FeeMath.sol#L115).

```solidity
File: FeeMath.sol
089:   function updateTimeDelinquentAndGetPenaltyTime(
090:     MarketState memory state,
091:     uint256 delinquencyGracePeriod,
092:     uint256 timeDelta
093:   ) internal pure returns (uint256 /* timeWithPenalty */) {
094:     // Seconds in delinquency at last update
095:     uint256 previousTimeDelinquent = state.timeDelinquent;
096: 
097:     if (state.isDelinquent) {
098:       // Since the borrower is still delinquent, increase the total
099:       // time in delinquency by the time elapsed.
100:       state.timeDelinquent = (previousTimeDelinquent + timeDelta).toUint32();
101: 
102:       // Calculate the number of seconds the borrower had remaining
103:       // in the grace period.
104:       uint256 secondsRemainingWithoutPenalty = delinquencyGracePeriod.satSub(
105:         previousTimeDelinquent
106:       );
107: 
108:       // Penalties apply for the number of seconds the market spent in
109:       // delinquency outside of the grace period since the last update.
110:       return timeDelta.satSub(secondsRemainingWithoutPenalty);
111:     }
112: 
113:     // Reduce the total time in delinquency by the time elapsed, stopping
114:     // when it reaches zero.
115:     state.timeDelinquent = previousTimeDelinquent.satSub(timeDelta).toUint32();
116: 
117:     // Calculate the number of seconds the old timeDelinquent had remaining
118:     // outside the grace period, or zero if it was already in the grace period.
119:     uint256 secondsRemainingWithPenalty = previousTimeDelinquent.satSub(delinquencyGracePeriod);
120: 
121:     // Only apply penalties for the remaining time outside of the grace period.
122:     return MathUtils.min(secondsRemainingWithPenalty, timeDelta);
123:   }
```

The borrower will not pay all the debt incurred while he was in delinquent mode when borrower closes the market.


## Proof of Concept

Please consider the next scenario [taking in consideration the example from the documentation](https://wildcat-protocol.gitbook.io/wildcat/using-wildcat/delinquency#how-delinquency-triggers):

Example:

> - A market with a 5 day grace period becomes delinquent for the first time, and the grace tracker begins counting up from zero.
> - The borrower takes 7 days to cure the market of its delinquency.
> - Once the delinquency is cured, the market calculates that 2 days of penalty APR must be applied and adjusts the scaling factor of market tokens accordingly.
> - The grace tracker counts back down to zero from this point - subsequent market state updates will detect when the tracker drops below the market grace period and adjust scaling factors appropriately.
> - A total of 4 days of penalty APR will be applied in total: the failure of a market to update its state in a timely fashion as the tracker drops below the grace period does not adversely impact the borrower.

Now adapting the example above to the next scenario:
1. Borrower decides to close the market on day 7 and he only pays the delinquent penalty for 2 days and the [time left](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/libraries/FeeMath.sol#L115) (the 2 days left) is not counted in the total debt.
2. Borrower only pay penalty for 2 days instead of 4 days.

## Tools used

Manual review

## Recommended Mitigation Steps

When borrower closes the market, consider to calculate all the debt when the borrower was in delinquency.



## Assessed type

Math