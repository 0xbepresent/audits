# Original link
https://github.com/code-423n4/2023-03-asymmetry-findings/issues/381
# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L228
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L116
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L73
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L92


# Vulnerability details

## Impact

In the ```Reth.sol``` contract the [poolPrice()](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L228) function helps to calculate the ```rETH``` per ```ETH```. On the other side, in the ```SfrxEth.sol``` contract the [SfrxEth.sol::ethPerDerivative()](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L111) function uses the ```FRX/ETH``` curve pool to get the derivative price in terms of ```ETH```.

Those prices extracted from the Uniswap and Curve pools are used when the [users stake](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#LL73) their ```ETH```, so the underlying value is calculated and it is used to calculate [the depositPrice](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L81) and finally the depositPrice is [used to calculate](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L98) the mint amount of ```SafEth``` tokens.

The problem is that the [Uniswap rETH/wETH pool](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L237) and the [Curve Frx/ETH pool](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L116) can be manipulated in order to get more ```safETH``` tokens with less ```ETH```. The attacker can manipulate the price pools with flash loans leading to mint more ```safETH``` tokens.

## Proof of Concept

The ```Reth.sol::poolPrice()``` relies on Uniswap pool price.

```solidity
File: Reth.sol
228:     function poolPrice() private view returns (uint256) {
229:         address rocketTokenRETHAddress = RocketStorageInterface(
230:             ROCKET_STORAGE_ADDRESS
231:         ).getAddress(
232:                 keccak256(
233:                     abi.encodePacked("contract.address", "rocketTokenRETH")
234:                 )
235:             );
236:         IUniswapV3Factory factory = IUniswapV3Factory(UNI_V3_FACTORY);
237:         IUniswapV3Pool pool = IUniswapV3Pool(
238:             factory.getPool(rocketTokenRETHAddress, W_ETH_ADDRESS, 500)
239:         );
240:         (uint160 sqrtPriceX96, , , , , , ) = pool.slot0();
241:         return (sqrtPriceX96 * (uint(sqrtPriceX96)) * (1e18)) >> (96 * 2);
242:     }
```

The ```SfrxEth.sol::ethPerDerivative()``` relies on the Curve pool price.

```solidity
File: SfrxEth.sol
111:     function ethPerDerivative(uint256 _amount) public view returns (uint256) {
112:         uint256 frxAmount = IsFrxEth(SFRX_ETH_ADDRESS).convertToAssets(
113:             10 ** 18
114:         );
115:         return ((10 ** 18 * frxAmount) /
116:             IFrxEthEthPool(FRX_ETH_CRV_POOL_ADDRESS).price_oracle());
117:     }
```

## Tools used

VScode

## Recommended Mitigation Steps

Secured price calculation can be performed by using time-weighted average prices (TWAPs) across longer time intervals.