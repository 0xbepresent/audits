# Original link
https://github.com/code-423n4/2023-05-venus-findings/issues/486
# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Comptroller.sol#L199
https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Comptroller.sol#L299
https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Comptroller.sol#L324
https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Comptroller.sol#L553
https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Comptroller.sol#L1240
https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Comptroller.sol#L1255
https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Comptroller.sol#L578


# Vulnerability details

## Impact
Incorrect borrowBalance and token collateral values. Could lead to many different exploits, such has:
- Users with a collateral token that fell in price substancially can borrow another underlying whose price has not been updated and earn profit.
- Users can borrow/redeem/transfer more if the interest/price was not updated.

## Proof of Concept
In the `Comptroller`, the total collateral balance and borrow balance is calculated at `_getHypotheticalLiquiditySnapshot(...)`. This function calculates these balances in the following loop:
```
for (uint256 i; i < assetsCount; ++i) {
    VToken asset = assets[i];

    // Read the balances and exchange rate from the vToken
    (uint256 vTokenBalance, uint256 borrowBalance, uint256 exchangeRateMantissa) = _safeGetAccountSnapshot(
        asset,
        account
    );

    // Get the normalized price of the asset
    Exp memory oraclePrice = Exp({ mantissa: _safeGetUnderlyingPrice(asset) });

    // Pre-compute conversion factors from vTokens -> usd
    Exp memory vTokenPrice = mul_(Exp({ mantissa: exchangeRateMantissa }), oraclePrice);
    Exp memory weightedVTokenPrice = mul_(weight(asset), vTokenPrice);

    // weightedCollateral += weightedVTokenPrice * vTokenBalance
    snapshot.weightedCollateral = mul_ScalarTruncateAddUInt(
        weightedVTokenPrice,
        vTokenBalance,
        snapshot.weightedCollateral
    );

    // totalCollateral += vTokenPrice * vTokenBalance
    snapshot.totalCollateral = mul_ScalarTruncateAddUInt(vTokenPrice, vTokenBalance, snapshot.totalCollateral);

    // borrows += oraclePrice * borrowBalance
    snapshot.borrows = mul_ScalarTruncateAddUInt(oraclePrice, borrowBalance, snapshot.borrows);

    // Calculate effects of interacting with vTokenModify
    if (asset == vTokenModify) {
        // redeem effect
        // effects += tokensToDenom * redeemTokens
        snapshot.effects = mul_ScalarTruncateAddUInt(weightedVTokenPrice, redeemTokens, snapshot.effects);

        // borrow effect
        // effects += oraclePrice * borrowAmount
        snapshot.effects = mul_ScalarTruncateAddUInt(oraclePrice, borrowAmount, snapshot.effects);
    }
}
```

As can be seen, the oracle price is not updated via calling `updatePrice(...)`, nor the borrow interest is updated by calling `AccrueInterest(...)`. Only the corresponding `VToken` that called the `borrow(...)`, transfer(...)` or `redeem(...)` has an updated price and interest, which could lead to critical inaccuracies for accounts with several VTokens.

## Tools Used
Vscode, Hardhat

## Recommended Mitigation Steps
Update the price and interest of every collateral except the VToken that triggered the hook, which has already been updated. Similarly to what is being done on `healAccount(...)`.
```
for (uint256 i; i < userAssetsCount; ++i) {
   userAssets[i].accrueInterest();
   oracle.updatePrice(address(userAssets[i]));
}
```


## Assessed type

Oracle