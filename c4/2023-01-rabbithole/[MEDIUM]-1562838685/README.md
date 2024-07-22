# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/552
# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L99
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L117-L133


# Vulnerability details

# DOS risk if enough tokens are minted in Quest.claim
## Description
```claim``` function can be summaraized in next steps:
1. Check that the quest is active
2. Check the contract is not paused
3. Get tokens corresponding to msg.sender for ```questId``` using ```rabbitHoleReceiptContract.getOwnedTokenIdsOfQuest```: **DOS**
4. Check that msg.sender owns at least one token
5. Count non claimed tokens
6. Check there is at least 1 unclaimed token
7. Calculate redeemable rewards: ```_calculateRewards(redeemableTokenCount);```
8. Set all token to claimed state
9. Update ```redeemedTokens```
10. Emit claim event

The problem with this functions relays in its dependency on ```RabbitHoleReceipt.getOwnedTokenIdsOfQuest```. It's behaviour can be summarized in next steps:
1. Get queried balance (claimingAddress_)
2. Get claimingAddress_ owned tokens 
3. Filter tokens corresponding to questId_
4. Return token of claimingAddress_ corresponding to questId_

If a user takes part in many quests and gets lot's of tokens, the claim function will eventually reach block gas limit. Therefore, it will be unable to claim their tokens.

If a user actively participates in multiple quests and accumulates a large number of tokens, the claim function may eventually reach the block gas limit. As a result, the user may be unable to successfully claim their earned tokens.

## Impact
It can be argued that function ```ERC721.burn``` can address the potential DOS risk in the claim process. However, it is important to note the following limitations and drawbacks associated with this approach:
1. Utilizing ```ERC721.burn``` does not prevent the user from incurring network fees if a griefer, who has already claimed their rewards, sends their tokens to the user with the intent of causing a DOS and inducing loss of gas.
1. If the user has not claimed any rewards from their accumulated tokens, they will still be forced to burn at least some of their tokens, resulting in a loss of these assets.


## POC
### Griefing
1. Alice has took part in many quests, and want to recieve her rewards, so she call Quest.claim() function
2. Bob also has already claimed many rewards from many quest, and decide to frontrun alice an send her all his tokens to DOS her
3. Alice run out of gas, she lose transaction fees.

### Lose of unclaimed rewards
1. Alice always takes part in many quest, but never claim her rewards. She trust RabbitHole protocol and is waiting to have much more rewards to claim in order to save some transaction fees
2. When Alice decide to call claim function she realize that she has run out of gas.

Then, Alice can only burn her some of her tokens to claim at least some rewards.

### Code
[Code sample](https://gist.github.com/carlitox477/85e37d26c6f810304c849c93235ee99e)

## Mittigation steps
If a user can sent a token list by parameter to claim function, then this vector attack can be mitigated.

To do this add next function to ```RabbitHoleReceipt.sol```:
```solidity
function checkTokenCorrespondToQuest(uint tokenId, string memory questId_) external view returns(bool){
    return keccak256(bytes(questIdForTokenId[tokenId])) == keccak256(bytes(questId_));
}
```

Then modify ```Quest.claim```: 

```diff
// Quest.sol
-   function claim() public virtual onlyQuestActive {
+   function claim(uint[] memory tokens) public virtual onlyQuestActive {
        if (isPaused) revert QuestPaused();

-       uint[] memory tokens = rabbitHoleReceiptContract.getOwnedTokenIdsOfQuest(questId, msg.sender);

        // require(tokens.length > 0)
        if (tokens.length == 0) revert NoTokensToClaim();

        uint256 redeemableTokenCount = 0;
        for (uint i = 0; i < tokens.length; i++) {
            // Check that the token correspond to this quest
            require(rabbitHoleReceiptContract.checkTokenCorrespondToQuest(tokens[i],questId))

-           if (!isClaimed(tokens[i])) {
+           if (!isClaimed(tokens[i]) && rabbitHoleReceiptContract.checkTokenCorrespondToQuest(tokens[i],questId)) {
                redeemableTokenCount++;
            }
        }

        if (redeemableTokenCount == 0) revert AlreadyClaimed();

        uint256 totalRedeemableRewards = _calculateRewards(redeemableTokenCount);
        _setClaimed(tokens);
        _transferRewards(totalRedeemableRewards);
        redeemedTokens += redeemableTokenCount;

        emit Claimed(msg.sender, totalRedeemableRewards);
    }