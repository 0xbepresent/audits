# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/350
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Staking.sol#L328


# Vulnerability details

## Impact

The [stakeGGP()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Staking.sol#LL319C45-L319C58) function helps to stake the GGP only if the protocol is not paused (whenNotPaused modifier). The [restakeGGP()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Staking.sol#L328) function does not validate if the protocol is paused causing that users stake their unclaimed GGP rewards even if the protocol is paused.

The ```stakeGGP()``` function is only possible if the protocol is not paused but the ```restakeGGP()``` function bypass that restriction. The protocol is still staking even when it is paused.

## Proof of Concept

1. As a NodeOp get rewards for the ggp staked.
2. The protocol is paused.
3. The NodeOp can call [claimAndRestake()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ClaimNodeOp.sol#L89) function in order to re-stake their unclaimed rewards even when the protocol is paused.


## Tools used
Manual review

## Recommended Mitigation Steps

It would be possible to create a function where the user can claim all the ggp staked plus rewards even if the protocol is paused and the ```claimAndRestake()``` can be used if the protocol is not paused.