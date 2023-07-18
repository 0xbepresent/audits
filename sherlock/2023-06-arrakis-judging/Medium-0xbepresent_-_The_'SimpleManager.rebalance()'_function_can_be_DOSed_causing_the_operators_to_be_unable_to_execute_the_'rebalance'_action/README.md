# Original link
https://github.com/sherlock-audit/2023-06-arrakis-judging/issues/132
0xbepresent

high

# The `SimpleManager.rebalance()` function can be DOSed causing the operators to be unable to execute the `rebalance` action

## Summary

The [SimpleManager.rebalance()](https://github.com/sherlock-audit/2023-06-arrakis/blob/main/v2-manager-templates/contracts/SimpleManager.sol#L128) function validates the [max price deviation](https://github.com/sherlock-audit/2023-06-arrakis/blob/main/v2-manager-templates/contracts/SimpleManager.sol#L189) using the [slot0()](https://github.com/sherlock-audit/2023-06-arrakis/blob/main/v2-manager-templates/contracts/SimpleManager.sol#L181) price calculation. Since the `slot0` can be manipulated easily, the attacker can DOS the `SimpleManager.rebalance()` function preventing the `rebalance` action to the `operators`.

## Vulnerability Detail

The [SimpleManager.rebalance()](https://github.com/sherlock-audit/2023-06-arrakis/blob/main/v2-manager-templates/contracts/SimpleManager.sol#L128) function is executed by the operators to `rebalance` the vault assets to Uniswap pools. The `SimpleManager.rebalance()` function validates the [max price deviation](https://github.com/sherlock-audit/2023-06-arrakis/blob/main/v2-manager-templates/contracts/SimpleManager.sol#L189) using the [slot0()](https://github.com/sherlock-audit/2023-06-arrakis/blob/main/v2-manager-templates/contracts/SimpleManager.sol#L181) and the [Oracle price](https://github.com/sherlock-audit/2023-06-arrakis/blob/main/v2-manager-templates/contracts/SimpleManager.sol#L159). The problem here is that the `slot0` is the most recent data and it is easy to manipulate.


Please see the next scenario:
1. An attacker see that the vault operator calls the `SimpleManager.rebalance()` function.
2. Immediately, the attacker inflates the pool price and the `slot0` is manipulated.
3. The `_checkDeviation()` function will revert the operator `rebalance` call because the price deviation between the `Oracle price` and the `pool price` is high. `require(deviation <= maxDeviation_, "maxDeviation");`.
4. The vault `rebalance` action is DOSed by an attacker.

## Impact

An attacker can block the `rebalance` action calls causing that the operators to unable to execute their strategies.

## Code Snippet

The [SimpleManager.rebalance()](https://github.com/sherlock-audit/2023-06-arrakis/blob/main/v2-manager-templates/contracts/SimpleManager.sol#L128):

```solidity
File: SimpleManager.sol
128:     function rebalance(
129:         address vault_,
130:         Rebalance calldata rebalanceParams_
131:     ) external {
...
...
157:         if (mintsLength > 0) {
158:             checked = new uint24[](mintsLength);
159:             oraclePrice = vaultInfo.oracle.getPrice0();
160:         }
...
...
178: 
179:             uint256 sqrtPriceX96;
180: 
181:             (sqrtPriceX96, , , , , , ) = pool.slot0();
...
...
189:             _checkDeviation(
190:                 poolPrice,
191:                 oraclePrice,
192:                 vaultInfo.maxDeviation,
193:                 token1Decimals
194:             );
...
```

The [SimpleManager._checkDeviation()](https://github.com/sherlock-audit/2023-06-arrakis/blob/main/v2-manager-templates/contracts/SimpleManager.sol#L366) function:

```solidity
File: SimpleManager.sol
366:     function _checkDeviation(
367:         uint256 currentPrice_,
368:         uint256 oraclePrice_,
369:         uint24 maxDeviation_,
370:         uint8 priceDecimals_
371:     ) internal pure {
372:         uint256 deviation = FullMath.mulDiv(
373:             FullMath.mulDiv(
374:                 currentPrice_ > oraclePrice_
375:                     ? currentPrice_ - oraclePrice_
376:                     : oraclePrice_ - currentPrice_,
377:                 10 ** priceDecimals_,
378:                 oraclePrice_
379:             ),
380:             hundred_percent,
381:             10 ** priceDecimals_
382:         );
383: 
384:         require(deviation <= maxDeviation_, "maxDeviation");
385:     }
```


## Tool used

Manual review

## Recommendation

Use `TWAP` price instead of `slot0` price.

Duplicate of #71
