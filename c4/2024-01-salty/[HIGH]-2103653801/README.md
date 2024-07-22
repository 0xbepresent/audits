# Original link
https://github.com/code-423n4/2024-01-salty-findings/issues/312
# Lines of code

https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/stable/CollateralAndLiquidity.sol#L140
https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/stable/CollateralAndLiquidity.sol#L70


# Vulnerability details

## Impact

The [CollateralAndLiquidity](https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/stable/CollateralAndLiquidity.sol) contract contains a critical vulnerability that allows a user undergoing liquidation to evade the process by manipulating the `user.cooldownExpiration` variable. This manipulation is achieved through the [CollateralAndLiquidity::depositCollateralAndIncreaseShare](https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/stable/CollateralAndLiquidity.sol#L70) function, specifically within the [StakingRewards::_increaseUserShare](https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/staking/StakingRewards.sol#L57) function (code line [#70](https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/staking/StakingRewards.sol#L70)):

```solidity
File: StakingRewards.sol
57: 	function _increaseUserShare( address wallet, bytes32 poolID, uint256 increaseShareAmount, bool useCooldown ) internal
58: 		{
59: 		require( poolsConfig.isWhitelisted( poolID ), "Invalid pool" );
60: 		require( increaseShareAmount != 0, "Cannot increase zero share" );
61: 
62: 		UserShareInfo storage user = _userShareInfo[wallet][poolID];
63: 
64: 		if ( useCooldown )
65: 		if ( msg.sender != address(exchangeConfig.dao()) ) // DAO doesn't use the cooldown
66: 			{
67: 			require( block.timestamp >= user.cooldownExpiration, "Must wait for the cooldown to expire" );
68: 
69: 			// Update the cooldown expiration for future transactions
70: 			user.cooldownExpiration = block.timestamp + stakingConfig.modificationCooldown();
71: 			}
72: 
73: 		uint256 existingTotalShares = totalShares[poolID];
74: 
75: 		// Determine the amount of virtualRewards to add based on the current ratio of rewards/shares.
76: 		// The ratio of virtualRewards/increaseShareAmount is the same as totalRewards/totalShares for the pool.
77: 		// The virtual rewards will be deducted later when calculating the user's owed rewards.
78:         if ( existingTotalShares != 0 ) // prevent / 0
79:         	{
80: 			// Round up in favor of the protocol.
81: 			uint256 virtualRewardsToAdd = Math.ceilDiv( totalRewards[poolID] * increaseShareAmount, existingTotalShares );
82: 
83: 			user.virtualRewards += uint128(virtualRewardsToAdd);
84: 	        totalRewards[poolID] += uint128(virtualRewardsToAdd);
85: 	        }
86: 
87: 		// Update the deposit balances
88: 		user.userShare += uint128(increaseShareAmount);
89: 		totalShares[poolID] = existingTotalShares + increaseShareAmount;
90: 
91: 		emit UserShareIncreased(wallet, poolID, increaseShareAmount);
92: 		}
```

Malicious user can perform front-running of the `liquidation` function by depositing small amounts of tokens to his position, incrementing the `user.cooldownExpiration` variable. Consequently, the execution of the `liquidation` function will be reverted with the error message `Must wait for the cooldown to expire.` This vulnerability could lead to attackers evading liquidation, potentially causing the system to enter into debt as liquidations are avoided.

## Proof of Concept

A test case, named `testUserLiquidationMayBeAvoided`, has been created to demonstrate the potential misuse of the system. The test involves the following steps:

1. User Alice deposits and borrow the maximum amount.
2. The collateral price crashes.
3. Alice maliciously front-runs the `liquidation` execution by depositing a the minimum amount using the `collateralAndLiquidity::depositCollateralAndIncreaseShare` function.
4. The `liquidation` transaction is reverted by "Must wait for the cooldown to expire" error.

```solidity
// Filename: src/stable/tests/CollateralAndLiquidity.t.sol:TestCollateral
// $ forge test --match-test "testUserLiquidationMayBeAvoided" --rpc-url https://yoururl -vv
//
    function testUserLiquidationMayBeAvoided() public {
        // Liquidatable user can avoid liquidation
        //
		// Have bob deposit so alice can withdraw everything without DUST reserves restriction
        _depositHalfCollateralAndBorrowMax(bob);
        //
        // 1. Alice deposit and borrow the max amount
        // Deposit and borrow for Alice
        _depositHalfCollateralAndBorrowMax(alice);
        // Check if Alice has a position
        assertTrue(_userHasCollateral(alice));
        //
        // 2. Crash the collateral price
        _crashCollateralPrice();
        vm.warp( block.timestamp + 1 days );
        //
        // 3. Alice maliciously front run the liquidation action and deposit a DUST amount
        vm.prank(alice);
		collateralAndLiquidity.depositCollateralAndIncreaseShare(PoolUtils.DUST + 1, PoolUtils.DUST + 1, 0, block.timestamp, false );
        //
        // 4. The function alice liquidation will be reverted by "Must wait for the cooldown to expire"
        vm.expectRevert( "Must wait for the cooldown to expire" );
        collateralAndLiquidity.liquidateUser(alice);
    }
```

## Tools used

Manual review

## Recommended Mitigation Steps

Consider modifying the [liquidation](https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/stable/CollateralAndLiquidity.sol#L154) function as follows:

```diff
	function liquidateUser( address wallet ) external nonReentrant
		{
		require( wallet != msg.sender, "Cannot liquidate self" );

		// First, make sure that the user's collateral ratio is below the required level
		require( canUserBeLiquidated(wallet), "User cannot be liquidated" );

		uint256 userCollateralAmount = userShareForPool( wallet, collateralPoolID );

		// Withdraw the liquidated collateral from the liquidity pool.
		// The liquidity is owned by this contract so when it is withdrawn it will be reclaimed by this contract.
		(uint256 reclaimedWBTC, uint256 reclaimedWETH) = pools.removeLiquidity(wbtc, weth, userCollateralAmount, 0, 0, totalShares[collateralPoolID] );

		// Decrease the user's share of collateral as it has been liquidated and they no longer have it.
--		_decreaseUserShare( wallet, collateralPoolID, userCollateralAmount, true );
++		 _decreaseUserShare( wallet, collateralPoolID, userCollateralAmount, false );

		// The caller receives a default 5% of the value of the liquidated collateral.
		uint256 rewardPercent = stableConfig.rewardPercentForCallingLiquidation();

		uint256 rewardedWBTC = (reclaimedWBTC * rewardPercent) / 100;
		uint256 rewardedWETH = (reclaimedWETH * rewardPercent) / 100;

		// Make sure the value of the rewardAmount is not excessive
		uint256 rewardValue = underlyingTokenValueInUSD( rewardedWBTC, rewardedWETH ); // in 18 decimals
		uint256 maxRewardValue = stableConfig.maxRewardValueForCallingLiquidation(); // 18 decimals
		if ( rewardValue > maxRewardValue )
			{
			rewardedWBTC = (rewardedWBTC * maxRewardValue) / rewardValue;
			rewardedWETH = (rewardedWETH * maxRewardValue) / rewardValue;
			}

		// Reward the caller
		wbtc.safeTransfer( msg.sender, rewardedWBTC );
		weth.safeTransfer( msg.sender, rewardedWETH );

		// Send the remaining WBTC and WETH to the Liquidizer contract so that the tokens can be converted to USDS and burned (on Liquidizer.performUpkeep)
		wbtc.safeTransfer( address(liquidizer), reclaimedWBTC - rewardedWBTC );
		weth.safeTransfer( address(liquidizer), reclaimedWETH - rewardedWETH );

		// Have the Liquidizer contract remember the amount of USDS that will need to be burned.
		uint256 originallyBorrowedUSDS = usdsBorrowedByUsers[wallet];
		liquidizer.incrementBurnableUSDS(originallyBorrowedUSDS);

		// Clear the borrowedUSDS for the user who was liquidated so that they can simply keep the USDS they previously borrowed.
		usdsBorrowedByUsers[wallet] = 0;
		_walletsWithBorrowedUSDS.remove(wallet);

		emit Liquidation(msg.sender, wallet, reclaimedWBTC, reclaimedWETH, originallyBorrowedUSDS);
		}
```

This modification ensures that the `user.cooldownExpiration` expiration check does not interfere with the `liquidation` process, mitigating the identified security risk.


## Assessed type

Invalid Validation