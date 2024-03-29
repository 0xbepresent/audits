# Original link
https://github.com/sherlock-audit/2023-09-Gitcoin-judging/issues/377
0xbepresent

high

# The `QVSimpleStrategy.maxVoiceCreditsPerAllocator` can be evaded by the allocator causing that he can allocate infinite credits to the same recipient

The allocator can evade the `QVSimpleStrategy.maxVoiceCreditsPerAllocator` limit causing that the allocator can allocate infinite credits to the same or multiple recipients.

## Vulnerability Detail

The allocator can assign infinite credits to the same recipient using the [QVSimpleStrategy::_allocate()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-simple/QVSimpleStrategy.sol#L107) function evading the [_hasVoiceCreditsLeft()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-simple/QVSimpleStrategy.sol#L121) function. The problem is that [allocator.voiceCredits](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-simple/QVSimpleStrategy.sol#L121C59-L121C81) does not increment when the credits are assigned so the `allocator.voiceCredits` always will be zero and the [_hasVoiceCreditsLeft()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-simple/QVSimpleStrategy.sol#L144) can be evaded.

Then the `recipient` can get a bigger payout because the [recipient.totalVotesReceived](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L527) can be increased without any restriction by the `allocator`. The function `_getPayout()` in the [code line 571](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L571C45-L571C63) the amount to pay is calculated based on the `recipient.TotalVotesReceived`:

```solidity
File: QVBaseStrategy.sol
559:     function _getPayout(address _recipientId, bytes memory)
560:         internal
561:         view
562:         virtual
563:         override
564:         returns (PayoutSummary memory)
565:     {
566:         Recipient memory recipient = recipients[_recipientId];
567: 
568:         // Calculate the payout amount based on the percentage of total votes
569:         uint256 amount;
570:         if (!paidOut[_recipientId] && totalRecipientVotes != 0) {
571:             amount = poolAmount * recipient.totalVotesReceived / totalRecipientVotes;
572:         }
573:         return PayoutSummary(recipient.recipientAddress, amount);
574:     }
```

I created a test where the same allocator can assign more than `100 credits` (the current `maxVoiceCreditsPerAllocator` is 100). That is incorrect because the same `allocator` can assign multiple credits to the same `recipient` causing that the recipient receives a bigger payout because he receives more votes. In the next test `recipientId` receives `180 credits` by the same allocator which is incorrect because the max limit per allocator is `100`:

```solidity
// File: test/foundry/strategies/QVSimpleStrategy.t.sol:QVSimpleStrategyTest
// $ forge test --match-test "testRevert_allocate_multiple_time_evading_maxVoiceCreditsPerAllocator" -vvv
//
    function testRevert_allocate_multiple_time_evading_maxVoiceCreditsPerAllocator() public {
        // The `maxVoiceCreditsPerAllocator` can be evaded because the `allocator.voiceCredits` does not
        // increment in each allocating action causing that the allocator can allocate infinate credits.
        //
        address recipientId = __register_accept_recipient();
        address allocator = randomAddress();
        //
        // Create the allowed allocator
        vm.startPrank(pool_manager2());
        qvSimpleStrategy().addAllocator(allocator);
        vm.stopPrank();
        vm.warp(allocationStartTime + 10);
        //
        // Allocator cast 90 credits to allocate to recipientId.
        // The recipientId has 9486832980 votes
        bytes memory allocateData = __generateAllocation(recipientId, 90);
        vm.startPrank(allocator);
        allo().allocate(poolId, allocateData);
        (uint256 recipientTotalVotesReceived,,,,) = qvSimpleStrategy().recipients(recipientId);
        assertEq(recipientTotalVotesReceived, 9486832980);
        //
        // Same Allocator cast another 90 credits to allocate to recipientId.
        // The recipientId has 13416407864 votes
        // The Allocator evaded the `maxVoiceCreditsPerAllocator`
        allo().allocate(poolId, allocateData);
        (recipientTotalVotesReceived,,,,) = qvSimpleStrategy().recipients(recipientId);
        assertEq(recipientTotalVotesReceived, 13416407864);
        vm.stopPrank();
    }
```
## Impact

The `allocator` can use his credits to multiply the `recipient` payout since the allocator can assign credits multiple times evading the `QVSimpleStrategy.maxVoiceCreditsPerAllocator`. Malicious allocator can collude with a recipient then extract all the available amount in the pool.

## Code Snippet

- [QVSimpleStrategy::_allocate()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-simple/QVSimpleStrategy.sol#L107C14-L107C23)
- [QVSimpleStrategy::_hasVoiceCreditsLeft](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-simple/QVSimpleStrategy.sol#L144C14-L144C34)
- [QVBaseStrategy::_qv_allocate()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L506)
- [QVBaseStrategy::_getPayout()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L559C14-L559C24)

## Tool used

Manual review

## Recommendation

Increase the `allocator.voiceCredits` in the [QVBaseStrategy::_qv_allocate()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L506C14-L506C26) function:

```diff
    function _qv_allocate(
        Allocator storage _allocator,
        Recipient storage _recipient,
        address _recipientId,
        uint256 _voiceCreditsToAllocate,
        address _sender
    ) internal onlyActiveAllocation {
        // check the `_voiceCreditsToAllocate` is > 0
        if (_voiceCreditsToAllocate == 0) revert INVALID();

        // get the previous values
        uint256 creditsCastToRecipient = _allocator.voiceCreditsCastToRecipient[_recipientId];
        uint256 votesCastToRecipient = _allocator.votesCastToRecipient[_recipientId];

        // get the total credits and calculate the vote result
        uint256 totalCredits = _voiceCreditsToAllocate + creditsCastToRecipient;
        uint256 voteResult = _sqrt(totalCredits * 1e18);

        // update the values
        voteResult -= votesCastToRecipient;
        totalRecipientVotes += voteResult;
        _recipient.totalVotesReceived += voteResult;

        _allocator.voiceCreditsCastToRecipient[_recipientId] += totalCredits;
        _allocator.votesCastToRecipient[_recipientId] += voteResult;
++      _allocator.voiceCredits += _voiceCreditsToAllocate;

        // emit the event with the vote results
        emit Allocated(_recipientId, voteResult, _sender);
    }
```

Then the [hasVoiceCreditsLeft()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-simple/QVSimpleStrategy.sol#L121C59-L121C82) call can work correctly.

Duplicate of #150
