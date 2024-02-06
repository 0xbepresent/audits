# stake.link - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. The lockId.boostAmount, of users who have locked their tokens, is not updated if maxBoost is changed, causing a wrong rewards calculation](#H-01)
    - ### [H-02. The `SDLPoolCCIPControllerPrimary::distributeRewards` function does not support sending rewards tokens more than the limit allowed by `CCIP`](#H-02)
    - ### [H-03. The `tokenApprovals[_lockId]` is not deleted if the `ERC721` is transferred between multiple chains](#H-03)
- ## Medium Risk Findings
    - ### [M-01. The user will lose their tokens on the `secondary chain` if they decide to withdraw all their tokens, then for some reason back out and deposit tokens for the same `lockId`](#M-01)
    - ### [M-02. The `WrappedTokenBridge::_transferTokens` function does not offer protection against an undesired amount of wrapped token output](#M-02)
    - ### [M-03. If there are no `whitelisted chains` while doing a rewards distribution, the `wrapped tokens` will be stuck in the controller contract](#M-03)
    - ### [M-04. If the value of `maxLockingDuration` is decreased while there are locked deposits in process, new deposits to those `lockIds` will be reverted](#M-04)
- ## Low Risk Findings
    - ### [L-01. Updates from the `secondary pool` to the `primary pool` may not be sent because there are `no rewards` for the secondary pool](#L-01)
    - ### [L-02. Creating/transfering the `lockId` does not verify that the creator implements `onERC721Received` if creator is a contract](#L-02)
    - ### [L-03. The `extraArgs` parameter used in the `WrappedTokenBridge::_buildCCIPMessage` should be mutable in order to allow compatibility with future CCIP upgrade](#L-03)
    - ### [L-04. The `SDLPoolSecondary::withdraw` function will be reverted if the chains are not synchronized with each other](#L-04)
    - ### [L-05. There is no limit to queuing updates on `secondary chains` when a `lockId` is modified](#L-05)
    - ### [L-06. There is not protection for fees increment while the `reSDL` is transferring to another chain](#L-06)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: stake.link

### Dates: Dec 22nd, 2023 - Jan 12th, 2024

[See more contest details here](https://www.codehawks.com/contests/clqf7mgla0001yeyfah59c674)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 4
   - Low: 6


# High Risk Findings

## <a id='H-01'></a>H-01. The lockId.boostAmount, of users who have locked their tokens, is not updated if maxBoost is changed, causing a wrong rewards calculation            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/LinearBoostController.sol#L56

## Summary

The `lockId.boostAmount`, of users who have locked their tokens, is not updated if `maxBoost` is changed.

## Vulnerability Details

In order to calculate the [`boostAmount`](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/LinearBoostController.sol#L36) when a user stakes, the variable `maxBoost` is used ([code line 38](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/LinearBoostController.sol#L38)):

```solidity
File: LinearBoostController.sol
36:     function getBoostAmount(uint256 _amount, uint64 _lockingDuration) external view returns (uint256) {
37:         if (_lockingDuration > maxLockingDuration) revert MaxLockingDurationExceeded();
38:         return (_amount * uint256(maxBoost) * uint256(_lockingDuration)) / uint256(maxLockingDuration);
39:     }
```

Afterwards, the `boostAmount` is used in the [lockId creation](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/base/SDLPool.sol#L392) or in the [lockId update](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/base/SDLPool.sol#L420) when user increases the duration of the locking.

The problem arises when the admin decides to change `maxBoost` because the new `maxBoost` will not be reflected in the `boostAmount` registered in the `lockId` of all users staking. Consider the following scenario:

> maxBoost = 2; maxLockingDuration = 1 year

1. `UserA` stakes `10 tokens` with a duration of 1 year, the `boostAmount is 20`, it is calculated as `_amount * maxBoost * _lockingDuration / maxLockingDuration = (10 * 2 maxBoost * 1 year) / 1 year = 20`
2. Admin changes maxBoost, now `maxBoost = 1`.
3. Now, the new UserA `boostAmount should be 10`, it would be calculated as `_amount * maxBoost * _lockingDuration / maxLockingDuration = (10 * 1 maxBoost * 1 year) / 1 year = 10`.

The problem is that the `user's boostAmount` is not modified after the `maxBoost` modification in `step 2`. In the example above, the user would prefer not to change his `boostAmount` after the `maxBoost` was modified simply because it is more convenient for the user since if he updates his `boostAmount` it will decrease.

In order to calculate rewards the [docs](https://github.com/Cyfrin/2023-12-stake-link?tab=readme-ov-file#sdlpoolprimary) says: *This effective balance is used to determine the amount of benefit a user receives in various aspects of the protocol such as the amount of rewards received.* So the rewards calculation uses effectiveBalances and the [`effectiveBalances` may change](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolPrimary.sol#L319) if `maxBoost` is modified.

## Impact

The `boostAmount` recorded in the `lockIds` will not change after `maxBoost` is modified, causing users to have an incorrect `boostAmount` and the `rewards` will be calculated wrongly.

## Tools used

Manual review

## Recommendations

Add an external function that helps update the `lockId.boostAmount` if applicable. Thus this function can be called by the owner of the `lockId` himself or by someone else.

```solidity
    function updateUserBoost(uint256 _lockId) external {
        uint256 baseAmount = locks[_lockId].amount;
        uint256 newBoostAmount = boostController.getBoostAmount(baseAmount, locks[_lockId].duration);

        locks[_lockId].boostAmount = boostAmount;
    }
```
## <a id='H-02'></a>H-02. The `SDLPoolCCIPControllerPrimary::distributeRewards` function does not support sending rewards tokens more than the limit allowed by `CCIP`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L56

## Summary

The function [SDLPoolCCIPControllerPrimary::distributeRewards](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L56C14-L56C31) does not support sending rewards tokens more than the limit allowed by `CCIP`.

## Vulnerability Details

The function [SDLPoolCCIPControllerPrimary::distributeRewards](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L56C14-L56C31) helps to be able to send the [supported token rewards](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L58C35-L58C78) to different `whitelisted chains`, this with the help of [`CCIP.ccipSend`](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L284). Afterwards the `secondary chains` can [receive the rewards](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L151-L157). 

On the other hand, `CCIP` has a maximum allowed number of tokens sent per message, this can be seen in the following validation [maxNumberOfTokensPerMsg](https://github.com/smartcontractkit/ccip/blob/a5a4a0ac1f8e18e156edfb2af9ec468a61747a4a/contracts/src/v0.8/ccip/onRamp/EVM2EVMOnRamp.sol#L384), that means that if in the function [SDLPoolCCIPControllerPrimary::distributeRewards](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L56C14-L56C31) distributes more tokens than allowed by `CCIP` the distribution of rewards will be reversed, making the distribution of rewards not possible.

The function [SDLPoolCCIPControllerPrimary::distributeRewards](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L56) obtains the `supported tokens` with help of the function [`ISDLPoolPrimary(sdlPool).supportedTokens()`](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L58C35-L58C78)), however it does not have a procedure to send multiple messages and thus not exceed the limit of tokens per message of `CCIP`.

```solidity
File: SDLPoolCCIPControllerPrimary.sol
56:     function distributeRewards() external onlyRewardsInitiator {
57:         uint256 totalRESDL = ISDLPoolPrimary(sdlPool).effectiveBalanceOf(address(this));
58:         address[] memory tokens = ISDLPoolPrimary(sdlPool).supportedTokens();
59:         uint256 numDestinations = whitelistedChains.length;
60: 
61:         ISDLPoolPrimary(sdlPool).withdrawRewards(tokens);
62: 
63:         uint256[][] memory distributionAmounts = new uint256[][](numDestinations);
64:         for (uint256 i = 0; i < numDestinations; ++i) {
65:             distributionAmounts[i] = new uint256[](tokens.length);
66:         }
67: 
68:         for (uint256 i = 0; i < tokens.length; ++i) {
69:             address token = tokens[i];
70:             uint256 tokenBalance = IERC20(token).balanceOf(address(this));
71: 
72:             address wrappedToken = wrappedRewardTokens[token];
73:             if (wrappedToken != address(0)) {
74:                 IERC677(token).transferAndCall(wrappedToken, tokenBalance, "");
75:                 tokens[i] = wrappedToken;
76:                 tokenBalance = IERC20(wrappedToken).balanceOf(address(this));
77:             }
78: 
79:             uint256 totalDistributed;
80:             for (uint256 j = 0; j < numDestinations; ++j) {
81:                 uint64 chainSelector = whitelistedChains[j];
82:                 uint256 rewards = j == numDestinations - 1
83:                     ? tokenBalance - totalDistributed
84:                     : (tokenBalance * reSDLSupplyByChain[chainSelector]) / totalRESDL;
85:                 distributionAmounts[j][i] = rewards;
86:                 totalDistributed += rewards;
87:             }
88:         }
89: 
90:         for (uint256 i = 0; i < numDestinations; ++i) {
91:             _distributeRewards(whitelistedChains[i], tokens, distributionAmounts[i]);
92:         }
93:     }
```

## Impact

The distribution of rewards [(SDLPoolCCIPControllerPrimary::distributeRewards)](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L56C14-L56C31) will not be executed if the number of tokens to be sent exceeds the limit allowed by `CCIP`. The rewards will not be distributed to the users so users will lost tokens rewards.

## Tools used

Manual review

## Recommendations

Modify the [SDLPoolCCIPControllerPrimary::distributeRewards](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L56C14-L56C31) function to be able to send multiple messages via `CCIP` to the `secondary chains` if [`supported tokens`](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L58) exceeds the limit allowed by `CCIP`. So the sending of rewards tokens will be in multiple batches in order to not exceed the `CCIP` limit.


## <a id='H-03'></a>H-03. The `tokenApprovals[_lockId]` is not deleted if the `ERC721` is transferred between multiple chains            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolPrimary.sol#L172

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L259

## Summary

The `tokenApprovals[_lockId]` is not deleted if the `ERC721` is transferred between multiple chains.

## Vulnerability Details

The `lockId` can be transferred from the `primary chain` to the `secondary chain` with the help of the function [RESDLTokenBridge::transferRESDL](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/RESDLTokenBridge.sol#L84C14-L84C27), this will transfer the token to the `destination chain`. Likewise, you can specify the [receiver](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/RESDLTokenBridge.sol#L86), that is, in transfer between chains the recipient can be another user.

The problem is that when transferring between chains, the `tokenApprovals[_lockId]` variable is not cleared. Keep in mind that when the `lockId` is transferred within the same chain `tokenApprovals[_lockId]` is [deleted](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/base/SDLPool.sol#L464C9-L464C40):

```solidity
File: SDLPool.sol
455:     function _transfer(
456:         address _from,
457:         address _to,
458:         uint256 _lockId
459:     ) internal virtual {
460:         if (_from != ownerOf(_lockId)) revert TransferFromIncorrectOwner();
461:         if (_to == address(0)) revert TransferToZeroAddress();
462:         if (_to == ccipController) revert TransferToCCIPController();
463: 
464:         delete tokenApprovals[_lockId];
465: 
466:         _updateRewards(_from);
467:         _updateRewards(_to);
468: 
469:         uint256 effectiveBalanceChange = locks[_lockId].amount + locks[_lockId].boostAmount;
470:         effectiveBalances[_from] -= effectiveBalanceChange;
471:         effectiveBalances[_to] += effectiveBalanceChange;
472: 
473:         balances[_from] -= 1;
474:         balances[_to] += 1;
475:         lockOwners[_lockId] = _to;
476: 
477:         emit Transfer(_from, _to, _lockId);
478:     }
```

## Impact

The `tokenApprovals[_lockId]` will be cleared when the `ERC721` is transferred within the same chain however `tokenApprovals[_lockId]` is NOT cleared when the transfer is made between multiple chains.

## Tools used

Manual review

## Recommendations

Clear `tokenApprovals[_lockId]` when transferring `ERC721` on multiple chains:

```diff
    function handleOutgoingRESDL(
        address _sender,
        uint256 _lockId,
        address _sdlReceiver
    )
        external
        onlyCCIPController
        onlyLockOwner(_lockId, _sender)
        updateRewards(_sender)
        updateRewards(ccipController)
        returns (Lock memory)
    {
        Lock memory lock = locks[_lockId];

++      // remove token approvals

        delete locks[_lockId].amount;
        delete lockOwners[_lockId];
        balances[_sender] -= 1;

        uint256 totalAmount = lock.amount + lock.boostAmount;
        effectiveBalances[_sender] -= totalAmount;
        effectiveBalances[ccipController] += totalAmount;

        sdlToken.safeTransfer(_sdlReceiver, lock.amount);

        emit OutgoingRESDL(_sender, _lockId);

        return lock;
    }
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. The user will lose their tokens on the `secondary chain` if they decide to withdraw all their tokens, then for some reason back out and deposit tokens for the same `lockId`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L451

## Summary

The user will lose their tokens on the `secondary chain` if they decide to withdraw all their tokens, then for some reason back out and deposit tokens for the same `lockId`.

## Vulnerability Details

The user can send `sdl tokens` on the `secondary chain` by queuing the action in [SDLPoolSecondary::_queueNewLock](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L363C14-L363C27), after the system is synchronized with `primary chain` that token is created with the help of the function [SDLPoolSecondary::_mintQueuedNewLocks](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L384C14-L384C33). Likewise, the user can withdraw their tokens with the help of the function [SDLPoolSecondary::withdraw](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L214C14-L214C22) which will [enqueue the action](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L234) and once the system is synchronized with `primary chain`, the [user will get his tokens](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L468-L479).

The problem arises when the user deposits tokens again once he has decided to withdraw all their tokens, this will cause them to lose their last deposited tokens. Please see the following test:

1. User deposits `100 tokens` on `secondary chain`
2. The system updates the new supply to the `primary chain` using the [handleOutgoingUpdate()](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L313). Now user mints the new token `lockId=1`.
3. User withdraws all his `100 tokens`, the action is queued and the new supply is send to primary chain in order to be synchronized.
4. User for some reason retracts and deposits `20 tokens` to the `lockId=1`, this action is executed before the `Controller` returns the response to the `secondary chain` (step 3). Now the user has two queued actions, the action which the amount is 0 (withdrawal) and the action which the user deposit `20 tokens`
5. Finally the controller updates the new info and the user execute the queued operations for the current batch using the [SDLPoolSecondary::_executeQueuedLockUpdates](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L451C14-L451C39). **At this point the `withdrawal is executed` and the [`lockId` is removed](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L470-L476)**. The user still has the `20 tokens` deposit action queued in the `getQueuedLockUpdates()` function.
6. System sends the new batch to L1 and then the controller returns the response.
7. Now the user wants to execute the queued `20 tokens deposit` using the [SDLPoolSecondary::_executeQueuedLockUpdates](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L451C14-L451C39) and it will be reverted because the `lockId` was removed in the `step 5`. User wants to withdraw and also it will be reverted. User lost his tokens.

```js
// Filename: test/core/sdlPool/sdl-pool-secondary.test.ts
// $ yarn test --grep "codehawks: mint, withdraw, deposit then revert"
//
  it('codehawks: mint, withdraw, deposit then revert', async () => {
    //
    // 1. User deposits 100 tokens
    await sdlToken.transferAndCall(
      sdlPool.address,
      toEther(100),
      ethers.utils.defaultAbiCoder.encode(['uint256', 'uint64'], [0, 0])
    )
    assert.deepEqual(parseNewLocks(await sdlPool.getQueuedNewLocksByOwner(accounts[0])), [
      [{amount: 100, boostAmount: 0, startTime: 0, duration: 0, expiry: 0,},],
      [1],
    ])
    //
    // 2. User mints the new token (lockId=1)
    await sdlPool.setCCIPController(accounts[0])
    await sdlPool.handleOutgoingUpdate()
    await sdlPool.handleIncomingUpdate(1)
    await sdlPool.executeQueuedOperations([])
    assert.equal(fromEther(await sdlPool.totalEffectiveBalance()), 100)
    assert.equal(await sdlPool.ownerOf(1), accounts[0])
    // the queue new lock is empty now
    assert.deepEqual(parseNewLocks(await sdlPool.getQueuedNewLocksByOwner(accounts[0])), [[], [],])
    //
    // 3. User withdraws all his tokens (100), the action is queued and the new supply is send to primary chain
    await sdlPool.withdraw(1, toEther(100))
    await sdlPool.handleOutgoingUpdate() // data is sent to L1
    assert.deepEqual(parseLockUpdates(await sdlPool.getQueuedLockUpdates([1])), [
      [
        {
          updateBatchIndex: 2,
          lock: { amount: 0, boostAmount: 0, startTime: 0, duration: 0, expiry: 0 },
        },
      ],
    ])
    //
    // 4. User deposits some tokens to the lockId=1 before the Controller returns the response
    await sdlToken.transferAndCall(
      sdlPool.address,
      toEther(20),
      ethers.utils.defaultAbiCoder.encode(['uint256', 'uint64'], [1, 0])
    )
    // User has a list of two arrays
    assert.deepEqual(parseLockUpdates(await sdlPool.getQueuedLockUpdates([1])), [
      [
        {
          updateBatchIndex: 2,
          lock: { amount: 0, boostAmount: 0, startTime: 0, duration: 0, expiry: 0 },
        },
        {
          updateBatchIndex: 3,
          lock: { amount: 20, boostAmount: 0, startTime: 0, duration: 0, expiry: 0 },
        },
      ],
    ])
    //
    // 5. Finally the controller updates the new info and the queued operations for the current batch
    // and lockId=1 are executed
    await sdlPool.handleIncomingUpdate(2)
    await sdlPool.executeQueuedOperations([1])
    assert.equal(fromEther(await sdlPool.totalEffectiveBalance()), 0)
    assert.equal(fromEther(await sdlToken.balanceOf(sdlPool.address)), 20);
    // The user still has the deposit of the 20 tokens in queue
    assert.deepEqual(parseLockUpdates(await sdlPool.getQueuedLockUpdates([1])), [
       [
         {
           updateBatchIndex: 3,
           lock: { amount: 20, boostAmount: 0, startTime: 0, duration: 0, expiry: 0 },
         },
       ],
     ])
    //
    // 6. System sends the new batch to L1 and the L1 returns the response
    await sdlPool.handleOutgoingUpdate()
    await sdlPool.handleIncomingUpdate(3)
    //
    // 7. User wants to execute the queued deposit and it will be reverted.
    // User wants to withdraw and also it will be reverted.
    // User lost his tokens.
    await expect(sdlPool.executeQueuedOperations([1])).to.be.revertedWith('InvalidLockId()')
    await expect(sdlPool.withdraw(1, toEther(20))).to.be.revertedWith('InvalidLockId()')
    assert.equal(fromEther(await sdlPool.totalEffectiveBalance()), 0)
    assert.equal(fromEther(await sdlToken.balanceOf(sdlPool.address)), 20); // pool still has the user's 20 tokens deposit
    // The user still has his queue deposit tokens
    assert.deepEqual(parseLockUpdates(await sdlPool.getQueuedLockUpdates([1])), [
      [
        {
          updateBatchIndex: 3,
          lock: { amount: 20, boostAmount: 0, startTime: 0, duration: 0, expiry: 0 },
        },
      ],
    ])
  })
```

## Impact

The user loses tokens.

## Tools used

Manual review

## Recommendations

If the user withdraws all his tokens from a `lockId`, no longer allow a subsequent deposit to the same `lockId` since those tokens will no longer be recovered.
## <a id='M-02'></a>M-02. The `WrappedTokenBridge::_transferTokens` function does not offer protection against an undesired amount of wrapped token output            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L158

## Summary

The `WrappedTokenBridge::_transferTokens` function does not offer protection against an undesired amount of wrapped token output

## Vulnerability Details

The [WrappedTokenBridge::_transferTokens](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L158) function helps to send wrapped tokens to a `destination chain`, the token is [transferred to the contract](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L83), then the deposited amount is [converted to a number of wrapped tokens](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L168) and finally it is [sent to the destination chain](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L172):

```solidity
File: WrappedTokenBridge.sol
158:     function _transferTokens(
159:         uint64 _destinationChainSelector,
160:         address _sender,
161:         address _receiver,
162:         uint256 _amount,
163:         bool _payNative,
164:         uint256 _maxLINKFee
165:     ) internal returns (bytes32 messageId) {
166:         uint256 preWrapBalance = wrappedToken.balanceOf(address(this));
167:         wrappedToken.wrap(_amount);
168:         uint256 amountToTransfer = wrappedToken.balanceOf(address(this)) - preWrapBalance;
169: 
170:         Client.EVM2AnyMessage memory evm2AnyMessage = _buildCCIPMessage(
171:             _receiver,
172:             amountToTransfer,
173:             _payNative ? address(0) : address(linkToken)
174:         );
175: 
176:         IRouterClient router = IRouterClient(this.getRouter());
177:         uint256 fees = router.getFee(_destinationChainSelector, evm2AnyMessage);
178: 
179:         if (_payNative) {
180:             if (fees > msg.value) revert InsufficientFee();
181:             messageId = router.ccipSend{value: fees}(_destinationChainSelector, evm2AnyMessage);
182:             if (fees < msg.value) {
183:                 (bool success, ) = _sender.call{value: msg.value - fees}("");
184:                 if (!success) revert TransferFailed();
185:             }
186:         } else {
187:             if (fees > _maxLINKFee) revert FeeExceedsLimit();
188:             linkToken.safeTransferFrom(_sender, address(this), fees);
189:             messageId = router.ccipSend(_destinationChainSelector, evm2AnyMessage);
190:         }
191: 
192:         emit TokensTransferred(
193:             messageId,
194:             _destinationChainSelector,
195:             _sender,
196:             _receiver,
197:             amountToTransfer,
198:             _payNative ? address(0) : address(linkToken),
199:             fees
200:         );
201:         return messageId;
202:     }
```

The problem is that the function [WrappedTokenBridge::_transferTokens](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L158) does not offer a protection to NOT execute the transaction if the amount of `wrapped tokens` is not a convenient amount for the user. This can happen if the transaction is in process in the mempool and while it is there, the number of `wrapped tokens` changes to an unwanted amount, causing the user to lose tokens.

## Impact

The user may receive an undesired amount of `wrapped tokens` because the transaction may be executed in an unforeseen amount of time, causing the amount of `wrapped tokens` that the user receives to be an undesired amount.

## Tools used

Manual review

## Recommendations

Add a protection so that the transaction is not executed if the number of `wrapped tokens` is not desired:

```diff
    function _transferTokens(
        uint64 _destinationChainSelector,
        address _sender,
        address _receiver,
        uint256 _amount,
        bool _payNative,
        uint256 _maxLINKFee,
++      uint256 _minWrappedTokens,
    ) internal returns (bytes32 messageId) {
        uint256 preWrapBalance = wrappedToken.balanceOf(address(this));
        wrappedToken.wrap(_amount);
        uint256 amountToTransfer = wrappedToken.balanceOf(address(this)) - preWrapBalance;
++      if (amountToTransfer < _minWrappedTokens) revert();
        ...
        ...
        ...
```
## <a id='M-03'></a>M-03. If there are no `whitelisted chains` while doing a rewards distribution, the `wrapped tokens` will be stuck in the controller contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L56

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L151

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L170

## Summary

If there are no `whitelisted chains` while doing a rewards distribution, the `wrapped tokens` will be stuck in the `controller` contract.

## Vulnerability Details

The [SDLPoolCCIPControllerPrimary::distributeRewards](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L56) function helps to distribute tokens or wrapped tokens to the corresponding whitelisted chains. The problem arises if for some reason there are NO `whitelisted chains`, the tokens that were converted to `wrapped tokens` will be [stuck in the `controller`](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L76), those rewards will NOT be distributed and the `wrapped tokens` will not be able to be recovered since the `controller` does not have the `unwrap` function implemented. In `SDLPoolCCIPControllerPrimary::distributeRewards` we can see that in [code line 73-77](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L73-L77) the tokens are converted to wrapped tokens, then if there are no `whitelisted chains` the _distributeRewards function will not be executed [(code line 91)](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L91), leaving the wrapped tokens inside the contract:

```solidity
File: SDLPoolCCIPControllerPrimary.sol
56:     function distributeRewards() external onlyRewardsInitiator {
57:         uint256 totalRESDL = ISDLPoolPrimary(sdlPool).effectiveBalanceOf(address(this));
58:         address[] memory tokens = ISDLPoolPrimary(sdlPool).supportedTokens();
59:         uint256 numDestinations = whitelistedChains.length;
60: 
61:         ISDLPoolPrimary(sdlPool).withdrawRewards(tokens);
62: 
63:         uint256[][] memory distributionAmounts = new uint256[][](numDestinations);
64:         for (uint256 i = 0; i < numDestinations; ++i) {
65:             distributionAmounts[i] = new uint256[](tokens.length);
66:         }
67: 
68:         for (uint256 i = 0; i < tokens.length; ++i) {
69:             address token = tokens[i];
70:             uint256 tokenBalance = IERC20(token).balanceOf(address(this));
71: 
72:             address wrappedToken = wrappedRewardTokens[token];
73:             if (wrappedToken != address(0)) {
74:                 IERC677(token).transferAndCall(wrappedToken, tokenBalance, "");
75:                 tokens[i] = wrappedToken;
76:                 tokenBalance = IERC20(wrappedToken).balanceOf(address(this));
77:             }
78: 
79:             uint256 totalDistributed;
80:             for (uint256 j = 0; j < numDestinations; ++j) {
81:                 uint64 chainSelector = whitelistedChains[j];
82:                 uint256 rewards = j == numDestinations - 1
83:                     ? tokenBalance - totalDistributed
84:                     : (tokenBalance * reSDLSupplyByChain[chainSelector]) / totalRESDL;
85:                 distributionAmounts[j][i] = rewards;
86:                 totalDistributed += rewards;
87:             }
88:         }
89: 
90:         for (uint256 i = 0; i < numDestinations; ++i) {
91:             _distributeRewards(whitelistedChains[i], tokens, distributionAmounts[i]);
92:         }
93:     }
```

The following test shows how the `controller` will keep the `wrapped tokens` instead of the `wTokenPool`.

```js
// File: test/core/ccip/sdl-pool-ccip-controller-primary.test.ts
// $ yarn test --grep "codehawks distributeRewards stuck wrapped tokens"
//
  it('codehawks distributeRewards stuck wrapped tokens', async () => {
    let wToken = await deploy('WrappedSDTokenMock', [token1.address])
    let rewardsPool = await deploy('RewardsPoolWSD', [
      sdlPool.address,
      token1.address,
      wToken.address,
    ])
    let wtokenPool = (await deploy('CCIPTokenPoolMock', [wToken.address])) as CCIPTokenPoolMock
    await sdlPool.addToken(token1.address, rewardsPool.address)
    await controller.approveRewardTokens([wToken.address])
    await controller.setWrappedRewardToken(token1.address, wToken.address)
    await onRamp.setTokenPool(wToken.address, wtokenPool.address)
    await offRamp.setTokenPool(wToken.address, wtokenPool.address)
    await controller.connect(signers[5]).handleOutgoingRESDL(77, accounts[0], 1)
    //
    // 1. send rewards to the pool
    await token1.transferAndCall(rewardsPool.address, toEther(500), '0x')
    //
    // 2. Remove all whitelisted chains
    assert.deepEqual(
      (await controller.getWhitelistedChains()).map((d) => d.toNumber()),
      [77]
    )
    await controller.removeWhitelistedChain(77)
    assert.deepEqual(
      (await controller.getWhitelistedChains()).map((d) => d.toNumber()),
      []
    )
    //
    // 3. Distribute rewards
    await controller.distributeRewards()
    //
    // 4. The wrapper tokens stays on the controller contract instead the wTokenPool
    assert.equal(fromEther(await wToken.balanceOf(controller.address)), 125)
    assert.equal(fromEther(await wToken.balanceOf(wtokenPool.address)), 0)
  })
```

Since the removing ([SDLPoolCCIPControllerPrimary::removeWhitelistedChain](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L170C14-L170C36)) and adding ([SDLPoolCCIPControllerPrimary::addWhitelistedChain](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L151C14-L151C33)) functions are separated, there may be times when the system runs out of `whitelisted chains`. Moreover the [SDLPoolCCIPControllerPrimary::removeWhitelistedChain](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L170C14-L170C36) function does not have a protection to leave at least one `whitelisted chain`, therefore procedures must be prevented while there are no `whitelisted chains` registered.

## Impact

`Wrapped tokens` can get trapped in the `controller` contract.

## Tools used

Manual review

## Recommendations

If there are no `whitelisted chains`, then do a revert:

```diff
// File: SDLPoolCCIPControllerPrimary.sol
//
   function distributeRewards() external onlyRewardsInitiator {
        uint256 totalRESDL = ISDLPoolPrimary(sdlPool).effectiveBalanceOf(address(this));
        address[] memory tokens = ISDLPoolPrimary(sdlPool).supportedTokens();
        uint256 numDestinations = whitelistedChains.length;
++      if (nummDestinations == 0) revert();
        ...
        ...
        ...
    }
```

Additionally, [SDLPoolCCIPControllerPrimary::removeWhitelistedChain](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L170C14-L170C36) function should leave at least one `whitelisted chain`.
## <a id='M-04'></a>M-04. If the value of `maxLockingDuration` is decreased while there are locked deposits in process, new deposits to those `lockIds` will be reverted            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/LinearBoostController.sol#L45

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolPrimary.sol#L307C14-L307C31

## Summary

If the value of `maxLockingDuration` is decreased while there are locked deposits in process, the deposit to those `lockIds` will be reverted, not allowing users to deposit more to their `lockId`.

## Vulnerability Details

The user can add amounts multiple times to the same `lockId` with the help of the function [SDLPoolPrimary::_storeUpdatedLock](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolPrimary.sol#L307C14-L307C31), this will cause it to be deposited to the same `lockId` if the user so wishes. The problem arises when the variable `maxLockingDuration` is decremented using the function [LinearBoostController::setMaxLockingDuration](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/LinearBoostcontroller.sol#L45), causing subsequent user deposits to the same `lockId` be reversed.

The following test shows how the user can deposit multiple times to the same `lockId=1`, however once `maxLockingDuration` is decreased, subsequent deposits for the same `lockId` will be reversed, making the user no longer able to deposit to the same `lockId`. Test steps:

1. User stakes `100 tokens` using the max locking duration (4 years).
2. User deposits another `100 tokens` to the `lockId=1` using the same duration 4 years.
3. Now the admin reduces the max locking duration to 2 years.
4. User deposits more tokens to the `lockId=1` using the same duration 4 years (like the step 2). The transaction will be reverted by `MaxLockingDurationExceeded()`
5. The user tries to reduce the duration time to an accepted time (1 year) and deposits to the same `lockId=1` but the transaction will be reverted by `InvalidLockingDuration`


```js
// Filename: test/core/sdlPool/sdl-pool-primary.test.ts
// $ yarn test --grep "codehawks: boost change maxLockDuration"
//
  it('codehawks: boost change maxLockDuration', async () => {
    //
    // 1. User stakes 100 tokens using the max locking duration (4 years)
    await sdlToken.transferAndCall(
      sdlPool.address,
      toEther(100),
      ethers.utils.defaultAbiCoder.encode(['uint256', 'uint64'], [0, 4 * 365 * DAY])
    )
    let ts1 = (await ethers.provider.getBlock(await ethers.provider.getBlockNumber())).timestamp
    assert.equal(fromEther(await sdlToken.balanceOf(sdlPool.address)), 100)
    // User owns the lockId=1
    assert.deepEqual(parseLocks(await sdlPool.getLocks([1])), [
      { amount: 100, boostAmount: 400, startTime: ts1, duration: 4 * 365 * DAY, expiry: 0 },
    ])
    assert.equal(await sdlPool.ownerOf(1), accounts[0])
    assert.equal(fromEther(await sdlPool.effectiveBalanceOf(accounts[0])), 500) // 100 tokens + 400 boostAmount
    //
    // 2. User deposits another 100 tokens to the lockId=1 using the same duration 4 years, everything is ok
    // now the effectiveBalance=1000
    await sdlToken.transferAndCall(
      sdlPool.address,
      toEther(100),
      ethers.utils.defaultAbiCoder.encode(['uint256', 'uint64'], [1, 4 * 365 * DAY])
    )
    assert.equal(fromEther(await sdlPool.effectiveBalanceOf(accounts[0])), 1000) // 200 tokens + 800 boostAmount
    //
    // 3. Now the admin reduces the locking duration to 2 years
    await boostController.setMaxLockingDuration(2 * 365 * DAY)
    //
    // 4. User deposits more tokens to the lockId=1 using the same duration (4 years). The transaction
    // will be reverted by MaxLockingDurationExceeded()
    await expect(sdlToken.transferAndCall(
      sdlPool.address,
      toEther(20),
      ethers.utils.defaultAbiCoder.encode(['uint256', 'uint64'], [1, 4 * 365 * DAY])
    )).to.be.revertedWith('MaxLockingDurationExceeded()')
    //
    // 5. The user tries to reduce the duration time to an accepted time (1 year) and deposits to the same lockId=1
    // but the transaction will be reverted by InvalidLockingDuration
    await expect(sdlToken.transferAndCall(
      sdlPool.address,
      toEther(20),
      ethers.utils.defaultAbiCoder.encode(['uint256', 'uint64'], [1, 365 * DAY])
    )).to.be.revertedWith('InvalidLockingDuration');
  })
```

## Impact

The user will not be able to deposit tokens to the same `lockId` if the `maxLockingDuration` decreases, even if the user reduces its duration, the transaction will be reversed.

## Tools used

Manual review

## Recommendations

Allow deposits to the same `lockId` even if `maxLockingDuration` decreases. Modifying `maxLockingDuration` should not affect user deposits. Another solution could be to not allow the decrease of `maxLockingDuration`, only allow the increase.

# Low Risk Findings

## <a id='L-01'></a>L-01. Updates from the `secondary pool` to the `primary pool` may not be sent because there are `no rewards` for the secondary pool            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L157

## Summary

The [SDLPoolCCIPControllerSecondary::performUpkeep()](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66) function is only available when there is a [`message of rewards`](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L157) from the `SDLPoolCCIPControllerPrimary`. That could be a problem if there are not rewards to distribute in a specific `secondary chain` causing that queue updates from the `secondarly chain` will not be informed to the `SDLPoolPrimary`.

## Vulnerability Details

The `secondary chain` informs to the `primary chain` the new `numNewRESDLTokens` and `totalRESDLSupplyChange` using the [SDLPoolCCIPControllerSecondary::performUpkeep](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66C14-L66C27) function, then the primary chain receives the [information](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L300) and it calculates the new [mintStartIndex](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L305). Note that the `primary chain` increments the `reSDLSupplyByChain` in the `code line 300`, this so that the `primary chain` has the information on how much supply of reSDL tokens there is in the `secondary chain`:

```solidity
File: SDLPoolCCIPControllerPrimary.sol
294:     function _ccipReceive(Client.Any2EVMMessage memory _message) internal override {
295:         uint64 sourceChainSelector = _message.sourceChainSelector;
296: 
297:         (uint256 numNewRESDLTokens, int256 totalRESDLSupplyChange) = abi.decode(_message.data, (uint256, int256));
298: 
299:         if (totalRESDLSupplyChange > 0) {
300:             reSDLSupplyByChain[sourceChainSelector] += uint256(totalRESDLSupplyChange);
301:         } else if (totalRESDLSupplyChange < 0) {
302:             reSDLSupplyByChain[sourceChainSelector] -= uint256(-1 * totalRESDLSupplyChange);
303:         }
304: 
305:         uint256 mintStartIndex = ISDLPoolPrimary(sdlPool).handleIncomingUpdate(numNewRESDLTokens, totalRESDLSupplyChange);
306: 
307:         _ccipSendUpdate(sourceChainSelector, mintStartIndex);
308: 
309:         emit MessageReceived(_message.messageId, sourceChainSelector);
310:     }
```

Now the [mintStartIndex is send to the secondary chain code line 307](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L307) and the secondary chain [receives the new mintStartIndex](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L161). This entire process helps to keep the information updated between the primary chain and the secondary chain.

On the other hand, when a secondary chain receive rewards, the secondary chain can call the function [SDLPoolCCIPControllerSecondary::performUpkeep](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66C14-L66C27) since `shouldUpdate` is `true` at [code line 157](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L157):

```solidity
File: SDLPoolCCIPControllerSecondary.sol
147:     function _ccipReceive(Client.Any2EVMMessage memory _message) internal override {
148:         if (_message.data.length == 0) {
149:             uint256 numRewardTokens = _message.destTokenAmounts.length;
150:             address[] memory rewardTokens = new address[](numRewardTokens);
151:             if (numRewardTokens != 0) {
152:                 for (uint256 i = 0; i < numRewardTokens; ++i) {
153:                     rewardTokens[i] = _message.destTokenAmounts[i].token;
154:                     IERC20(rewardTokens[i]).safeTransfer(sdlPool, _message.destTokenAmounts[i].amount);
155:                 }
156:                 ISDLPoolSecondary(sdlPool).distributeTokens(rewardTokens);
157:                 if (ISDLPoolSecondary(sdlPool).shouldUpdate()) shouldUpdate = true;
158:             }
159:         } else {
160:             uint256 mintStartIndex = abi.decode(_message.data, (uint256));
161:             ISDLPoolSecondary(sdlPool).handleIncomingUpdate(mintStartIndex);
162:         }
163: 
164:         emit MessageReceived(_message.messageId, _message.sourceChainSelector);
165:     }
```

Once `shouldUpdate` is `true`, the function [SDLPoolCCIPControllerSecondary::performUpkeep](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66C14-L66C27) can be called in order to send the new information (`numNewRESDLTokens` and `totalRESDLSupplyChange`) to the primary chain:

```solidity
    function performUpkeep(bytes calldata) external {
        if (!shouldUpdate) revert UpdateConditionsNotMet();

        shouldUpdate = false;
        _initiateUpdate(primaryChainSelector, primaryChainDestination, extraArgs);
    }
```

The problem is that the `primary chain` needs to send rewards to the `secondary chain` so that `shouldUpdate` is true and the function [SDLPoolCCIPControllerSecondary::performUpkeep](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66C14-L66C27) can be called. However, in certain circumstances it is possible that the `secondary chain` may never be able to send information to the primary chain since there may not be any rewards for the secondary chain. Please consider the next scenario:

1. `UserA` stakes directly in the `secondary chain` and the [queuedRESDLSupplyChange increments](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L373)
2. The increase in supply CANNOT be reported to the `primary chain` since `shouldUpdate = false` and the function [SDLPoolCCIPControllerSecondary::performUpkeep](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66C14-L66C27) will be reverted.
3. Rewards are calculated on the primary chain, however because the `secondary chain` has not been able to send the new supply information, [zero rewards reSDLSupplyByChain](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L84C39-L84C72) will be calculated for the secondary chain since `reSDLSupplyByChain[chainSelector]` has not been increased with the new information from `step 1`.
4. Since there are NO rewards assigned for the `secondary chain`, it is not possible to [set `shouldUpdate=True`](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L157), therefore the function [SDLPoolCCIPControllerSecondary::performUpkeep](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66C14-L66C27) will be reverted.

The following test shows that a user can send `sdl` tokens to the `secondary pool` however [SDLPoolCCIPControllerSecondary::performUpkeep](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66C14-L66C27) cannot be called since there are no rewards assigned to the `secondary pool`:

```js
// File: test/core/ccip/sdl-pool-ccip-controller-secondary.test.ts
// $ yarn test --grep "codehawks performUpkeep reverts"
// 
  it('codehawks performUpkeep reverts', async () => {
    await token1.transfer(tokenPool.address, toEther(1000))
    let rewardsPool1 = await deploy('RewardsPool', [sdlPool.address, token1.address])
    await sdlPool.addToken(token1.address, rewardsPool1.address)
    assert.equal(fromEther(await sdlPool.totalEffectiveBalance()), 400)
    assert.equal((await controller.checkUpkeep('0x'))[0], false)
    assert.equal(await controller.shouldUpdate(), false)
    //
    // 1. Mint in the secondary pool
    await sdlToken.transferAndCall(
      sdlPool.address,
      toEther(100),
      ethers.utils.defaultAbiCoder.encode(['uint256', 'uint64'], [0, 0])
    )
    //
    // 2. The secondary pool needs to update data to the primary chain but the `controller.shouldUpdate` is false so `performUpkeep` reverts the transaction
    assert.equal(await sdlPool.shouldUpdate(), true)
    assert.equal((await controller.checkUpkeep('0x'))[0], false)
    assert.equal(await controller.shouldUpdate(), false)
    await expect(controller.performUpkeep('0x')).to.be.revertedWith('UpdateConditionsNotMet()')
  })
```

## Impact

`numNewRESDLTokens` and `totalRESDLSupplyChange` updates from the `secondary pool` to the `primary pool` may not be executed, causing the [rewards calculation](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L84C39-L84C72) to be incorrect for each chain.

## Tools used

Manual review

## Recommendations

The [SDLPoolCCIPControllerSecondary::performUpkeep](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66C14-L66C27) function may check if the `secondary pool` has new information and so do not wait for rewards to be available for the `secondary pool`:

```diff
    function performUpkeep(bytes calldata) external {
--      if (!shouldUpdate) revert UpdateConditionsNotMet();
++      if (!shouldUpdate && !ISDLPoolSecondary(sdlPool).shouldUpdate()) revert UpdateConditionsNotMet();

        shouldUpdate = false;
        _initiateUpdate(primaryChainSelector, primaryChainDestination, extraArgs);
    }
```
## <a id='L-02'></a>L-02. Creating/transfering the `lockId` does not verify that the creator implements `onERC721Received` if creator is a contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolPrimary.sol#L280

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L384

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L289

## Summary

Creating the `lockId` does not verify that the creator implements `onERC721Received` if it is a contract. Likewise, the transfer between chains of the `ERC721` does not verify that the recipient is a contract.

## Vulnerability Details

The creation of a `lockId` in the `primary chain` is with the help of the function [SDLPoolPrimary::_storeNewLock](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolPrimary.sol#L280), likewise in the `secondary chain` the function [SDLPoolSecondary::_mintQueuedNewLocks](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L384) is used. These functions assign the corresponding owner within the [`lockOwners[lockId]=owner`](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolPrimary.sol#L289C9-L289C27). On the other hand, when sending an `ERC721` from a `primary chain` to a `secondary chain`, a receiver can be specified and the receiver will get the token on the `secondary chain` with the help of the function [SDLPoolSecondary::handleIncomingRESDL](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L289)

The problem is that the functions don't check that if the owner is a contract, it should implement `onERC721Received`.

When transfering the `lockId`, if the receiver is a contract, the code does verify the implementation of `onERC721Received` with the help of [SDLPool::_checkOnERC721Received](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/base/SDLPool.sol#L492C14-L492C36), therefore when creating the `lockId` the implementation of `onERC721Received` should be verified if the new owner is a contract.

## Impact

If the implementation of `onERC721Received` is not explicitly verified and the `owner` is a contract, it may lose the `ERC721` token.

## Tools used

Manual review

## Recommendations

Add a validation in [SDLPoolPrimary::_storeNewLock](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolPrimary.sol#L280), [SDLPoolSecondary: : _mintQueuedNewLocks](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L384) and [SDLPoolSecondary::handleIncomingRESDL]( https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L289) to check the implementation of `onERC721Received` if the owner is a contract.
## <a id='L-03'></a>L-03. The `extraArgs` parameter used in the `WrappedTokenBridge::_buildCCIPMessage` should be mutable in order to allow compatibility with future CCIP upgrade            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L210

## Summary

The `extraArgs` parameter used in the [WrappedTokenBridge::_buildCCIPMessage](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L210C14-L210C31) function should be mutable in order to allow compatibility with future CCIP upgrade.

## Vulnerability Details

The parameter [`extraArgs`](https://docs.chain.link/ccip/api-reference/client#evmextraargsv1) helps to specify the [`gasLimit`](https://docs.chain.link/ccip/best-practices#setting-gaslimit), in the other hand if there is a transfer of tokens directly to an EOA the value can be zero.

The problem is that inside the function [WrappedTokenBridge::_buildCCIPMessage](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L210C14-L210C31) an immutable `extraArgs` is used ([code line 223](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L223)), which can be harmful since in order to have compatibility with `CCIP` updates it is necessary for `extraArgs` to be mutable.

```solidity
File: WrappedTokenBridge.sol
210:     function _buildCCIPMessage(
211:         address _receiver,
212:         uint256 _amount,
213:         address _feeTokenAddress
214:     ) internal view returns (Client.EVM2AnyMessage memory) {
215:         Client.EVMTokenAmount[] memory tokenAmounts = new Client.EVMTokenAmount[](1);
216:         Client.EVMTokenAmount memory tokenAmount = Client.EVMTokenAmount({token: address(wrappedToken), amount: _amount});
217:         tokenAmounts[0] = tokenAmount;
218: 
219:         Client.EVM2AnyMessage memory evm2AnyMessage = Client.EVM2AnyMessage({
220:             receiver: abi.encode(_receiver),
221:             data: "",
222:             tokenAmounts: tokenAmounts,
223:             extraArgs: "0x",
224:             feeToken: _feeTokenAddress
225:         });
226: 
227:         return evm2AnyMessage;
228:     }
```

According to [CCIP documentation](https://docs.chain.link/ccip/best-practices#using-extraargs):

*The purpose of extraArgs is to allow compatibility with future CCIP upgrades. To get this benefit, make sure that extraArgs is mutable in production deployments.*

## Impact

Compatibility problems may exist in `CCIP` updates if `extraArgs` remains immutable.

## Tools used

Manual review

## Recommendations

Add appropriate modifications to be able to change `extraArgs` if necessary.
## <a id='L-04'></a>L-04. The `SDLPoolSecondary::withdraw` function will be reverted if the chains are not synchronized with each other            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L214

## Summary

The withdraw function in the `secondary chain` will be reverted while the system is not synchronized with the `primary chain`

## Vulnerability Details

The user can [stake `sdl`](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L146) tokens on the `secondary chain` and these tokens will be queued using the [sdlpoolSecondary::_queuewlock](https://github.com/cyfrin/2023-12-stake-link/blob/549b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/sdlpoolsecary.sol#L363) function, then the `secondary chain` sends the update of the new supply of tokens to the `primary chain` with the help of the function [SDLPoolCCIPControllerSecondary::performUpkeep](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66C14-L66C27), finally the user can now execute the queued functions. On the other hand, the user can use the function [SDLPoolSecondary::withdraw](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L214) to be able to obtain the deposited tokens if they have not been locked.

The problem is that as long as the system is not updated (secondary chain <-> primary chain) the withdraw function will be reverted in certain circumstances. Please see the following test:

1. The user deposits `100 tokens` and the `secondary chain` sends the new information to the `primary chain`.
2. The user deposits another `20 tokens` however at this point the secondary chain does NOT send the new information to the primary chain.
3. The user tries to withdraw his `120 tokens` however the withdraw function will be reverted due to an `Arithmetic operation underflowed or overflowed` error.

```js
// File: test/core/sdlPool/sdl-pool-secondary.test.ts
// $ yarn test --grep "codehawks queue and withdraw reverts"
//
  it('codehawks queue and withdraw reverts', async () => {
    // 
    // 1. Mint 100 tokens and lock false. Now balance is 100 and the effective balance is 100
    await mintLock(false)
    assert.equal(fromEther(await sdlToken.balanceOf(sdlPool.address)), 100)
    assert.equal(fromEther(await sdlPool.totalEffectiveBalance()), 100)
    assert.equal(fromEther(await sdlPool.effectiveBalanceOf(accounts[0])), 100)
    assert.deepEqual(
      (await sdlPool.getLockIdsByOwner(accounts[0])).map((v) => v.toNumber()),
      [1]
    )
    //
    // 2. User add 20 more tokens to his lockId (0)
    await sdlToken
      .connect(signers[0])
      .transferAndCall(
        sdlPool.address,
        toEther(20),
        ethers.utils.defaultAbiCoder.encode(['uint256', 'uint64'], [1, 0])
      )

    assert.equal(fromEther(await sdlToken.balanceOf(sdlPool.address)), 120)
    assert.equal(fromEther(await sdlPool.totalEffectiveBalance()), 100)
    assert.equal(fromEther(await sdlPool.effectiveBalanceOf(accounts[0])), 100)
    //
    // 3. The user withdrawal will be reverted by overflow
    await expect(sdlPool.withdraw(1, toEther(120))).to.be.revertedWith("Arithmetic operation underflowed or overflowed outside of an unchecked block");    
  })
```

## Impact

The user cannot obtain his deposited tokens in the `secondary chain` while the system is not updated between `secondary chain` and `primary chain`. There is no specification that the user CANNOT withdraw their tokens if the system is not updated, the user technically did not lock their tokens therefore they should be able to obtain all their tokens.

## Tools used

Manual review

## Recommendations

Allow the user to get all their deposited tokens even if the chains are not synchronized.
## <a id='L-05'></a>L-05. There is no limit to queuing updates on `secondary chains` when a `lockId` is modified            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L428

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L451C14-L451C39

## Summary

In the `SDLPoolSecondary` contract the stake deposits to the same `lockId` are going to be queued using the function [SDLPoolSecondary::_queueLockUpdate](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L428), then once the L2 chain is updated it is possible to execute the queued operations using the [SDLPoolSecondary::_executeQueuedLockUpdates](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L451C14-L451C39). The problem is that the user lockId updating is not limited, affecting the user because it may necessary to execute a large array of queues.

## Vulnerability Details

The user can send tokens to the `SDLPoolSecondary` contract specifying the `lockId`, then the [_queueLockUpdate](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L144) function will be executed, the [SDLPoolSecondary::_queueLockUpdate](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L428) function push the update [code line 436](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L436) to an array:

```solidity
File: SDLPoolSecondary.sol
428:     function _queueLockUpdate(
429:         address _owner,
430:         uint256 _lockId,
431:         uint256 _amount,
432:         uint64 _lockingDuration
433:     ) internal onlyLockOwner(_lockId, _owner) {
434:         Lock memory lock = _getQueuedLockState(_lockId);
435:         LockUpdate memory lockUpdate = LockUpdate(updateBatchIndex, _updateLock(lock, _amount, _lockingDuration));
436:         queuedLockUpdates[_lockId].push(lockUpdate);
437:         queuedRESDLSupplyChange +=
438:             int256(lockUpdate.lock.amount + lockUpdate.lock.boostAmount) -
439:             int256(lock.amount + lock.boostAmount);
440:         if (updateNeeded == 0) updateNeeded = 1;
441: 
442:         emit QueueUpdateLock(_owner, _lockId, lockUpdate.lock.amount, lockUpdate.lock.boostAmount, lockUpdate.lock.duration);
443:     }
```

Once the data is updated accordly to the data from the primary chain (mainnet), it is possible to execute the queue operations using the [SDLPoolSecondary::_executeQueuedLockUpdates](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L451C14-L451C39) function. The problem is that the function iterates over all the user queue updates [code line 461](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L461):

```solidity
File: SDLPoolSecondary.sol
451:     function _executeQueuedLockUpdates(address _owner, uint256[] memory _lockIds) internal updateRewards(_owner) {
...
...
454:         for (uint256 i = 0; i < _lockIds.length; ++i) {
455:             uint256 lockId = _lockIds[i];
456:             _onlyLockOwner(lockId, _owner);
457:             uint256 numUpdates = queuedLockUpdates[lockId].length;
458: 
459:             Lock memory curLockState = locks[lockId];
460:             uint256 j = 0;
461:             while (j < numUpdates) {
...
...
499:             }
...
...
509:         }
510:     }
```

That could be a problem if the user send multiple updates to the same `lockId` because the `_executeQueuedLockUpdates` function will consume more gas.

## Impact

The [SDLPoolSecondary::_executeQueuedLockUpdates](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L451C14-L451C39) execution may cost more gas than expected.


## Tools used

Manual review

## Recommendations

When the user mints new `lockId`, it has a validation that verifies [code line 368](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L368) the user has not exceed a limit:

```solidity
File: SDLPoolSecondary.sol
363:     function _queueNewLock(
364:         address _owner,
365:         uint256 _amount,
366:         uint64 _lockingDuration
367:     ) internal {
368:         if (newLocksByOwner[_owner].length >= queuedNewLockLimit) revert TooManyQueuedLocks();
...
...
```

The same validation should be added when the user updates a `lockId`:

```diff
// File: SDLPoolSecondary.sol
    function _queueLockUpdate(
        address _owner,
        uint256 _lockId,
        uint256 _amount,
        uint64 _lockingDuration
    ) internal onlyLockOwner(_lockId, _owner) {
++      if (updateLocksByOwner[_owner].length >= queuedUpdatesLockLimit) revert TooManyQueuedLocks();
        Lock memory lock = _getQueuedLockState(_lockId);
        LockUpdate memory lockUpdate = LockUpdate(updateBatchIndex, _updateLock(lock, _amount, _lockingDuration));
        queuedLockUpdates[_lockId].push(lockUpdate);
        queuedRESDLSupplyChange +=
            int256(lockUpdate.lock.amount + lockUpdate.lock.boostAmount) -
            int256(lock.amount + lock.boostAmount);
        if (updateNeeded == 0) updateNeeded = 1;

        emit QueueUpdateLock(_owner, _lockId, lockUpdate.lock.amount, lockUpdate.lock.boostAmount, lockUpdate.lock.duration);
    }
```
## <a id='L-06'></a>L-06. There is not protection for fees increment while the `reSDL` is transferring to another chain            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/RESDLTokenBridge.sol#L113-L119

## Summary

When an `reSDL` token is transferring to another chain, it is necessary to pay a gas fee in `native token`, those gas fee can be increased while the transaction is in the mempool causing the user to pay undesired amounts of fees.

## Vulnerability Details

The function [RESDLTokenBridge.transferRESDL()](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/RESDLTokenBridge.sol#L84C14-L84C27) helps to send a `reSDL` token to another chain. In order to send the token to another chain it is necessary to pay a [fee](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/RESDLTokenBridge.sol#L111) which will be calculated depending on the message created.

The problem arises when the user wants to pay the fees in `native token` and there is an increase in the fees while the transaction is in process. It is important to note that `CCIP` allows you to specify an [`extraArgs`](https://docs.chain.link/ccip/api-reference/client#evm2anymessage) in the message that is going to be sent to the other chain:

```solidity
struct EVM2AnyMessage {
  bytes receiver;
  bytes data;
  struct Client.EVMTokenAmount[] tokenAmounts;
  address feeToken;
  bytes extraArgs;
}
```

Inside the `extraArgs` it can be [specified the `gasLimit`](https://docs.chain.link/ccip/api-reference/client#evmextraargsv1) to use. The `gasLimit` is a very important variable to be able to determine the gas fees, as it says in its [documentation](https://docs.chain.link/ccip/best-practices#setting-gaslimit): *It is the main factor in determining the fee to send a message. Unspent gas is not refunded.*. Taking into account that the `extraArgs` can be modified using the function [RESDLTokenBridge::setExtraArgs()](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/RESDLTokenBridge.sol#L161) then the following scenario may arise:

1. The user determines that the fees to sent his tokens are `10e13` native tokens.
2. The transaction is sent and waits in the mempool.
3. The admin changes the values of `extraArgs` and now the fees increase to `20e13` native tokens. This transaction occurs before `step 2`.
4. Now the transaction from `step 2` is executed with an unwanted increase in fees.

Likewise, the fee calculation may be increased by Chainlink while the sending token transaction is waiting in the mempool.

The parameter [`_maxLINKFee`](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/RESDLTokenBridge.sol#L89C17-L89C28) helps determine the maximum in the LINK token, however, there is no validation for the `native token`.

## Impact

The user will pay unwanted fees if those are increased while the sending transaction is waiting to be executed.

## Tools used

Manual review

## Recommendations

Add a parameter that helps specify the maximum fee in native tokens that the user is willing to pay:

```diff
    function transferRESDL(
        uint64 _destinationChainSelector,
        address _receiver,
        uint256 _tokenId,
        bool _payNative,
--      uint256 _maxLINKFee
++      uint256 _maxFee
    ) external payable returns (bytes32 messageId) {
        ...
        ...
        ...

        uint256 fees = IRouterClient(sdlPoolCCIPController.getRouter()).getFee(_destinationChainSelector, evm2AnyMessage);

++      if (fees > _maxFee) revert FeeExceedsLimit();
        if (_payNative) {
            if (fees > msg.value) revert InsufficientFee();
            messageId = sdlPoolCCIPController.ccipSend{value: fees}(_destinationChainSelector, evm2AnyMessage);
            if (fees < msg.value) {
                (bool success, ) = msg.sender.call{value: msg.value - fees}("");
                if (!success) revert TransferFailed();
            }
        } else {
--          if (fees > _maxLINKFee) revert FeeExceedsLimit();
            linkToken.safeTransferFrom(msg.sender, address(sdlPoolCCIPController), fees);
            messageId = sdlPoolCCIPController.ccipSend(_destinationChainSelector, evm2AnyMessage);
        }
        ...
        ...
        ...
    }
```


