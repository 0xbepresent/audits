# Original link
https://github.com/sherlock-audit/2023-09-Gitcoin-judging/issues/534
0xbepresent

high

# Error in counting the `allocator.voiceCreditsCastToRecipient` causing the `recipient` to have more votes and get the majority of the pool
## Summary

There is an error counting in the `allocator.voiceCreditsCastToRecipient` variable in the [QVBaseStrategy::_qv_allocate()] causing that an allocator to assign more votes.

## Vulnerability Detail

The `allocator` can cast votes using the [QVBaseStrategy::_qv_allocate()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L506C14-L506C26) function. The problem is that there is an error in the counting [voiceCreditsCastToRecipient](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L529C9-L529C78) variable in the code line 529:

```solidity
File: QVBaseStrategy.sol
506:     function _qv_allocate(
507:         Allocator storage _allocator,
508:         Recipient storage _recipient,
509:         address _recipientId,
510:         uint256 _voiceCreditsToAllocate,
511:         address _sender
512:     ) internal onlyActiveAllocation {
513:         // check the `_voiceCreditsToAllocate` is > 0
514:         if (_voiceCreditsToAllocate == 0) revert INVALID();
515: 
516:         // get the previous values
517:         uint256 creditsCastToRecipient = _allocator.voiceCreditsCastToRecipient[_recipientId];
518:         uint256 votesCastToRecipient = _allocator.votesCastToRecipient[_recipientId];
519: 
520:         // get the total credits and calculate the vote result
521:         uint256 totalCredits = _voiceCreditsToAllocate + creditsCastToRecipient;
522:         uint256 voteResult = _sqrt(totalCredits * 1e18);
523: 
524:         // update the values
525:         voteResult -= votesCastToRecipient;
526:         totalRecipientVotes += voteResult;
527:         _recipient.totalVotesReceived += voteResult;
528: 
529:         _allocator.voiceCreditsCastToRecipient[_recipientId] += totalCredits;
530:         _allocator.votesCastToRecipient[_recipientId] += voteResult;
531: 
532:         // emit the event with the vote results
533:         emit Allocated(_recipientId, voteResult, _sender);
534:     }
```

Please see the next scenario:

`AllocatorC` cast 2 voice credits to the `recipientB`.

1. `AllocatorC` allocates `1 voice credit` to the `recipientB`. The `creditsCastToRecipient` is zero because is the first time the `allocatorC` cast votes to the `recipientB` so `_allocator.voiceCreditsCastToRecipient[_recipientId]` is zero. [Code line 517](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L517C9-L517C95).
2. Now, the `_allocator.voiceCreditsCastToRecipient[_recipientId]` [code line 529 increases](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L529) to 1 voice credit.
3. `AllocatorC` allocates another `1 voice credit` to the `recipientB`. The `creditsCastToRecipient` now is `1 voice credit` because there is one credit casted in the step 2. So now totalCredits will be `2` because `_voiceCreditsToAllocate` is 1 and `creditsCastToRecipient` is 1, in the [code line 521](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L521).
4. Now the variable `_allocator.voiceCreditsCastToRecipient[_recipientId]` will be `3` in the [code line 529](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L529). The new `2` sum from the `step 3` and the casted `one` allocator.voiceCreditsCastToRecipient from the first step, **that is incorrect because the current voice credits used by the `AllocatorC` to the `recipientB` is `2` NOT `3`**.

At the end, the `allocatorC` uses only 2 voice credits but the `recipientB` has 3 `voiceCreditsCastToRecipient` which is incorrect because `recipientB` should have only `2 voice casted credits`.

## Impact

An allocator can assign more votes to a recipient causing that the recipient to recive a bigger payout. The [calculation of the voteResult](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L522) is based on the allocator's casted credits so the allocator can cast one by one credit to the same recipient exponentiating the emited voice credits to the recipient. Malicious Allocator and recipient can collude to get the majority of the pool.


## Code Snippet

- [QVBaseStrategy::_qv_allocate()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L506C14-L506C26)
- [QVBaseStrategy::_getPayout()](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L559C14-L559C24)

## Tool used

Manual review

## Recommendation

Modify the [code line 529](https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/strategies/qv-base/QVBaseStrategy.sol#L529) so the casted credits to a recipient are not counted multiple times:

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

--      _allocator.voiceCreditsCastToRecipient[_recipientId] += totalCredits;
++      _allocator.voiceCreditsCastToRecipient[_recipientId] = totalCredits;
        _allocator.votesCastToRecipient[_recipientId] += voteResult;

        // emit the event with the vote results
        emit Allocated(_recipientId, voteResult, _sender);
    }
```

Duplicate of #48
