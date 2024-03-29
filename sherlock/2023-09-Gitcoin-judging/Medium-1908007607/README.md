# Original link
https://github.com/sherlock-audit/2023-09-Gitcoin-judging/issues/536
0xbepresent

medium

# Pool's strategies does not support `fee on transfer` tokens causing an error in the counting system

There is not support on `fee on transfer` tokens when funding a pool causing an error in the counting system.

## Vulnerability Detail

The [Allo::_fundPool()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/core/Allo.sol#L502) function helps to depositants to [send tokens to the _strategy](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/core/Allo.sol#L516). The problem is that pools can use `fee on transfer` tokens, so when the transfer transaction occurs the current amount sent to the strategy could be less.

The transfer occurs in the [code line 516](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/core/Allo.sol#L516) using the amount `amountAfterFee`, then in the [code line 517](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/core/Allo.sol#L517) the strategy amount is increased:

```solidity
File: Allo.sol
502:     function _fundPool(uint256 _amount, uint256 _poolId, IStrategy _strategy) internal {
503:         uint256 feeAmount;
504:         uint256 amountAfterFee = _amount;
505: 
506:         Pool storage pool = pools[_poolId];
507:         address _token = pool.token;
508: 
509:         if (percentFee > 0) {
510:             feeAmount = (_amount * percentFee) / getFeeDenominator();
511:             amountAfterFee -= feeAmount;
512: 
513:             _transferAmountFrom(_token, TransferData({from: msg.sender, to: treasury, amount: feeAmount}));
514:         }
515: 
516:         _transferAmountFrom(_token, TransferData({from: msg.sender, to: address(_strategy), amount: amountAfterFee}));
517:         _strategy.increasePoolAmount(amountAfterFee);
518: 
519:         emit PoolFunded(_poolId, amountAfterFee, feeAmount);
520:     }
```

The [BaseStrategy::increasePoolAmount()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/BaseStrategy.sol#L153C14-L153C32) function is not considering the fee charged by the token.

## Impact

There are errors in the counting system ([poolAmount](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/BaseStrategy.sol#L155)) that can affect the distribution of the tokens to the recipients because the calculation can be for a certain amount but actually there could be a less amount in the pools. E.g. the [QVBaseStrategy::_getPayout()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L571) calculations may be using more tokens than the available in the pool.


## Code Snippet

- [Allo::_fundPool()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/core/Allo.sol#L502)
- [BaseStrategy::increasePoolAmount()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/BaseStrategy.sol#L153C14-L153C32)

## Tool used

Manual review

## Recommendation

Calculates the strategy balance before and after the transfer transaction:

```diff
   function _fundPool(uint256 _amount, uint256 _poolId, IStrategy _strategy) internal {
        uint256 feeAmount;
        uint256 amountAfterFee = _amount;

        Pool storage pool = pools[_poolId];
        address _token = pool.token;

        if (percentFee > 0) {
            feeAmount = (_amount * percentFee) / getFeeDenominator();
            amountAfterFee -= feeAmount;

            _transferAmountFrom(_token, TransferData({from: msg.sender, to: treasury, amount: feeAmount}));
        }
++      uint256 strategyBalanceBefore = _token.balanceOf(address(strategy));
        _transferAmountFrom(_token, TransferData({from: msg.sender, to: address(_strategy), amount: amountAfterFee}));
++      uint256 strategyBalanceAfter = _token.balanceOf(address(strategy));
--      _strategy.increasePoolAmount(amountAfterFee);
++      _strategy.increasePoolAmount(strategyBalanceAfter - strategyBalanceBefore);

        emit PoolFunded(_poolId, amountAfterFee, feeAmount);
    }
```

Of course elaborate a better solution that consider `NATIVE` tokens

Duplicate of #19
