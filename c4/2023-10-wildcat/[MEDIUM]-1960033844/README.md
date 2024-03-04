# Original link
https://github.com/code-423n4/2023-10-wildcat-findings/issues/244
# Lines of code

https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L468
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L410


# Vulnerability details

## Impact

When the market is deployed, the [annualInterestBips](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L313) is validated that the value is in range of [MinimumAnnualInterestBips and MaximumAnnualInterestBips](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L410-L415).

```solidity
File: WildcatMarketController.sol
394:   function enforceParameterConstraints(
395:     string memory namePrefix,
396:     string memory symbolPrefix,
397:     uint16 annualInterestBips,
398:     uint16 delinquencyFeeBips,
399:     uint32 withdrawalBatchDuration,
400:     uint16 reserveRatioBips,
401:     uint32 delinquencyGracePeriod
402:   ) internal view virtual {
...
...
410:     assertValueInRange(
411:       annualInterestBips,
412:       MinimumAnnualInterestBips,
413:       MaximumAnnualInterestBips,
414:       AnnualInterestBipsOutOfBounds.selector
415:     );
```

The problem is the validation is not executed when the borrower changes the `annualInterestBips` using [setAnnualInterestBips()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L468) function. The borrower can deploy a market using a validated `annualInterestBips` then changes it to a non validated using [setAnnualInterestBips()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L468). The [controller may have a certain MinimumAnnualInterestBips value](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketControllerFactory.sol#L99) which can by bypassed.

## Proof of Concept

I created a test where the `Controller.MinimumAnnualInterestBips` is `500` and the borrower calls [setAnnualInterestBips()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L468) function using `0` as the `annualInterestBips`. The `Controller.MinimumAnnualInterestBips` is bypassed:

```solidity
// File: test/market/WildcatMarket.t.sol:WildcatMarketTest
// $ forge test --match-test "test_BorrowerCanEvadeTheAnnualInterestBipsRestriction" -vvv
//
  function test_BorrowerCanEvadeTheAnnualInterestBipsRestriction() external asAccount(address(controller)) {
    //
    // 1. Alice deposits 1e18
    _deposit(alice, 1e18);
    //
    // 2. Assert the market.annualInterestBips() is 1000 and the MinimumAnnualInterestBips is 500
    assertEq(market.annualInterestBips(), 1000);
    MarketParameterConstraints memory constraints = controller.getParameterConstraints();
    assertEq(constraints.minimumAnnualInterestBips, 500);
    //
    // 3. Borrower calls setAnnualInterestBips() to 0 bps. Assert the new market.annualInterestBips() is 0. The constraints.minimumAnnualInterestBips is bypassed
    startPrank(borrower);
    controller.setAnnualInterestBips(address(market), 0);
    assertEq(market.annualInterestBips(), 0);
    assertEq(constraints.minimumAnnualInterestBips, 500);
  }
```

I modified the `TestConstants.sol` file in order to set the `Controller.MinimumAnnualInterestBips` to `500`:

```diff
--- a/test/shared/TestConstants.sol
+++ b/test/shared/TestConstants.sol
@@ -26,5 +26,5 @@ uint16 constant MaximumDelinquencyFeeBips = 10_000;
 uint32 constant MinimumWithdrawalBatchDuration = 0;
 uint32 constant MaximumWithdrawalBatchDuration = 365 days;
 
-uint16 constant MinimumAnnualInterestBips = 0;
+uint16 constant MinimumAnnualInterestBips = 500;
 uint16 constant MaximumAnnualInterestBips = 10_000;
```

## Tools used

Manual review

## Recommended Mitigation Steps

Validates the new `annualInterestBips` is in range `MinimumAnnualInterestBips-MaximumAnnualInterestBips` in the [setAnnualInterestBips()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L468) function:

```diff
  function setAnnualInterestBips(
    address market,
    uint16 annualInterestBips
  ) external virtual onlyBorrower onlyControlledMarket(market) {
++  assertValueInRange(
++    annualInterestBips,
++    MinimumAnnualInterestBips,
++    MaximumAnnualInterestBips,
++    AnnualInterestBipsOutOfBounds.selector
++  );
    // If borrower is reducing the interest rate, increase the reserve
    // ratio for the next two weeks.
    if (annualInterestBips < WildcatMarket(market).annualInterestBips()) {
      TemporaryReserveRatio storage tmp = temporaryExcessReserveRatio[market];

      if (tmp.expiry == 0) {
        tmp.reserveRatioBips = uint128(WildcatMarket(market).reserveRatioBips());

        // Require 90% liquidity coverage for the next 2 weeks
        WildcatMarket(market).setReserveRatioBips(9000);
      }

      tmp.expiry = uint128(block.timestamp + 2 weeks);
    }

    WildcatMarket(market).setAnnualInterestBips(annualInterestBips);
  }
```


## Assessed type

Invalid Validation