# Original link
https://github.com/sherlock-audit/2023-09-nounsbuilder-judging/issues/77
0xbepresent

high

# Founders won't be able to claim some tokens if the `Token` is initialized using a non-zero `reservedUntilTokenId`

## Summary

Founders won't be able to claim some tokens if the Token is initialized using a non-zero `reservedUntilTokenId`

## Vulnerability Detail

When the token is initialized, it is possible to set a non-zero `reservedUntilTokenId` in order to reserve some tokens, so the [Token::_addFounders()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L120) function will start from the `reservedUntilTokenId` (code line 161) to assign the claimable token to the founders:

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

The problem arises when a new `baseTokenId` is calculated in the [code line 174](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L174) as it does not factor in the `reservedUntilTokenId`, resulting in the founder being assigned to a reserved id. Please refer to next scenario:

1. The token is initialized using `reservedUntilTokenId = 90` and a founder is assigned `10 as ownership`.
2. The `baseTokenId` will be 90 in the [code line 161](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L161) because `reservedUntilTokenId` is 90.
3. The first assigned position to the founder will be the number `90` because `_getNextTokenId(baseTokenId) = 90` (code line [166](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L166)) then `tokenRecipient[90] = newFounder` (code line [169](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L169)).
4. Subsequently, the next `baseTokenId` calculation is `baseTokenId = (90 + 10) % 100 = 0` (code line [174](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L174)), resulting in the next founder assigned position being `tokenRecipient[0] = newFounder`.
5. Subsequently, the next founder assigned position will be `tokenRecipient[10] = newFounder` and so on until `tokenRecipient[80] = newFounder`.

The founder will only be able to claim one token, the `tokenRecipient[90]` because the [tokenID starts at the position `90`](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L235). The IDs from `0 to 89` are reserved to the [reserve](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L213) so the founder won't be able to claim in that range.

## Impact

Founders will not be able to claim assigned tokens to them if the `Token` is initialized using a non-zero `reservedUntilTokenId`. I created a test where the token is initialized with `reservedUntilTokenId=90` and one founder with 10 ownership, then the founder is assigned the IDs 0, 10, 20, ... , 90 but he will only able to claim only **one token** because the others IDs are reserved.

```solidity
// File: Token.t.sol
// $ forge test --match-test "test_FounderWillBeAbleToClaimTokensFromTheReserve" -vvv
//
    function test_FounderWontBeAbleToClaimTokensIfTheTokenIsInitializedUsingAReservedUntilTokenId() public {
        // Founders won't be able to claim assigned ownership tokens if the token is initialized using
        // non-zero reservedUntilTokenId
        //
        // 1. Token is deployed and a founder is assigned with owner ship of 10 and reservedUntilTokenId = 90.
        createUsers(1, 1 ether);
        address[] memory wallets = new address[](1);
        uint256[] memory percents = new uint256[](1);
        uint256[] memory vestExpirys = new uint256[](1);
        address founder = otherUsers[0];
        wallets[0] = founder;
        percents[0] = 10;
        vestExpirys[0] = 4 weeks;
        deployWithCustomFoundersAndCustomReserve(wallets, percents, vestExpirys, 90);
        //
        // 2. Assert the scheduledRecipient and the founder is assigned the position 0, 10, 20, 30 ... 90
        // because the token was deployed using reservedUntilTokenId = 90.
        // That's is a problem because ids from 0 - 89 are reserved and the founder won't be able to claim in that range.
        assertEq(token.totalFounders(), 1);
        assertEq(token.totalFounderOwnership(), 10);
        assertEq(token.getScheduledRecipient(90).wallet, founder);
        assertEq(token.getScheduledRecipient(0).wallet, founder);
        assertEq(token.getScheduledRecipient(10).wallet, founder);
        assertEq(token.getScheduledRecipient(20).wallet, founder);
        assertEq(token.getScheduledRecipient(30).wallet, founder);
        assertEq(token.getScheduledRecipient(40).wallet, founder);
        assertEq(token.getScheduledRecipient(50).wallet, founder);
        assertEq(token.getScheduledRecipient(60).wallet, founder);
        assertEq(token.getScheduledRecipient(70).wallet, founder);
        assertEq(token.getScheduledRecipient(80).wallet, founder);
        //
        // 3. Auction starts and the founder is only able to claim one token because the tokenId starts at number 90
        // As a result, the previous assigned tokens to the founder will be unclaimable.
        vm.prank(address(auction));
        token.mint();
        assertEq(token.balanceOf(founder), 1);
    }
```

Additionally, the next function is added to the `NounsBuilderTest.sol` file:

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

- [Token::_addFounders()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L120)
- [Token::_getNextTokenId()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L186C14-L186C29)
- [Token::_isForFounder()](https://github.com/sherlock-audit/2023-09-nounsbuilder/blob/main/nouns-protocol/src/token/Token.sol#L263)


## Tool used

Manual review

## Recommendation

The `_addFounders()`, `_getNextTokenId()` and `_isForFounder()` functions should consider the `reservedUntilTokenId` value in order to assign tokens to the founders:

```diff
    function _addFounders(IManager.FounderParams[] calldata _founders, uint256 reservedUntilTokenId) internal {
        ...
        ...
        ..

        unchecked {
            // For each founder:
            for (uint256 i; i < _founders.length; ++i) {
                ...
                ...
                ...

                // Used to store the base token id the founder will recieve
                uint256 baseTokenId = reservedUntilTokenId;

                // For each token to vest:
                for (uint256 j; j < founderPct; ++j) {
                    // Get the available token id
                    baseTokenId = _getNextTokenId(baseTokenId);

                    // Store the founder as the recipient
                    tokenRecipient[baseTokenId] = newFounder;

                    emit MintScheduled(baseTokenId, founderId, newFounder);

                    // Update the base token id
--                  baseTokenId = (baseTokenId + schedule) % 100;
++                  baseTokenId = (baseTokenId + schedule) % (100 + reservedUntilTokenId);
                }
            }
            ...
            ...
            ...
        }
    }

    function _getNextTokenId(uint256 _tokenId) internal view returns (uint256) {
        unchecked {
            while (tokenRecipient[_tokenId].wallet != address(0)) {
--              _tokenId = (++_tokenId) % 100;
++              _tokenId = (++_tokenId) % (100 + reservedUntilTokenId);
            }

            return _tokenId;
        }
    }

    function _isForFounder(uint256 _tokenId) private returns (bool) {
        // Get the base token id
--      uint256 baseTokenId = _tokenId % 100;
++      uint256 baseTokenId = _tokenId % (100 + reservedUntilTokenId);

        // If there is no scheduled recipient:
        if (tokenRecipient[baseTokenId].wallet == address(0)) {
            return false;

            // Else if the founder is still vesting:
        } else if (block.timestamp < tokenRecipient[baseTokenId].vestExpiry) {
            // Mint the token to the founder
            _mint(tokenRecipient[baseTokenId].wallet, _tokenId);

            return true;

            // Else the founder has finished vesting:
        } else {
            // Remove them from future lookups
            delete tokenRecipient[baseTokenId];

            return false;
        }
    }
```

Duplicate of #42
