# Original link
https://github.com/code-423n4/2023-04-caviar-findings/issues/335
# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L733
https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L385


# Vulnerability details

## Impact

The private pool does not have any restriction about the [base token](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L175), so the creator can initialize the private pool with any token. The problem is the the private pool can be initialized with tokens of 3 decimals or less causing reverts in some functions.

The [changeFeeQuote()](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L731) function will be reverted by an ```arithmetic error``` causing that all trades via [change()](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L416) function will fail.

## Proof of Concept

I created a test where a private pool is created with a 3 decimals token, then the ```changeFeeQuote()``` function will be reverted by an arithmeticError causing that all the trades via ```change()``` function will fail.

```solidity
File: Change.t.sol
324:     function test_RevertIfWithThreeDecimalsToken() public {
325:         // PrivatePools does not support tokens with decimals less than four.
326:         // 1. Create a private pool with 3 decimals token.
327:         // 2. The changeFeeQuote() function will fail due an arithmeticError causing that
328:         //    all the trades via change() function will fail.
329:         //
330:         // 1. Create a private pool with 3 decimals token.
331:         //
332:         privatePool = new PrivatePool(address(factory), address(royaltyRegistry), address(stolenNftOracle));
333:         privatePool.initialize(
334:             address(threeDecimalsToken),
335:             nft,
336:             virtualBaseTokenReserves,
337:             virtualNftReserves,
338:             changeFee,
339:             feeRate,
340:             merkleRoot,
341:             true,
342:             false
343:         );
344:         //
345:         // 2. The changeFeeQuote() function will fail due an arithmeticError, causing that
346:         //    all the trades via change() function will fail.
347:         //
348:         vm.expectRevert(stdError.arithmeticError);
349:         privatePool.changeFeeQuote(1);
350:     }
```

## Tools used

VScode/Foundry

## Recommended Mitigation Steps

In the private pool creation validates that the ```base token``` is 4 decimals places or more.