# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/156
# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L219
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L81


# Vulnerability details

## Impact

The quest participants can mint receipts of a certain quest once they completed on-chain tasks. Also the quest creators put the [endTime](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L77) of the quest in the quest creation.

The quest creator uses the endTime because they are allow to pay rewards for those on-chain completed tasks for a certain period of time. If the Rabbithole protocol allows participants to mint receipts even after the ```Quest.endTime``` then the quest creator will lost rewards tokens they don't wanted to lost.

The quest creator can [withdraw the remaining tokens](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L81) after the quest ends but if the protocol allows mint receipts even if the quest has ended then the quest creator will withdraw less tokens. The quest creator uses an endTime parameter because it is the range time they want to pay rewards for completed on-chain tasks, so mint receipts after the endTime will affect the quest creators rewards.

There is an [off-chain protection](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L222-L223) because the participant needs a valid sig hash in order to mint the receipt but there should be a on-chain validation in order to protect the quest creators rewards.

## Proof of Concept

As you can see in the ```mintReceipt``` function, the participant can mint receipts even if the quest has ended.

```solidity
File: QuestFactory.sol
219:     function mintReceipt(string memory questId_, bytes32 hash_, bytes memory signature_) public {
220:         if (quests[questId_].numberMinted + 1 > quests[questId_].totalParticipants) revert OverMaxAllowedToMint();
221:         if (quests[questId_].addressMinted[msg.sender] == true) revert AddressAlreadyMinted();
222:         if (keccak256(abi.encodePacked(msg.sender, questId_)) != hash_) revert InvalidHash();
223:         if (recoverSigner(hash_, signature_) != claimSignerAddress) revert AddressNotSigned();
224: 
225:         quests[questId_].addressMinted[msg.sender] = true;
226:         quests[questId_].numberMinted++;
227:         emit ReceiptMinted(msg.sender, questId_);
228:         rabbitholeReceiptContract.mint(msg.sender, questId_);
229:     }

```

## Tools used

VSCode

## Recommended Mitigation Steps

Add a validation in the ```mintReceipt()``` function where it is only possible to mint a receipt if the ```block.timestamp``` is in the ```Quest.startTime/Quest.endTime``` range.