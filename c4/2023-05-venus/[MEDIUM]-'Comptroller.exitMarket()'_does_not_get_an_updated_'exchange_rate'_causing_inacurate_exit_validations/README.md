# Original link
https://github.com/code-423n4/2023-05-venus-findings/issues/502
# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L187
https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L1240
https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L1296
https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L1402
https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/VToken.sol#L1463
https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/VToken.sol#L678


# Vulnerability details

## Impact

The user can use the [exitMarket()](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L187) function to remove an asset from the account liquidity calculation; disabling them as collateral.

The problem is that the user can call `exitMarket()` before the vToken updates their accrue interests via [accrueInterest()](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/VToken.sol#L678) function causing that the user can exit from the market even if he has debts `snapshot.shortfall > 0`.

The [interests are increased](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/VToken.sol#L712) when the chain block increases. If the `exitMarket()` is called before anyone can call `accrueInterest()`, the `exitMarket()` function will have outdated data to validates if the user can exit the market or not.

## Proof of Concept

Please see the next path execution:

1. The [exitMarket()](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L187) calls [_checkRedeemAllowed()](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L199)
2. The `_checkRedeemAllowed()` function calls [_getHypotheticalLiquiditySnapshot()](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#LL1255C52-L1255C85) function.
3. The `_getHypotheticalLiquiditySnapshot()` function calls [_safeGetAccountSnapshot()](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#LL1311C92-L1311C115).
4. The `_safeGetAccountSnapshot()` function calls [vToken.getAccountSnapshot()](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#LL1412C69-L1412C94)
5. The `vToken.getAccountSnapshot()` calls [_exchangeRateStored()](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/VToken.sol#L572) function.
6. The `_exchangeRateStored()` function calculates the exchange rate [based on the totalBorrows](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/VToken.sol#L1477). The exchange rate here could be outdated because the [vToken.accrueInterest()](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/VToken.sol#L727) is responsable to calculate the new `total borrows` and it is possible to call `_exchangeRateStored()` before anyone can call `accrueInterests()`.
7. The outdated exchange rate calculated will be used in the [vToken price calculation](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L1320) causing to have innacurate information to [decide if the borrower is shortfall or not](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L1351).
8. The `_checkRedeemAllowed()` function may evaluate the [shortfall check incorrectly](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L1262).

The interests are increased by the [block delta](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/VToken.sol#L710). So the `exitMarket()` must calculate the current block interests to be able to check if the user is able to exit the market or not.

## Tools used

VSCode

## Recommended Mitigation Steps

Accrue the vToken interests [accrueInterest()](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/VToken.sol#L678) once the user call `exitMarket()` function so the function can get updated info and it is possible to validate if the user is able to exit or not with updated info.





## Assessed type

Other