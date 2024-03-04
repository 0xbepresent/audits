# Original link
https://github.com/code-423n4/2023-10-wildcat-findings/issues/238
# Lines of code

https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketToken.sol#L64
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatSanctionsEscrow.sol#L33
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketConfig.sol#L74


# Vulnerability details

## Impact

The user can be marked as [blocked](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L167) using the [WildcatMarketConfig::nukeFromOrbit()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketConfig.sol#L74C12-L74C25) function, then funds will be sent to the [escrow contract](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L178). The sanctioned user needs to be being unsanctioned in order [to be able to claim the assets](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatSanctionsEscrow.sol#L34).

The problem is that the lender who is going to be marked as `blocked` can transfer his assets to another collude lender, avoiding the funds to be sent to the escrow contract therefore avoiding the assets to be [locked until the user is `unsanctioned`](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatSanctionsEscrow.sol#L34). Consider the next scenario:

1. `LenderA` is sanctioned and the [WildcatMarketConfig::nukeFromOrbit()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketConfig.sol#L74C12-L74C25) function is called.
2. `LenderA` sees the transaction and he [transfers](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketToken.sol#L64) his assets to a collude lender before the execution of the step 1 (nukeFromOrbit()).
3. Now `LenderA` is [blocked](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L167) and nothing is sent to the [escrow contract](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L178).
4. The colluded lender can help to withdraw the `LenderA` funds from the market.

## Proof of Concept

I created a test where Alice (Lender) transfer his funds to the collude lender (Bob) before the [WildcatMarketConfig::nukeFromOrbit()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketConfig.sol#L74C12-L74C25) function is called, therefore the lender Bob can get the Alice assets (sanctioned user). Test steps:

1. Lender Alice deposit 1e18
2. Alice is sanctioned and she [transfer](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketToken.sol#L64) his assets to another collude lender (Bob) before the [nukeFromOrbit()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketConfig.sol#L74C12-L74C25) is called.
3. The `nukeFromOrbit()` is called and the [escrow contract is not created](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L170) since the sanctioned lender Alice has not funds because she transferred the assets in the step 2.
4. The collude lender (bob) can execute the withdrawal and he gets the alice's assets.

```solidity
// File: test/market/WildcatMarketWithdrawals.t.sol:WithdrawalsTest
// $ forge test --match-test "test_SanctionedUser_transferAssetsToAnotherLender" -vvv
//
  function test_SanctionedUser_transferAssetsToAnotherLender() external {
    // Lender, who is going to be blocked (sanctioned) can transfer his assets to a collude lender in order to avoid the 
    // escrow restrictions.
    //
    _authorizeLender(bob);
    uint256 previousBalanceBob = asset.balanceOf(bob);
    // 
    // 1. Lender Alice deposit 1e18
    _deposit(alice, 1e18);
    assertEq(market.scaledBalanceOf(alice), 1e18);
    assertEq(market.scaledBalanceOf(bob), 0);
    //
    // 2. Alice is sanctioned and she transfer his assets to another collude lender (Bob)
    // before the nukeFromOrbit() is called.
    sanctionsSentinel.sanction(alice);
    vm.prank(alice);
    market.transfer(bob, 1e18);
    assertEq(market.scaledBalanceOf(alice), 0);
    assertEq(market.scaledBalanceOf(bob), 1e18);
    //
    // 3. The nukeFromOrbit() is called and the escrow contract is not created.
    market.nukeFromOrbit(alice);
    //
    // 4. The collude lender (bob) can execute the withdrawal and he gets the alice's assets
    _requestWithdrawal(bob, 1e18);
    fastForward(parameters.withdrawalBatchDuration);
    market.executeWithdrawal(bob, uint32(block.timestamp));
    assertEq(market.scaledBalanceOf(alice), 0);
    assertEq(market.scaledBalanceOf(bob), 0);
    assertEq(previousBalanceBob + 1e18, asset.balanceOf(bob));  // Bob has the alice's assets
  }
```

## Tools used

Manual review

## Recommended Mitigation Steps

Validates in the transfer function if the `from` address is sanctioned:

```diff
  function _transfer(address from, address to, uint256 amount) internal virtual {
    MarketState memory state = _getUpdatedState();
    uint104 scaledAmount = state.scaleAmount(amount).toUint104();

    if (scaledAmount == 0) {
      revert NullTransferAmount();
    }

++  if (IWildcatSanctionsSentinel(sentinel).isSanctioned(borrower, from)) {
++    revert TheFromAddressIsSactioned();
++  }

    Account memory fromAccount = _getAccount(from);
    fromAccount.scaledBalance -= scaledAmount;
    _accounts[from] = fromAccount;

    Account memory toAccount = _getAccount(to);
    toAccount.scaledBalance += scaledAmount;
    _accounts[to] = toAccount;

    _writeState(state);
    emit Transfer(from, to, amount);
  }
```


## Assessed type

Token-Transfer