# Original link
https://github.com/code-423n4/2024-01-salty-findings/issues/752
# Lines of code

https://github.com/code-423n4/2024-01-salty/blob/aab6bbc6fe49d4dd37becc7bbd0c847ec4a7c1e6/src/dao/DAO.sol#L157-L164
https://github.com/code-423n4/2024-01-salty/blob/aab6bbc6fe49d4dd37becc7bbd0c847ec4a7c1e6/src/pools/PoolStats.sol#L51-L55


# Vulnerability details

## Impact

When a pool that has been excluded from the whitelist is added again, it can receive liquidity rewards based on the previous `_arbitrageProfits`.

## Proof of Concept

When unwhitelisting, `_arbitrageProfits` is not cleared. When added to the whitelist again, rewards can be distributed unfairly due to the remaining `_arbitrageProfits`.

Let's assume there are A/WETH, A/WBTC, B/WETH, B/WBTC, WETH/WBTC pools. A swap occurred in each of the A/WETH, B/WETH pools, generating 1 ether of arbitrage profit. If the liquidity rewards are settled at this time, each pool can receive liquidity rewards at a ratio of A/WETH: A/WBTC: B/WETH: B/WBTC: WETH/WBTC = 1: 1: 1: 1: 2. This is because they can take as much as the proportion of profits generated in each pool.

```solidity
function _calculateArbitrageProfits( bytes32[] memory poolIDs, uint256[] memory _calculatedProfits ) internal view
{
    for( uint256 i = 0; i < poolIDs.length; i++ )
    {
        // references poolID(arbToken2, arbToken3) which defines the arbitage path of WETH->arbToken2->arbToken3->WETH
        bytes32 poolID = poolIDs[i];

        // Split the arbitrage profit between all the pools that contributed to generating the arbitrage for the referenced pool.
@>      uint256 arbitrageProfit = _arbitrageProfits[poolID] / 3;
        if ( arbitrageProfit > 0 )
        {
            ArbitrageIndicies memory indicies = _arbitrageIndicies[poolID];

            if ( indicies.index1 != INVALID_POOL_ID )
                _calculatedProfits[indicies.index1] += arbitrageProfit;

            if ( indicies.index2 != INVALID_POOL_ID )
                _calculatedProfits[indicies.index2] += arbitrageProfit;

            if ( indicies.index3 != INVALID_POOL_ID )
                _calculatedProfits[indicies.index3] += arbitrageProfit;
        }
    }
}

function profitsForWhitelistedPools() external view returns (uint256[] memory _calculatedProfits)
{
@>  bytes32[] memory poolIDs = poolsConfig.whitelistedPools();

    _calculatedProfits = new uint256[](poolIDs.length);
@>  _calculateArbitrageProfits( poolIDs, _calculatedProfits );
}
```

Suppose you unwhitelist B tokens without settling rewards. The B/WETH and B/WBTC pools will be unwhitelisted. Since they are not included in the whitelist, they cannot be involved in reward settlements, so even if the someone calls `performUpkeep` , their `_arbitrageProfits` remain the same.

```solidity
function clearProfitsForPools() external
{
    require(msg.sender == address(exchangeConfig.upkeep()), "PoolStats.clearProfitsForPools is only callable from the Upkeep contract" );

@>  bytes32[] memory poolIDs = poolsConfig.whitelistedPools();

    for( uint256 i = 0; i < poolIDs.length; i++ )
        _arbitrageProfits[ poolIDs[i] ] = 0;
}
```

Time has passed and B token is added to the whitelist again, so that the B/WETH, B/WBTC pools are registered on the whitelist. Since the past `_arbitrageProfits` were not cleared, 1 ether remains in B/WETH pool's `_arbitrageProfits`. The rewards for the other pools have already been settled and `_arbitrageProfits` has been reset.

After that, the A/WBTC pool makes new arbitrage profit of 0.1 ether. If the reward is settled in this state, they will receive rewards at a ratio of A/WETH: A/WBTC: B/WETH: B/WBTC: WETH/WBTC = 1: 1: 10: 10: 11.

This is the PoC. You can add it to the DAO.t.sol file and test it.

```solidity
function testPoCArbitrageProfitsNotCleared() public
{
	vm.prank(DEPLOYER);
	salt.approve(address(liquidityRewardsEmitter), type(uint256).max);
	
	IERC20 token1 = new TestERC20("TEST", 18);
	IERC20 token2 = new TestERC20("TEST", 18);
	
	token1.transfer(DEPLOYER, 1000000 ether);
	token2.transfer(DEPLOYER, 1000000 ether);
	token1.transfer(alice, 1000000 ether);
	token2.transfer(alice, 1000000 ether);
	token1.transfer(bob, 1000000 ether);
	token2.transfer(bob, 1000000 ether);
	
	// --- add whitelist ---
	
	vm.startPrank(alice);
	staking.stakeSALT( 1000000 ether );
	
	salt.transfer( address(dao), 2000000 ether );
	
	uint256 ballotId = proposals.proposeTokenWhitelisting( token1, "", "" );
	_voteForAndFinalizeBallot(ballotId, Vote.YES);
	
	ballotId = proposals.proposeTokenWhitelisting( token2, "", "" );
	_voteForAndFinalizeBallot(ballotId, Vote.YES);
	vm.stopPrank();
	
	// --- Add liquidity ---
	vm.startPrank(DEPLOYER);
	token1.approve(address(collateralAndLiquidity), type(uint256).max);
	token2.approve(address(collateralAndLiquidity), type(uint256).max);
	weth.approve(address(collateralAndLiquidity), type(uint256).max);
	wbtc.approve(address(collateralAndLiquidity), type(uint256).max);
	// Adds token1/WETH, token1/WBTC liquidity
	collateralAndLiquidity.depositLiquidityAndIncreaseShare(token1, weth, 1000 ether, 10 ether, 0, block.timestamp, false);
	collateralAndLiquidity.depositLiquidityAndIncreaseShare(token1, wbtc, 1000 ether, 10 * 10**8, 0, block.timestamp, false);
	// Adds token2/WETH, token2/WBTC liquidity
	collateralAndLiquidity.depositLiquidityAndIncreaseShare(token2, weth, 1000 ether, 10 ether, 0, block.timestamp, false);
	collateralAndLiquidity.depositLiquidityAndIncreaseShare(token2, wbtc, 1000 ether, 10 * 10**8, 0, block.timestamp, false);
	// Adds WETH/WBTC liquidity
	collateralAndLiquidity.depositCollateralAndIncreaseShare(1000 * 10**8, 1000 ether, 0, block.timestamp, false);
	
	vm.stopPrank();
	
	
		// --- arbitrage profit ---
	vm.startPrank(bob);
	token1.approve(address(pools), type(uint256).max);
	token2.approve(address(pools), type(uint256).max);
	pools.depositSwapWithdraw(token1, wbtc, 1 ether, 0, block.timestamp);
	pools.depositSwapWithdraw(token2, wbtc, 1 ether, 0, block.timestamp);
	vm.stopPrank();
	
	uint256 token2_wbtc_index;
	uint256 token2_weth_index;
	
	bytes32[] memory poolIDs = poolsConfig.whitelistedPools();
	
	for(uint256 i = 0; i < poolIDs.length; i ++){
		if(poolIDs[i] == PoolUtils._poolID(token2, wbtc)){
			token2_wbtc_index = i;
			continue;
		}
		if(poolIDs[i] == PoolUtils._poolID(token2, weth)){
			token2_weth_index = i;
			continue;
		}
	}
	assertTrue(token2_wbtc_index != 0 && token2_weth_index != 0, "wrong index");
	
	uint256[] memory old_profits = pools.profitsForWhitelistedPools();
	
	// --- unwhitelist pool ---
	
	vm.startPrank(alice);
	ballotId = proposals.proposeTokenUnwhitelisting( token2, "", "" );
	_voteForAndFinalizeBallot(ballotId, Vote.YES);
	vm.stopPrank();
	
	// Check for the effects of the vote
	assertFalse( poolsConfig.tokenHasBeenWhitelisted(token2, wbtc, weth), "Token should not be whitelisted" );
	
	// user calls upkeep, the old reward cleared (but token2's reward is not cleared)
	vm.prank(address(upkeep));
	pools.clearProfitsForPools();
	
	
	// --- re-whitelist ---
	
	vm.startPrank(alice);
	ballotId = proposals.proposeTokenWhitelisting( token2, "", "" );
	_voteForAndFinalizeBallot(ballotId, Vote.YES);
	vm.stopPrank();
	
	poolIDs = poolsConfig.whitelistedPools();
	
	uint256 new_token2_wbtc_index;
	uint256 new_token2_weth_index;
	
	for(uint256 i = 0; i < poolIDs.length; i ++){
		if(poolIDs[i] == PoolUtils._poolID(token2, wbtc)){
			new_token2_wbtc_index = i;
			continue;
		}
		if(poolIDs[i] == PoolUtils._poolID(token2, weth)){
			new_token2_weth_index = i;
			continue;
		}
	}
	assertTrue(new_token2_wbtc_index != 0 && new_token2_weth_index != 0, "wrong index");
	
	// --- old and new reward for token2 pool are same ---
	uint256[] memory new_profits = pools.profitsForWhitelistedPools();
	
	assertEq(old_profits[token2_wbtc_index], new_profits[new_token2_wbtc_index], "old _arbitrageProfits remain");
	assertEq(old_profits[token2_weth_index], new_profits[new_token2_weth_index], "old _arbitrageProfits remain");

}
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Before finalizing the unwhitelist ballot, user should call `performUpkeep` first to force the rewards to be settled. Check if `_arbitrageProfits` is cleared.

```diff
function _executeApproval( Ballot memory ballot ) internal
{
    if ( ballot.ballotType == BallotType.UNWHITELIST_TOKEN )
    {
+       require(pools._arbitrageProfits(poolId_wbtc) == 0, "not cleared yet");
+       require(pools._arbitrageProfits(poolId_weth) == 0, "not cleared yet");
        // All tokens are paired with both WBTC and WETH so unwhitelist those pools
        poolsConfig.unwhitelistPool( pools, IERC20(ballot.address1), exchangeConfig.wbtc() );
        poolsConfig.unwhitelistPool( pools, IERC20(ballot.address1), exchangeConfig.weth() );

        emit UnwhitelistToken(IERC20(ballot.address1));
    }
    ...
}

```


## Assessed type

Other