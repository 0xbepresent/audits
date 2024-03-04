# Original link
https://github.com/code-423n4/2023-10-wildcat-findings/issues/164
# Lines of code

https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatSanctionsEscrow.sol#L33
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketWithdrawals.sol#L166
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L172
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatSanctionsSentinel.sol#L95


# Vulnerability details

# Once the sanctioned lender is being unsanctioned, the lender assets are transferred to the borrower instead of the lender, causing the lost of lender's assets

## Lines of code
- https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatSanctionsEscrow.sol#L33
- https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketWithdrawals.sol#L166
- https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L172
- https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatSanctionsSentinel.sol#L95

## Impact

If the lender is sanctioned while there is a withdrawal process, the assets are [transferred to the escrow contract](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketWithdrawals.sol#L166-L171) in the code line 166-171:

```solidity
File: WildcatMarketWithdrawals.sol
137:   function executeWithdrawal(
138:     address accountAddress,
139:     uint32 expiry
140:   ) external nonReentrant returns (uint256) {
...
...
163: 
164:     if (IWildcatSanctionsSentinel(sentinel).isSanctioned(borrower, accountAddress)) {
165:       _blockAccount(state, accountAddress);
166:       address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(
167:         accountAddress,
168:         borrower,
169:         address(asset)
170:       );
171:       asset.safeTransfer(escrow, normalizedAmountWithdrawn);
...
...
188:   }
```

The assets can be claimed using the [WildcatSanctionsEscrow::releaseEscrow()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatSanctionsEscrow.sol#L33) function once the sanctioned lender is being unsanctioned. The problem is that the parameters are not correctly ordered when the ```createEscrow()``` function is called. 

The escrow contract is [called](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketWithdrawals.sol#L166-L171) using the parameters order ```createEscrow(accountAddress, borrower, address(asset))```:

```solidity
File: WildcatMarketWithdrawals.sol
166:       address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(
167:         accountAddress,
168:         borrower,
169:         address(asset)
170:       );
```

But the [WildcatSanctionsSentinel::createEscrow()](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/WildcatSanctionsSentinel.sol#L95) function has a different parameters order ```borrower, account, asset```: 

```solidity
File: WildcatSanctionsSentinel.sol
95:   function createEscrow(
96:     address borrower,
97:     address account,
98:     address asset
99:   ) public override returns (address escrowContract) {
```

The account and borrower parameters are inversed. This behaivour causes that once the sanctioned lender is being unsanctioned, the assets will be transferred to the borrower instead the lender. Lender's assets will be lost.

## Proof of Concept

I created a test where Alice (lender) is sanctioned and his assets are transferred to the escrow contract. Then Alice is being unsanctioned and Alice assets are transferred to the borrower instead Alice (lender):

```solidity
import { IWildcatSanctionsEscrow } from '../../src/interfaces/IWildcatSanctionsEscrow.sol';
//
// File: test/market/WildcatMarketWithdrawals.t.sol:WithdrawalsTest
// $ forge test --match-test "test_executeWithdrawal_Sanctioned_ParamsAreNotCorrectlyOrdered"
//
  function test_executeWithdrawal_Sanctioned_ParamsAreNotCorrectlyOrdered() external {
    // Once the sanctioned lender is being unsanctioned, the lender balance is transferred to the borrower instead of the lender.
    // The problem is that the function params are not correctly ordered.
    // 
    _deposit(alice, 1e18);
    _requestWithdrawal(alice, 1e18);
    fastForward(parameters.withdrawalBatchDuration);
    uint256 previousBalanceAlice = asset.balanceOf(alice);
    uint256 previousBalanceBorrower = asset.balanceOf(borrower);
    //
    // 1. Alice is sanctioned and his balance is transferred to the escrow contract
    sanctionsSentinel.sanction(alice);
    address escrow = sanctionsSentinel.getEscrowAddress(alice, borrower, address(asset));
    market.executeWithdrawal(alice, uint32(block.timestamp));
    assertEq(IWildcatSanctionsEscrow(escrow).balance(), 1e18);
    //
    // 2. Alice is unsanctioned and Alice is able to call releaseEscrow() but assets are transferred
    // to the borrower instead alice
    sanctionsSentinel.unsanction(alice);
    IWildcatSanctionsEscrow(escrow).releaseEscrow();
    assertEq(previousBalanceAlice, asset.balanceOf(alice));
    assertEq(previousBalanceBorrower + 1e18, asset.balanceOf(borrower));
    assertEq(IWildcatSanctionsEscrow(escrow).balance(), 0);
  }
```

I added the next function to the ```test/helpers/MockSanctionsSentinel.sol``` file:

```diff
--- a/test/helpers/MockSanctionsSentinel.sol
+++ b/test/helpers/MockSanctionsSentinel.sol
@@ -14,4 +14,8 @@ contract MockSanctionsSentinel is WildcatSanctionsSentinel {
   function sanction(address account) external {
     MockChainalysis(chainalysisSanctionsList).sanction(account);
   }
+
+  function unsanction(address account) external {
+    MockChainalysis(chainalysisSanctionsList).unsanction(account);
+  }
 }
```

## Tools used

Manual review

## Recommended Mitigation Steps

Please correct the order of the parameters in the [WildcatMarketWithdrawals::executeWithdrawal()#166](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketWithdrawals.sol#L166-L171) and [WildcatMarketBase::_blockAccount()#172](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L172)


## Assessed type

Token-Transfer