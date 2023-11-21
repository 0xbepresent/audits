# Original link
https://github.com/code-423n4/2023-07-pooltogether-findings/issues/152
# Lines of code

https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L491
https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L590


# Vulnerability details

## Impact

The [TwabController.delegate()](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L491) function helps to delegate the user's balance to another address. The problem is that if the user accidentally/intentionally delegates to zero address, his funds will be trapped in the `vault` contract.

## Proof of Concept

I created a test where `Alice` deposits 1000e18 to the `Vault` and the funds will be trapped. Test steps:

1. Alice deposits 1000e18 to the vault.
2. Alice accidentally/intentionally delegates to zero address.
3. Assert the `maxWithdraw()` function returns the amount available to withdraw.
4. Alice calls the `withdraw()` function but is reverted by `DelegateBalanceLTAmount.selector`.
5. Alice tries to delegate from `address(0)` to `herself` but the function is reverted by `SameDelegateAlreadySet.selector`.
6. Alice tries to delegate from `address(0)` to `another account` but the function is reverted by `DelegateBalanceLTAmount.selector`.

```solidity
// File: test/unit/Vault/Withdraw.t.sol:VaultWithdrawTest
// $ forge test --match-test "testWithdrawRevertIfUserDelegatesToZero"
//
import { SameDelegateAlreadySet } from "v5-twab-controller/TwabController.sol";
import { DelegateBalanceLTAmount } from "v5-twab-controller/libraries/TwabLib.sol";

  function testWithdrawRevertIfUserDelegatesToZero() external {
    // User funds may be trapped in the vault if he delegates to zero address
    // 1. Alice deposit 1000e18 to the vault
    // 2. Alice accidentally/intentionally delegates to zero
    // 3. Assert the maxWithdraw() function returns the amount available to withdraw.
    // 4. Alice calls the withdraw() function but is reverted by "DelegateBalanceLTAmount.selector"
    // 5. Alice tries to delegate from address(0) to herself but the function is reverted by "SameDelegateAlreadySet.selector".
    // 6. Alice tries to delegate from address(0) to another account but the function is reverted by "DelegateBalanceLTAmount.selector".
    //
    // 1. Alice deposit 1000e18 to the vault 
    //
    vm.startPrank(alice);
    uint256 _amount = 1000e18;
    underlyingAsset.mint(alice, _amount);
    _deposit(underlyingAsset, vault, _amount, alice);
    //
    // 2. Alice accidentally/intentionally delegates to zero
    //
    twabController.delegate(address(vault), address(0));
    //
    // 3. Assert the maxWithdraw() function returns the amount available to withdraw.
    //
    uint256 maxWithdraw = vault.maxWithdraw(alice);
    assertEq(maxWithdraw, _amount);
    //
    // 4. Alice calls the withdraw() function but is reverted by "DelegateBalanceLTAmount.selector"
    //
    vm.expectRevert(abi.encodeWithSelector(DelegateBalanceLTAmount.selector, 0, 1e21, "TC/observation-burn-lt-delegate-balance"));
    vault.withdraw(maxWithdraw, alice, alice);
    //
    // 5. Alice tries to delegate from address(0) to herself but the function is reverted by "SameDelegateAlreadySet.selector".
    // 
    vm.expectRevert(abi.encodeWithSelector(SameDelegateAlreadySet.selector, alice));
    twabController.delegate(address(vault), alice);
    //
    // 6. Alice tries to delegate from address(0) to another account but the function is reverted by "DelegateBalanceLTAmount.selector".
    // 
    vm.expectRevert(abi.encodeWithSelector(DelegateBalanceLTAmount.selector, 0, 1e21, "TC/observation-burn-lt-delegate-balance"));
    twabController.delegate(address(vault), bob);

    vm.stopPrank();
  }
```

## Tools used

Manual review

## Recommended Mitigation Steps

SInce the [TwabController._delegateOf()](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L596-L600) function considers that zero address is the user who is delegating, implements a validation in the [TwabController.delegate()](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L491C12-L491C20) function that if the delegation is to the zero addres then use the same user who is delegating. 




## Assessed type

Invalid Validation