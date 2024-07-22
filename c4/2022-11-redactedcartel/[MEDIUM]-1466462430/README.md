# Original link
https://github.com/code-423n4/2022-11-redactedcartel-findings/issues/271
# Lines of code

https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L390-L391


# Vulnerability details

## Impact
A user (which can also be one of the autocompounding contracts, `AutoPxGlp` or `AutoPxGmx`) can loss a reward as a result of reward tokens mismanagement by the owner.
## Proof of Concept
The protocol defines a short list of reward tokens that are hard coded in the `claimRewards` function of the `PirexGmx` contract ([PirexGmx.sol#L756-L759](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexGmx.sol#L756-L759)):
```solidity
rewardTokens[0] = gmxBaseReward;
rewardTokens[1] = gmxBaseReward;
rewardTokens[2] = ERC20(pxGmx); // esGMX rewards distributed as pxGMX
rewardTokens[3] = ERC20(pxGmx);
```

The fact that these addresses are hard coded means that no other reward tokens will be supported by the protocol. However, the `PirexRewards` contract maintains a different list of reward tokens, one per producer token ([PirexRewards.sol#L19-L31](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L19-L31)):
```solidity
struct ProducerToken {
    ERC20[] rewardTokens;
    GlobalState globalState;
    mapping(address => UserState) userStates;
    mapping(ERC20 => uint256) rewardStates;
    mapping(address => mapping(ERC20 => address)) rewardRecipients;
}

// Producer tokens mapped to their data
mapping(ERC20 => ProducerToken) public producerTokens;
```

These reward tokens can be added ([PirexRewards.sol#L151](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L151)) or removed ([PirexRewards.sol#L179](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L179)) by the owner, which creates the possibility of a mismanagement:
1. the owner can mistakenly remove one of the reward tokens hard coded in the `PirexGmx` contract;
1. the owner can add reward tokens that are not supported by the `PirexGmx` contract.

Such mismanagement can cause users to lose rewards for two reasons:
1. reward state of a user is updated *before* their rewards are claimed;
1. it's the reward token addresses set by the owner of the `PirexRewards` contract that are used to transfer rewards.

In the `claim` function:
1. `harvest` is called to pull rewards from GMX ([PirexRewards.sol#L377](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L377)):
    ```solidity
    harvest();
    ```
1. `claimReward` is called on `PirexGmx` to pull rewards from GMX and get the hard coded lists of producer tokens, reward tokens, and amounts ([PirexRewards.sol#L346-L347](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L346-L347)):
    ```solidity
    (_producerTokens, rewardTokens, rewardAmounts) = producer
    .claimRewards();
    ```
1. rewards are recorded for each of the hard coded reward token ([PirexRewards.sol#L361](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L361)):
    ```solidity
    if (r != 0) {
        producerState.rewardStates[rewardTokens[i]] += r;
    }
    ```
1. later in the `claim` function, owner-set reward tokens are read ([PirexRewards.sol#L386-L387](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L386-L387)):
    ```solidity
    ERC20[] memory rewardTokens = p.rewardTokens;
    uint256 rLen = rewardTokens.length;
    ```
1. user reward state is set to 0, which means they've claimed their entire share of rewards ([PirexRewards.sol#L391](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L391)), however this is done before a reward is actually claimed:
    ```solidity
    p.userStates[user].rewards = 0;
    ```
1. the owner-set reward tokens are iterated and the previously recorded rewards are distributed ([PirexRewards.sol#L396-L415](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/PirexRewards.sol#L396-L415)):
    ```solidity
    for (uint256 i; i < rLen; ++i) {
        ERC20 rewardToken = rewardTokens[i];
        address rewardRecipient = p.rewardRecipients[user][rewardToken];
        address recipient = rewardRecipient != address(0)
            ? rewardRecipient
            : user;
        uint256 rewardState = p.rewardStates[rewardToken];
        uint256 amount = (rewardState * userRewards) / globalRewards;

        if (amount != 0) {
            // Update reward state (i.e. amount) to reflect reward tokens transferred out
            p.rewardStates[rewardToken] = rewardState - amount;

            producer.claimUserReward(
                address(rewardToken),
                amount,
                recipient
            );
        }
    }
    ```

In the above loop, there can be multiple reasons for rewards to not be sent:
1. one of the hard coded reward tokens is missing in the owner-set reward tokens list;
1. the owner-set reward token list contains a token that's not supported by `PirexGmx` (i.e. it's not in the hard coded reward tokens list);
1. the `rewardTokens` array of a producer token turns out to be empty due to a mismanagement by the owner.

In all of the above situations rewards won't be sent, however user's reward state will still be set to 0.

Also, notice that calling `claim` won't revert if reward tokens are misconfigured, and the `Claim` event will be emitted successfully, which makes reward tokens mismanagement hard to detect.

The amount of lost rewards can be different depending on how much GMX a user has staked and how often they claim rewards. Of course, if a mistake isn't detected quickly, multiple users can suffer from this issue. The autocompounding contracts (`AutoPxGlp` and `AutoPxGmx`) are also users of the protocol, and since they're intended to hold big amounts of real users' deposits (they'll probably be the biggest stakers), lost rewards can be big.
## Tools Used
Manual review
## Recommended Mitigation Steps
Consider having one source of reward tokens. Since they're already hard coded in the `PirexGmx` contract, consider exposing them so that `PirexRewards` could read them in the `claim` function. This change will also mean that the `addRewardToken` and `removeRewardToken` functions won't be needed, which makes contract management simpler.
Also, in the `claim` function, consider updating global and user reward states only after ensuring that at least one reward token was distributed.