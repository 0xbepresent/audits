# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/673
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Staking.sol#L328-L332
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ClaimNodeOp.sol#L89-L114
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ClaimProtocolDAO.sol#L20-L35


# Vulnerability details

## Impact

The `whenNotPaused` modifier is used to pause minipool creation and staking/withdrawing GGP. However, there are several cases this modifier could be bypassed, which breaks the intended admin control function and special mode.


## Proof of Concept

### `stake()`

In paused mode, no more `stakeGGP()` is allowed, 
```solidity
File: contract/Staking.sol
319: 	function stakeGGP(uint256 amount) external whenNotPaused {
320: 		// Transfer GGP tokens from staker to this contract
321: 		ggp.safeTransferFrom(msg.sender, address(this), amount);
322: 		_stakeGGP(msg.sender, amount);
323: 	}
```

However, `restakeGGP()` is still available, which potentially violate the purpose of pause mode.
```solidity
File: contract/Staking.sol
328: 	function restakeGGP(address stakerAddr, uint256 amount) public onlySpecificRegisteredContract("ClaimNodeOp", msg.sender) {
329: 		// Transfer GGP tokens from the ClaimNodeOp contract to this contract
330: 		ggp.safeTransferFrom(msg.sender, address(this), amount);
331: 		_stakeGGP(stakerAddr, amount);
332: 	}
```

### `withdraw()`

In paused mode, no more `withdrawGGP()` is allowed, 
```solidity
File: contract/Staking.sol
358: 	function withdrawGGP(uint256 amount) external whenNotPaused {

373: 		vault.withdrawToken(msg.sender, ggp, amount);
```

However, `claimAndRestake()` is still available, which can withdraw from the vault.
```solidity
File: contract/ClaimNodeOp.sol
089: 	function claimAndRestake(uint256 claimAmt) external {

103: 		if (restakeAmt > 0) {
104: 			vault.withdrawToken(address(this), ggp, restakeAmt);
105: 			ggp.approve(address(staking), restakeAmt);
106: 			staking.restakeGGP(msg.sender, restakeAmt);
107: 		}
108: 
109: 		if (claimAmt > 0) {
110: 			vault.withdrawToken(msg.sender, ggp, claimAmt);
111: 		}
```

THe function `spend()` can also ignore the pause mode to withdraw from the vault. But this is a guardian function. It could be intended behavior.
```solidity
File: contract/ClaimProtocolDAO.sol
20: 	function spend(
21: 		string memory invoiceID,
22: 		address recipientAddress,
23: 		uint256 amount
24: 	) external onlyGuardian {

32: 		vault.withdrawToken(recipientAddress, ggpToken, amount);
```



## Tools Used
Manual analysis.

## Recommended Mitigation Steps

- add the `whenNotPaused` modifier to `restakeGGP()` and `claimAndRestake()`
- maybe also for guardian function `spend()`.