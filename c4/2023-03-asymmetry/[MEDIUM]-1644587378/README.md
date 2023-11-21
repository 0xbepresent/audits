# Original link
https://github.com/code-423n4/2023-03-asymmetry-findings/issues/375
# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L92
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L177


# Vulnerability details

## Impact

The ```Reth.sol``` contract is used by the protocol to manage the user's staking in that ```Derivative```. When the Rocket pool doesn't accept deposits directly, the [code uses an Uniswap swap](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L177) in order to get ```rETH``` in exchange of the user's ```ETH```. The [UniswapRouter.exactInputSingle()](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L101) is used to obtain the ```rETH``` tokens.

In UniSwap, there is a parameter called [deadline](https://docs.uniswap.org/contracts/v3/guides/swaps/single-swaps#swap-input-parameters) which helps to protect against long-pending transactions and wild swings in prices.

The problem here is that the [deadline](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L92) parameter is not used when a swap is executing. If the ```deadline``` parameter is not present, the protocol can perform bad trades:

- The [minOut](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L173) can be outdated. The ```minOut``` amount can be calculated at a certain minimum acceptable amount, so the trade can be pending in the ```mempool``` for many circunstances like miners are not incentivited, and while the transaction is pending the ```rETH/wETH``` price can change and the ```minOut``` can be outdated.
- The transaction can be pending in the mempool for extended periods of time which it could be a problematic when the ```rETH/ETH``` prices changes constantly.

The ```deadline``` parameter helps to protect the protocol swap trades against fluctuation in the market prices and miners who decide how fast the transaction are executed.

## Proof of Concept

The ```Reth.sol::swapExactInputSingleHop()``` function doesn't use the deadline parameter.

```solidity
File: Reth.sol
083:     function swapExactInputSingleHop(
084:         address _tokenIn,
085:         address _tokenOut,
086:         uint24 _poolFee,
087:         uint256 _amountIn,
088:         uint256 _minOut
089:     ) private returns (uint256 amountOut) {
090:         IERC20(_tokenIn).approve(UNISWAP_ROUTER, _amountIn);
091:         ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
092:             .ExactInputSingleParams({
093:                 tokenIn: _tokenIn,
094:                 tokenOut: _tokenOut,
095:                 fee: _poolFee,
096:                 recipient: address(this),
097:                 amountIn: _amountIn,
098:                 amountOutMinimum: _minOut,
099:                 sqrtPriceLimitX96: 0
100:             });
101:         amountOut = ISwapRouter(UNISWAP_ROUTER).exactInputSingle(params);
102:     }
```

## Tools used

VScode

## Recommended Mitigation Steps

Add the ```deadline``` parameter in order to ensure that the transaction can be executed in a short period of time.