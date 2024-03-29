# Original link
https://github.com/sherlock-audit/2023-09-nounsbuilder-judging/issues/5
0xbepresent

high

# The `Token::updateFounders()` func does not remove the previous founders, which leads to them being able to claim tokens from the DAO

## Summary

The `Token::updateFounders()` function does not remove the previous founders, which leads to them being able to claim tokens.
## Vulnerability Detail

Founders are added to the DAO's founders list during the deployment of the DAO by using the [Token::_addFounders()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L92) function. As a result, the founder can obtain a token from the DAO. The `Token::_addFounders()` func starts from the [reservedUntilTokenId](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L161) number in order to assign it to the corresponding `tokenRecipient` array in the [code line 169](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L169):

```solidity
File: Token.sol
160:                 // Used to store the base token id the founder will recieve
161:                 uint256 baseTokenId = reservedUntilTokenId;
162: 
163:                 // For each token to vest:
164:                 for (uint256 j; j < founderPct; ++j) {
165:                     // Get the available token id
166:                     baseTokenId = _getNextTokenId(baseTokenId);
167: 
168:                     // Store the founder as the recipient
169:                     tokenRecipient[baseTokenId] = newFounder;
170: 
171:                     emit MintScheduled(baseTokenId, founderId, newFounder);
172: 
173:                     // Update the base token id
174:                     baseTokenId = (baseTokenId + schedule) % 100;
175:                 }
176:             }
```

In the other hand, the owner can update the founders list using the [Token::updateFounders()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L375) func. The function will remove the previous assigned founders in the [code line 420](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L420):

```solidity
File: Token.sol
411:                 // Used to reverse engineer the indices the founder has reserved tokens in.
412:                 uint256 baseTokenId;
413: 
414:                 for (uint256 j; j < cachedFounder.ownershipPct; ++j) {
415:                     // Get the next index that hasn't already been cleared
416:                     while (clearedTokenIds[baseTokenId] != false) {
417:                         baseTokenId = (++baseTokenId) % 100;
418:                     }
419: 
420:                     delete tokenRecipient[baseTokenId];
421:                     clearedTokenIds[baseTokenId] = true;
422: 
423:                     emit MintUnscheduled(baseTokenId, i, cachedFounder);
424: 
425:                     // Update the base token id
426:                     baseTokenId = (baseTokenId + schedule) % 100;
427:                 }
428:             }
```

A problem arises because the function starts from the [baseTokenId zero](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L412), an action that is incorrect because it does not consider the `reservedUntilTokenId` in the [code line 412](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L412). As a result, previous founders remain on the list, enabling them to claim a token from the DAO.

## Impact

The [Token::updateFounders()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L375C14-L375C28) does not work as intended, it attempts to remove the previous founders from the `tokenRecipient[]` array but fails, leaving previous founders able to claim a token from the DAO.

I created a test where the DAO is deployed using a list of two founders and a `reservedUntilTokenId=10`, then the owner updates the founders using another list of founders but the previous founders still are in the `tokenRecipient[]` array.

```solidity
// File: Token.t.sol
// $ forge test --match-test "test_UpdateFundersDoesNotUseTheReservedUntilTokenId" -vvv
//
    function test_UpdateFundersDoesNotUseTheReservedUntilTokenId() public {
        //
        // The previous founders are not removed once the owner calls updateFounders() using another founders list causing that
        // the previous founders are able to claim a token
        createUsers(2, 1 ether);
        address[] memory wallets = new address[](2);
        uint256[] memory percents = new uint256[](2);
        uint256[] memory vestExpirys = new uint256[](2);
        unchecked {
            for (uint256 i; i < 2; ++i) {
                wallets[i] = otherUsers[i];
                percents[i] = 1;
                vestExpirys[i] = 4 weeks;
            }
        }
        deployWithCustomFoundersAndCustomReserve(wallets, percents, vestExpirys, 10);
        //
        // assert totalFounders is 1
        assertEq(token.totalFounders(), 2);
        assertEq(token.totalFounderOwnership(), 2);
        //
        // The scheduledRecipient starts in number `10` because the token was deployed using reservedUntilTokenId = 10
        assertEq(token.getScheduledRecipient(10).wallet, otherUsers[0]);
        assertEq(token.getScheduledRecipient(11).wallet, otherUsers[1]);
        //
        // The founders are updated but the previous founders are not removed
        IManager.FounderParams[] memory newFoundersArr = new IManager.FounderParams[](1);
        newFoundersArr[0] = IManager.FounderParams({
            wallet: address(0x06B59d0b6AdCc6A5Dc63553782750dc0b41266a3),
            ownershipPct: 10,
            vestExpiry: 2556057600
        });
        vm.prank(address(wallets[0]));
        token.updateFounders(newFoundersArr);
        //
        // Assert the previous founder still has the baseTokenId 10 and 11 which is incorrect
        // because the previous founders list should had been removed
        assertEq(token.getScheduledRecipient(10).wallet, otherUsers[0]);
        assertEq(token.getScheduledRecipient(11).wallet, otherUsers[1]);
        assertEq(token.getScheduledRecipient(12).wallet, address(0x06B59d0b6AdCc6A5Dc63553782750dc0b41266a3));
    }
```

Additionally, the next funcion is added to the `NounsBuilderTest.sol` file:

```diff
+++ b/nouns-protocol/test/utils/NounsBuilderTest.sol
@@ -274,6 +274,25 @@ contract NounsBuilderTest is Test {
         setMockMetadata();
     }
 
+    function deployWithCustomFoundersAndCustomReserve(
+        address[] memory _wallets,
+        uint256[] memory _percents,
+        uint256[] memory _vestExpirys,
+        uint256 _reservedUntilTokenId
+    ) internal virtual {
+        setFounderParams(_wallets, _percents, _vestExpirys);
+
+        setMockTokenParamsWithReserve(_reservedUntilTokenId);
+
+        setMockAuctionParams();
+
+        setMockGovParams();
+
+        deploy(foundersArr, tokenParams, auctionParams, govParams);
+
+        setMockMetadata();
+    }
+
```

## Code Snippet

- [Token::_addFounders()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L92)
- [Token::updateFounders()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L375)


## Tool used

Manual review

## Recommendation

Please, ensure the usage of `reservedUntilTokenId` in the [Token::updateFounders()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L412) function:

```diff
File: Token.sol
410: 
411:                 // Used to reverse engineer the indices the founder has reserved tokens in.
--                   uint256 baseTokenId;
++                   uint256 baseTokenId = reservedUntilTokenId;
413: 
414:                 for (uint256 j; j < cachedFounder.ownershipPct; ++j) {
415:                     // Get the next index that hasn't already been cleared
416:                     while (clearedTokenIds[baseTokenId] != false) {
417:                         baseTokenId = (++baseTokenId) % 100;
418:                     }
419: 
420:                     delete tokenRecipient[baseTokenId];
421:                     clearedTokenIds[baseTokenId] = true;
422: 
423:                     emit MintUnscheduled(baseTokenId, i, cachedFounder);
424: 
425:                     // Update the base token id
426:                     baseTokenId = (baseTokenId + schedule) % 100;
427:                 }
428:             }
```

Moreover, adapt the necessary procedures if the `reservedUntilTokenId` changes via the [Token::setReservedUntilTokenId()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L486C14-L486C37) function.

Duplicate of #42
