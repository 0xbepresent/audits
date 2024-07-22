# Original link
https://github.com/code-423n4/2023-02-ethos-findings/issues/255
# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultV2.sol#L191-L199
https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L123


# Vulnerability details



 The function `updateStrategyAllocBPS()` can cause ActivePool to record an incorrect profit after `setEmergencyExit()` is triggered. 

## Impact
The incorrect profit will cause a large portion of the ActivePool's collateral to be distributed to Treasury, Staking Pool and Stability Pool. Depositors and Stakers can then withdraw the profits, leading to loss of ActivePool's collateral.


### Background
In Ethos Reserve, the Vault rehypothecates the collateral from ActivePool using one or more Strategy, which will deposit the funds in other protocols (e.g. lending pool) to farm for yields. 

Only Guardian and above roles are able to trigger `setEmergencyExit()` on a specific Strategy to force it to exit all its position upon the next harvest, depositing all funds from lending pool into the Vault. How it works, is that `setEmergencyExit()` will trigger Vault to `revokeStrategy()`, setting the strategy's `allocBPS` to `0`. This sets Strategy allocation to `0` and increases Strategy's debt, so that it will repay Vault all the funds.
(see https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L156-L160)

Note that `setEmergencyExit()` is not reversible. I believe this is to protects funds from being re-deployed into the lending pool during an emergency situation (e.g. lending pool hacked or market crash). And it is different from Vault's EmergencyShutdown, which is effected on all Strategies and is reversible.

### Detailed Explanation
The issue is that, Strategist (a lower privilege role than Guardian) is able to reverse `revokeStrategy()` by calling `updateStrategyAllocBPS()` with a non-zero value to increase the strategy allocation. This will lead to a reduction of the strategy's debt and cause an incorrect profit to be recorded when it liquidate its positions in the next harvest. Due to the incorrect profit, a fee will be charged on it and transferred to Treasury, leaving less funds for the Vault. 

https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultV2.sol#L191-L199 

https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L123

Even worse, `updateStrategyAllocBPS()` will cause an increase to vault's totalAllocated value during `harvest()`, while the incorrect profit is transferred to the Vault during harvest. Both of these changes will lead to a discrepancy in the vault's `totalAllocated` and its token balance, causing the vault's total balance to be incorrect and higher than actual. This leads to a higher share price. 

https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultV2.sol#L521

https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultV2.sol#L528

With a higher share price, ActivePool's owned asset value in the Vault will be inflated. This will cause ActivePool to record an incorrect profit in the next  `_rebalance()`,  and distrbute them to Treasury, Staking Pool and Stability Pool. 

https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/ActivePool.sol#L239-L309

Depositors and Stakers can then withdraw the profits, leading to loss of ActivePool's collateral.


## Proof of Concept

Add the following test case to `Ethos-Vault/test/start-test.js`. This shows that ActivePool's asset value will be inflated due to the issue. The next test case will show that inflated asset value will cause ActivePool's `_rebalance()` to record a profit and distribute them to the respective pools, that can be withdrawn.

    it.only('updateStrategyAllocBPS can cause loss of ActivePool collateral during emergency exit', async function () {
      const {vault, strategy, want, wantHolder, strategyAddress, strategist, guardian} = 
        await loadFixture(deployVaultAndStrategyAndGetSigners);
      // intialize guardian account with ETH for gas
      const tx = await strategist.sendTransaction({
        to: guardianAddress,
        value: ethers.utils.parseEther('0.1'),
      });
      await tx.wait();

      // Treasury owned asset value starts with zero
      const treasurySharesBefore = await vault.balanceOf(treasuryAddr);
      const treasuryAssetsBefore = await vault.convertToAssets(treasurySharesBefore);
      expect(treasuryAssetsBefore).to.equal(0);

      // ActivePool deposits 10 WBTC 
      await vault.connect(wantHolder)['deposit(uint256)'](toWantUnit('10'));
      await strategy.harvest();
     
      // Expect ActivePool's owned share and asset to be equal to 10 WBTC as deposited
      const activePoolSharesBefore = await vault.balanceOf(wantHolder.address);
      const activePoolAssetsBefore = await vault.convertToAssets(activePoolSharesBefore);
      expect(activePoolSharesBefore).to.equal(toWantUnit('10'));
      expect(activePoolAssetsBefore).to.equal(toWantUnit('10'));

      /* Guardian calls setEmergencyExit().
      *  This triggers Vault to revokeStrategy() and set strategy's allocBPS to 0.
      *  By design, this will force strategy to exit all its position and 
      *  return funds to vault in the next harvest().
      */
      await strategy.connect(guardian).setEmergencyExit();

      /* Strategist set AllocBPS back to 10000 (100%). 
      *  This will reverse the revokeStrategy() and cause strategy's debt value 
      *  to be reduced in next harvest()
      */
      await vault.connect(strategist).updateStrategyAllocBPS(strategy.address, 10000);

      /* Strategy will liquidate all its position due to emergency exit state.
      * However, it will also record an incorrect profit due to reduced debt value.
      */
      await strategy.harvest();

      // Jump ahead for incorrect profit to unlock
      await moveTimeForward(3600*7);

      // Treasury will gain fees of 1.56 WBTC on the incorrect profit value
      const treasurySharesAfter = await vault.balanceOf(treasuryAddr);
      const treasuryAssetsAfter = await vault.convertToAssets(treasurySharesAfter);
      expect(treasuryAssetsAfter).to.not.equal(treasuryAssetsBefore);
      expect(treasuryAssetsAfter).to.equal(156880733);

      // ActivePool's owned asset value is incorrectly inflated 
      // This is due to increased share price from the incorrect profit and wrong accounting from allocBPS
      const activePoolSharesAfter = await vault.balanceOf(wantHolder.address);
      const activePoolAssetsAfter = await vault.convertToAssets(activePoolSharesAfter);
      expect(activePoolAssetsAfter).to.equal("1743119266");
      expect(activePoolAssetsAfter).to.not.equal(activePoolAssetsBefore);

      /* ActivePool will record a profit of 7.43 WBTC (74% of initial deposit) due to the inflated asset value
      *  In the next ActivePool's _rebalance(), the  incorrect profit will be distributed to Treasury, 
      *  Staking Pool and StabilityPool. 
      *  Depositors and Stakers will be able to withdraw the profits, leading to loss of borrowers's collateral.
      */
      const estimatedActivePoolProfit = activePoolAssetsAfter - activePoolAssetsBefore;
      expect(estimatedActivePoolProfit).to.be.equal(743119266);

    });


Add the following test case to `Ethos-Core/test/PoolsTest.js`. Note that this is an  test independent from the preivous test case just to show that ActivePool will record a profit when the share asset value increases, and the profit will be distributed to the respective pools.

	it.only('simulate incorrect profit to show that _rebalance() called by sendCollateral() will distributes profit', async () => {
		await setReasonableDefaultStateForYielding();

		// Simulate incorrect profit: mint 1 ether to vault.  
		// This will increase the vault share price and inflate the ActivePool's owned asset value.
		await collaterals[0].mint(vaults[0].address, dec(1, 'ether'))

		// Trigger ActivePool's _rebalance() via sendCollateral(). 
		// ActivePool will record a profit due to the inflated owned asset value.
		const sendCollData = th.getTransactionData('sendCollateral(address,address,uint256)', 
		  [collaterals[0].address, alice, web3.utils.toHex(dec(1, 'ether'))])
		await mockBorrowerOperations.forward(activePool.address, sendCollData, { from: owner })

		// The incorrect profit will be distributed to Treasury, StabilityPool and Staking Pool
		assert.equal((await collaterals[0].balanceOf(treasury.address)).toString(), '200000000000000000') // 0.2 ether
		assert.equal((await collaterals[0].balanceOf(stabilityPool.address)).toString(), '300000000000000000') // 0.3 ether
		assert.equal((await collaterals[0].balanceOf(lqtyStaking.address)).toString(), '500000000000000000') // 0.5 ether
	})

## Recommended Mitigation Steps
The fix is to block all changes to strategy's `allocBPS` after `setEmergencyExit()`. 

Since `allocBPS` is already tracked within `ReaperVaultV2.sol`, it is better to refactor and shift `emergencyExit` from `ReaperBaseStrategyv4.sol` to `ReaperVaultV2.StrategyParams`. With that, the fix can simply just to add a check for emergency exit within `updateStrategyAllocBPS()`.

