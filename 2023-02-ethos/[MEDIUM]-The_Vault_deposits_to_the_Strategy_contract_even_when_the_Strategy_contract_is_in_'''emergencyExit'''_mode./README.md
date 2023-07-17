# Original link
https://github.com/code-423n4/2023-02-ethos-findings/issues/367
# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L156
https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L109
https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L493


# Vulnerability details

## Impact

The [ReaperBaseStrategyv4.sol::setEmergencyExit()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L156) helps to put the Strategy contract in ```emergencyExit``` mode which means that the Strategy will exit its position upon the next harvest, depositing all funds into the Vault as quickly as is reasonable given on-chain conditions.

The problem is that the Strategy contract still receives funds even the Strategy is set on ```emergencyExit``` mode. That is incorrect because in the ```emergecyExit``` mode the Strategy should transfer all his funds to the Vault in order to keep the funds safe.

Please follow the next execution path:
1. The Guardian calls [setEmergencyExit()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L156) and sets the ```emergencyExit``` mode.
2. The [revokeStrategy()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L159) is called which means that the [strategy allocBPS is assigned to zero](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#LL215C31-L215C39).
3. For some reason, the Strategist role calls the [updateStrategyAllocBPS()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L191) which means that the strategy now has a new ```allocBPS``` assigned.
4. The Keeper calls the [harvest()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L109) function and because the Strategy is in ```emergencyExit``` mode, the Strategy will [liquidate all his positions](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L119), then the [report()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L134) function is called.
5. The ```report()``` function is called and the function will [get the available capital](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L508) which means that it will get the available capital for the Strategy. In the ```step 3``` the strategist re-assigned the ```allocBPS``` so now the ```availableCapital()``` function will [get some funds to the Strategy](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L231).
6. If there are some funds to the Strategy, [the vault will deposit the credit to the Strategy contract](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L526).


The Strategy contract will have funds even when it was set to ```emergencyExit``` mode. That is not the desired behaivor because the Strategy contract is in ```emergencyExit``` mode and the Strategy contract should have zero funds.

## Proof of Concept

I created a test in ```starter-test.js```. Test steps:

1. Deposit to the vault
2. Guardian sets the vault to emergency exit via ```setEmergencyShutdown()``` function.
3. Strategist for some reason calls ```updateStrategyAllocBPS()``` in order to set the ```strategy allocBPS```.
4. There are another deposit to the vault.
5. Keeper calls the harvest() function in ```emergencyExit``` mode but the Strategy contract balance will still have money.

```javascript
it(['Strategy should handle emergency exit with re-allocBPS', async function () {
    // 1. Deposit to the vault
    // 2. Guardian sets the vault to emergency exit via "setEmergencyShutdown()"
    // 3. Strategist for some reason calls updateStrategyAllocBPS() in order to set the allocBPS
    // 4. There are another deposit to the vault.
    // 5. Keeper calls the harvest() function in emergencyExit mode but the Strategy contract balance will still have money.
    const { vault, strategy, want, wantHolder, strategist, guardian, owner } = await loadFixture(deployVaultAndStrategyAndGetSigners);
    const tx = await strategist.sendTransaction({
    to: guardianAddress,
    value: ethers.utils.parseEther('0.1'),
    });
    await tx.wait();
    const tx2 = await owner.sendTransaction({
    to: strategistAddr,
    value: ethers.utils.parseEther('0.1'),
    });
    await tx2.wait();
    //
    // 1. Deposit to the vault.
    //
    await vault.connect(wantHolder)['deposit(uint256)'](toWantUnit('10'));
    await strategy.harvest();
    await moveTimeForward(3600);
    let vaultBalance = await want.balanceOf(vault.address);
    expect(vaultBalance).to.be.gte(toWantUnit('1'));
    let stratBalance = await strategy.balanceOf();
    expect(stratBalance).to.be.gte(toWantUnit('9'));
    //
    // 2. Guardian sets the vault to emergency exit via "setEmergencyShutdown()".
    //
    await strategy.connect(guardian).setEmergencyExit();
    //
    // 3. Strategist for some reason calls updateStrategyAllocBPS() in order to set the allocBPS.
    //
    await vault.connect(strategist).updateStrategyAllocBPS(strategy.address, 9000);
    //
    // 4. There are another deposit to the vault.
    //
    await vault.connect(wantHolder)['deposit(uint256)'](toWantUnit('10'));
    vaultBalance = await want.balanceOf(vault.address);
    expect(vaultBalance).to.be.gte(toWantUnit('11'));
    //
    // 5. Keeper calls the harvest() function in emergencyExit mode but the Strategy contract balance will still have money.
    //
    await strategy.harvest();
    stratBalance = await strategy.balanceOf();
    expect(stratBalance).not.to.be.equal(0); // The Strategy contract balance IS NOT ZERO
    expect(stratBalance).to.be.equal(toWantUnit('9')); // The Strategy contract balance IS NOT ZERO
    vaultBalance = await want.balanceOf(vault.address);
    expect(vaultBalance).to.be.gte(toWantUnit('11'));
});]
```

## Tools used

VScode/JS

## Recommended Mitigation Steps

The recommendation is to have a "revoked" field in the [StrategyParams struct](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L25), so it can be filled to ```strategies[_strategy].revoked = true``` in the [revokeStrategy()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L205) function and the [availableCapital()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L227) can check if the strategy is revoked.