# Original link
https://github.com/code-423n4/2023-03-asymmetry-findings/issues/932
# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/Reth.sol#L83-L102


# Vulnerability details

Lack of deadline for uniswap AMM

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/Reth.sol#L83-L102

## Proof of Concept

The ISwapRouter.exactInputSingle params (used in the rocketpool derivative) does not include a deadline currently.

```
ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
    .ExactInputSingleParams({
        tokenIn: _tokenIn,
        tokenOut: _tokenOut,
        fee: _poolFee,
        recipient: address(this),
        amountIn: _amountIn,
        amountOutMinimum: _minOut,
        sqrtPriceLimitX96: 0
    });
```

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/Reth.sol#L83-L102

The following scenario can happen:
1. User is staking and some/all the weight is in Reth.
2. The pool can't deposit eth, so uniswap will be used to convert eth to weth.
3. A validator holds the tx until it becomes advantageous due to some market condition (e.g. slippage or running his tx before and frontrun the original user stake).
4. This could potentially happen to a large amount of stakes, due to widespread usage of bots and MEV.

## Impact

Because Front-running is a key aspect of AMM design, deadline is a useful tool to ensure that your tx cannot be “saved for later”.

Due to the removal of the check, it may be more profitable for a validator to deny the transaction from being added until the transaction incurs the maximum amount of slippage.

## Tools Used

Manual review.

## Recommended Mitigation Steps

The `Reth.deposit()` function should accept a user-input `deadline` param that should be passed along to Reth.swapExactInputSingleHop() and ISwapRouter.exactInputSingle().
