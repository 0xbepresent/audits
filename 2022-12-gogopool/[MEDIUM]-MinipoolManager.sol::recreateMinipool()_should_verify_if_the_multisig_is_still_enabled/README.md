# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/343
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L444
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L127
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MultisigManager.sol#L68
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MultisigManager.sol#L80


# Vulnerability details

## Impact

The ```MinipoolManager.sol::createMinipool()``` function assign a "enabled multisig" with the [multisigManager.requireNextActiveMultisig()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L236) function, so the minipool has a enabled multisig assigned.

It is possible that in the meantime where the minipool/node/validator is running, the multisig could be disabled by the [MultisigManager.sol::disableMultisig()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MultisigManager.sol#L68) function.

For some reason (maybe a compromised multisig) the multisig could be [disabled](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MultisigManager.sol#L68) and the multisig should not recreate the minipool and get access to the funds via [claimAndInitiateStaking()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L324) function.

The funds can be compromised by a disabled multisig.

## Proof of Concept

The ```recreateMinipool()``` only get the multisig assigned to the nodeID but ```onlyValidMultisig()``` does not validate if the multisig is still enabled. The next scenario is possible:

1. The minipool is assigned with a valid multisig.
2. While the node is running the multisig is disabled.
3. Now the compromised multisig recreate the minipool.
4. The compromised multisig calls claimAndInitiateStaking() and get the funds.

```solidity
File: MinipoolManager.sol
444: 	function recreateMinipool(address nodeID) external whenNotPaused {
445: 		int256 minipoolIndex = onlyValidMultisig(nodeID);
446: 		requireValidStateTransition(minipoolIndex, MinipoolStatus.Prelaunch);
447: 		Minipool memory mp = getMinipool(minipoolIndex);
```

```solidity
File: MinipoolManager.sol
127: 	function onlyValidMultisig(address nodeID) private view returns (int256) {
128: 		int256 minipoolIndex = requireValidMinipool(nodeID);
129: 
130: 		address assignedMultisig = getAddress(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".multisigAddr")));
131: 		if (msg.sender != assignedMultisig) {
132: 			revert InvalidMultisigAddress();
133: 		}
134: 		return minipoolIndex;
135: 	}
```

## Tools used

Manual review

## Recommended Mitigation Steps

The [recreateMinipool()](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L444) should validate  if the assigned multisig is still enabled and assign another enabled multisig if it is necessary with the ``` multisigManager.requireNextActiveMultisig()``` function.