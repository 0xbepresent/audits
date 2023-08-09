# Original link
https://github.com/code-423n4/2023-03-polynomial-findings/issues/64
# Lines of code

https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L200
https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L219


# Vulnerability details

## Impact

The [LiquidityPool.sol::queueDeposit()](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L200) function helps to queue the user deposit and then the deposit will be processed by the [LiquidityPool.sol::processDeposits()](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L200) function after the delay is completed. The deposits are used to cover the [margin required in every operation](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L807).

The problem is that an attacker can create many deposits with ```zero amounts```, so it is possible to flood the [depositQueue](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L206) with zero amount deposits and the ```processDeposits()``` function will cost more work/gas to process.

Please see the next scenario:
1. For some reason there are not ```sUSD``` in the LiquidityPool.
2. The protocol needs to cover his margins using the [increaseMargin()](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L613) function but is not possible because there are not any ```sUSD``` in the LiquidityPool.
3. The liquidity providers use ```queueDeposit()``` function because the function doesn't have fees.
4. The ```attacker``` frontruns the liquidity providers and he floods the ```depositQueue``` with zero amounts causing the processing of the ```processDeposits()``` function to cost more work/gas.
5. The protocol can't hedge his exposure and they need to use the [deposit()](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L184) function instead.

At the end, the attacker can cause the ```processDeposits()``` function to take more work/gas to process, causing valid deposits to be unavailable.

## Proof of Concept

I created a basic test where you can see that it is possible to queue deposits with amount zero:

```solidity
function testZeroQueueDeposit() public {
    // It is possible to flood the depositQueue with zero amounts in the queueDeposit()
    // causing a DOS in the processDeposits()
    // 1. Call the queueDeposit() with zero amounts
    // 2. Assert queuedDepositHead and nextQueuedDepositId
    // 3. Call the processDeposits()
    // 4. Assert the incremention of queuedDepositHead
    //
    // 1. Call the queueDeposit() with zero amounts
    //
    pool.queueDeposit(0, user_1);
    pool.queueDeposit(0, user_2);
    pool.queueDeposit(0, user_3);
    //
    // 2. Assert queuedDepositHead and nextQueuedDepositId
    //
    assertEq(pool.queuedDepositHead(), 1);
    assertEq(pool.nextQueuedDepositId(), 4);
    //
    // 3. Call the processDeposits()
    //
    vm.warp(block.timestamp + pool.minDepositDelay());
    pool.processDeposits(1);
    //
    // 4. Assert the incremention of queuedDepositHead
    //
    assertEq(pool.queuedDepositHead(), 2);
}
```

## Tools used

VScode

## Recommended Mitigation Steps

Checks that the deposited ```amount``` is not zero in the ```queueDeposit()``` function.