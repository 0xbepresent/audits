# Original link
https://github.com/code-423n4/2023-02-ethos-findings/issues/318
# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L205
https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L191


# Vulnerability details

## Impact

The [revokeStrategy()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L205) function helps to revoke the strategy setting the ```allocBPS``` to zero. The function can be called only by the ```Strategy``` contract or having at least the ```Guardian``` role.

The [updateStrategyAllocBPS()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L191) function helps to manage the strategy ```allocBPS``` value. The function can be called only by having at least the ```Strategist role```.

The problem here is the ```Strategist role``` can set a strategy allocBPS to zero via the ```updateStrategyAllocBPS()``` which is the same as using the ```revokeStrategy()``` function but the differeance is that the ```revokeStrategy()``` is only accessible from the ```strategy contract``` or having at least the ```guardian role```.

The ```Strategist role``` can revoke an strategy without having at least the ```guardian role``` or being a Strategy.

## Proof of Concept

I created a basic test in ```starter-test.js``` where it is possible to see that the strategy allocBPS is set to zero which is the same as using the ```revokeStrategy()``` function but without having the ```guardian role```. Steps:

1. SuperAdmin creates the strategy.
2. Strategist calls updateStrategyAllocBPS() and set the allocBPS to zero. Technically the strategy is now revoked.
3. The guardian role calls revokeStrategy() but the allocBPS is now zero so the event "StrategyRevoked()" is not emmited.

```javascript
it('strategist role can revoke an strategy', async function () {
    // 1. SuperAdmin creates the strategy
    // 2. Strategist calls updateStrategyAllocBPS() and set the allocBPS to zero. Technically the strategy is now revoked.
    // 3. The guardian role calls revokeStrategy() but the allocBPS is now zero so the
    //    event "StrategyRevoked()" is not emmited
    const { vault, owner, strategist, superAdmin, guardian } = await loadFixture(deployVaultAndStrategyAndGetSigners);
    await owner.sendTransaction({
    to: superAdminAddress,
    value: ethers.utils.parseEther('0.1'),
    });
    await owner.sendTransaction({
    to: strategistAddr,
    value: ethers.utils.parseEther('0.1'),
    });
    await owner.sendTransaction({
    to: guardianAddress,
    value: ethers.utils.parseEther('0.1'),
    });
    //
    // 1. SuperAdmin creates the strategy
    //
    const Strategy = await ethers.getContractFactory('ReaperStrategyGranarySupplyOnly');
    const strategy = await upgrades.deployProxy(
    Strategy,
    [vault.address, strategists, multisigRoles, gWantAddress], {
    kind: 'uups',
    });
    await strategy.deployed();
    await expect(vault.connect(superAdmin).addStrategy(strategy.address, 1000, 1000)).to.not.be.reverted;
    //
    // 2. Strategist calls updateStrategyAllocBPS() and set the allocBPS to zero
    //
    await expect(vault.connect(strategist).updateStrategyAllocBPS(strategy.address, 0)).to.not.be.reverted;
    //
    // 3. The guardian role calls revokeStrategy() but the allocBPS is now zero so the
    // event "StrategyRevoked()" is not emmited
    //
    await expect(vault.connect(guardian).revokeStrategy(strategy.address)).to.emit(vault, "StrategyRevoked")
});
```

## Tools used

VScode

## Recommended Mitigation Steps

Use a specific strategy state to have a fully revoked strategy in the ```revokeStrategy()``` function. I see the ```revokeStrategy()``` can be called if there is an [emergency exit](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L159) in the Strategy so maybe create a ```revoked``` field in the [StrategyParams struct](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L25) could be a good option.