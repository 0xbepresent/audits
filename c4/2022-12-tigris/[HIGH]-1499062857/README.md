# Original link
https://github.com/code-423n4/2022-12-tigris-findings/issues/264
# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Lock.sol#L19
https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Lock.sol#L84


# Vulnerability details

## Impact

The [extendLock()](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Lock.sol#L84) function helps to change the BondNFT lock parameters (amount, period). The problem is ```totalLocked``` is not increased.

The ```totalLocked``` is a public variable, so there may be wrong information for those who consult that specific information:
- Users
- Other protocols
- Bots

## Proof of Concept

I created this test in ```test/09.Bond.js``` where you can see the [extendLock()](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Lock.sol#L84) does not increase the ```totalLocked``` variable::

```javascript
// npx hardhat test --grep "Adding amount and time to lock the totalLocked has not changed" test/09.Bonds.js
it("Adding amount and time to lock the totalLocked has not changed", async function () {
    await stabletoken.connect(owner).mintFor(user.address, ethers.utils.parseEther("100"));
    await lock.connect(user).lock(StableToken.address, ethers.utils.parseEther("100"), 10);

    await network.provider.send("evm_increaseTime", [432000]); // Skip 5 days
    await network.provider.send("evm_mine");

    // Get the Bond 1
    [id, _owner, asset, amount, mintEpoch, mintTime, expireEpoch, pending, shares, period, expired] = await bond.idToBond(1);
    expect(amount).to.be.equals(ethers.utils.parseEther("100"));
    expect(period).to.be.equals(10);

    // Total locked is the amount == 100
    expect(await lock.totalLocked(asset)).to.be.equals(ethers.utils.parseEther("100"));

    // ExtendLock for 100 more and 10 more period.
    await stabletoken.connect(owner).mintFor(user.address, ethers.utils.parseEther("100"));
    await lock.connect(user).extendLock(1, ethers.utils.parseEther("100"), 10);  // add 100, 10

    //Total Locked is now the amount before + the new amount == 200
    expect(await lock.totalLocked(asset)).to.be.equals(ethers.utils.parseEther("200"));

    [id, _owner, asset, amount, mintEpoch, mintTime, expireEpoch, pending, shares, period, expired] = await bond.idToBond(1);
    expect(amount).to.be.equals(ethers.utils.parseEther("200"));  // new amount
    expect(period).to.be.equals(20); // New period
    //Total Locked should be amount before + the new amount == 200
    expect(await lock.totalLocked(asset)).to.be.equals(ethers.utils.parseEther("200"));
});
```

## Tools used
VsCode/Hardhat

## Recommended Mitigation Steps
Add the ```totalLocked``` incremention in the ```Lock.sol::extendLock()``` function