# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/150
# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L99
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L109


# Vulnerability details

## Impact

The RabbitHole participant can have many receipts from all the quest he has participated. The [RabbitHoleReceipt.sol::getOwnedTokenIdsOfQuest()](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L109) function helps to get the receipts which are owned by the user per questId.

If the participant accumulate a lot of RabbitHoleReceipts the [for statement](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L117) which iterates through all receipts from the participant can run out of gas.

The [Quest.sol::claim()](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L99) function will be reverted for users who have many receipts causing the rewards may be trapped for the participants and the quest creator because the [ERC20Quest.sol::withdrawRemainingTokens()](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L81) is only able to withdraw the non claimable tokens.

## Proof of Concept

The ```RabbitHoleReceipt.sol::getOwnedTokenIdsOfQuest()``` function calculates the user balance in the line 113. Then the user balance is used in the for statement in the line 117.

The participant balance could be a large amount of receipts because the user can participate in many quests then the ```for``` statement could be reverted by insufficient gas.

```solidity
File: RabbitHoleReceipt.sol
109:     function getOwnedTokenIdsOfQuest(
110:         string memory questId_,
111:         address claimingAddress_
112:     ) public view returns (uint[] memory) {
113:         uint msgSenderBalance = balanceOf(claimingAddress_);
114:         uint[] memory tokenIdsForQuest = new uint[](msgSenderBalance);
115:         uint foundTokens = 0;
116: 
117:         for (uint i = 0; i < msgSenderBalance; i++) {
118:             uint tokenId = tokenOfOwnerByIndex(claimingAddress_, i);
119:             if (keccak256(bytes(questIdForTokenId[tokenId])) == keccak256(bytes(questId_))) {
120:                 tokenIdsForQuest[i] = tokenId;
121:                 foundTokens++;
122:             }
123:         }
```

I created a basic test where you can see the receipt is not burned after the rewards claim, then the participant is accumulating many receipts:

```javascript
it('the receipt is not burned after the reward was claimed', async () => {
    // Receipts are not burned after the rewards claim()
    // 1. Mint a new Receipt for the firstAddress
    await deployedRabbitholeReceiptContract.mint(firstAddress.address, questId)
    // 2. Check the firstAddress receipt balance
    expect(await deployedRabbitholeReceiptContract.balanceOf(firstAddress.address)).to.equal(1)
    await deployedQuestContract.start()
    await ethers.provider.send('evm_increaseTime', [86400])
    // 3. Claim the rewards as the firstAddress
    await deployedQuestContract.connect(firstAddress).claim()
    // 4. After the rewards was claimed, the receipt is not burned then the user is accumulating many receipts
    expect(await deployedRabbitholeReceiptContract.balanceOf(firstAddress.address)).to.equal(1)
})
```

## Tools used

VSCode

## Recommended Mitigation Steps

If the rewards was claimed then burn the participant receipt.