# Original link
https://github.com/code-423n4/2023-10-wildcat-findings/issues/196
# Lines of code

https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatMarketController.sol#L468


# Vulnerability details

When a `WildcatMarketController` is created, it is hardcoded with a `MinimumAnnualInterestBips` and `MaximumAnnualInterestBips` ramge that cannot be changed; these values come from `WildcatMarketControllerFactory` where they are specified by the protocol owners.

When a market is created at the borrower's request, the annual interest requested by the borrower is validated to sit within these bounds.

However, the borrower is also allowed to change this value at a later point through the `WildcatMarketController.setAnnualInterestBips` function. This entry point does not offer any validation, except for the downstream `WildcatMarket.setAnnualInterestBips` that checks for the value not to exceed `BIPS`.

## Impact
After the creation of a market, the borrower is allowed to change its annual interest outside the bounds allowed by the protocol.

## Proof of Concept
```Solidity
    function testArbitraryInterestRate() public {
        WildcatArchController archController = new WildcatArchController();
        SanctionsList sanctionsList = new SanctionsList();
        WildcatSanctionsSentinel sentinel = new WildcatSanctionsSentinel(
            address(archController), 
            address(sanctionsList));


        // the protocol mandates a 10% annual interest
        WildcatMarketControllerFactory controllerFactory = new WildcatMarketControllerFactory(
            address(archController), // _archController,
            address(sentinel), // _sentinel,
            MarketParameterConstraints( // constraints
                uint32(0), // minimumDelinquencyGracePeriod;
                uint32(0), // maximumDelinquencyGracePeriod;
                uint16(0), // minimumReserveRatioBips;
                uint16(0), // maximumReserveRatioBips;
                uint16(0), // minimumDelinquencyFeeBips;
                uint16(0), // maximumDelinquencyFeeBips;
                uint32(0), // minimumWithdrawalBatchDuration;
                uint32(0), // maximumWithdrawalBatchDuration;
                uint16(10_00), // minimumAnnualInterestBips;
                uint16(10_00)  // maximumAnnualInterestBips;
            )
        );

        address borrower = address(uint160(uint256(keccak256("borrower"))));

        archController.registerBorrower(borrower);
        archController.registerControllerFactory(address(controllerFactory));

        MockERC20 token = new MockERC20();

        vm.startPrank(borrower);
        WildcatMarketController controller = WildcatMarketController(
            controllerFactory.deployController());

        // the borrower creates a market with 10% annual interest - all good till now
        address market = controller.deployMarket(
            address(token), // asset,
            "3d", // namePrefix,
            "3", // symbolPrefix,
            type(uint128).max, // maxTotalSupply,
            10_00, // annualInterestBips
            0, // delinquencyFeeBips,
            0, // withdrawalBatchDuration,
            0, // reserveRatioBips,
            0 // delinquencyGracePeriod
        );

        // now, the borrower is allowed to change the interest below 10%
        // which is not allowed by the protocol config
        controller.setAnnualInterestBips(market, 5_00);

        // and get away with it
        assertEq(5_00, WildcatMarket(market).annualInterestBips());
    }
```

## Tools Used
Code review, Foundry

## Recommended Mitigation Steps
Consider introducing the validation of the new interest rate:
```diff
  // WildcatMarketController.sol:468
  function setAnnualInterestBips(
    address market,
    uint16 annualInterestBips
  ) external virtual onlyBorrower onlyControlledMarket(market) {
+   assertValueInRange(
+     annualInterestBips,
+     MinimumAnnualInterestBips,
+     MaximumAnnualInterestBips,
+     AnnualInterestBipsOutOfBounds.selector
+   );
    // If borrower is reducing the interest rate, increase the reserve
    // ratio for the next two weeks.
```


## Assessed type

Invalid Validation