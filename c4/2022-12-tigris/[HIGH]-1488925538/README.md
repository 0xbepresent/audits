# Original link
https://github.com/code-423n4/2022-12-tigris-findings/issues/23
# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Lock.sol#L10
https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Lock.sol#L61-L76
https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Lock.sol#L84-L92
https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Lock.sol#L98-L105


# Vulnerability details

## Impact
The `Lock` contract ([https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Lock.sol#L10](https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Lock.sol#L10)) allows end-users to interact with bonds.  

There are two functions that allow to lock some amount of assets. The first function is `Lock.lock` ([https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Lock.sol#L61-L76](https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Lock.sol#L61-L76)) which creates a new bond. The second function is `Lock.extendLock` ([https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Lock.sol#L84-L92](https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Lock.sol#L84-L92)). This function extends the lock for some `_period` and / or increases the locked amount by some `_amount`.  

The issue is that the `Lock.extendLock` function does not increase the value in `totalLocked[_asset]`. This however is necessary because `totalLocked[_asset]` is reduced when `Lock.release` ([https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Lock.sol#L98-L105](https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Lock.sol#L98-L105)) is called.  

Therefore only the amount of assets deposited via `Lock.lock` can be released again. The amount of assets deposited using `Lock.extendLock` can never be released again because reducing `totalLocked[_asset]` will cause a revert due to underflow.  

So the amount of assets deposited using `Lock.extendLock` is lost.  

## Proof of Concept
1. User A calls `Lock.lock` to lock a certain `_amount` (amount1) of `_asset` for a certain `_period`.
2. User A calls then `Lock.extendLock` and increases the locked amount of the bond by some amount2
3. User A waits until the bond has expired
4. User A calls `Lock.release`. This function calculates `totalLocked[asset] -= lockAmount;`. Which will cause a revert because the value of `totalLocked[asset]` is only amount1

You can add the following test to the `Bonds` test in `Bonds.js`:  
```javascript
describe("ReleaseUnderflow", function () {
    it("release can cause underflow", async function () {
        await stabletoken.connect(owner).mintFor(user.address, ethers.utils.parseEther("110"));
        // Lock 100 for 9 days
        await lock.connect(user).lock(StableToken.address, ethers.utils.parseEther("100"), 9);

        await bond.connect(owner).setManager(lock.address);

        await stabletoken.connect(user).approve(lock.address, ethers.utils.parseEther("10"));

        // Lock another 10
        await lock.connect(user).extendLock(1, ethers.utils.parseEther("10"), 0);

        await network.provider.send("evm_increaseTime", [864000]); // Skip 10 days
        await network.provider.send("evm_mine");

        // Try to release 110 after bond has expired -> Underflow
        await lock.connect(user).release(1);
    });
});
```
Run it with `npx hardhat test --grep "release can cause underflow"`.  
You can see that it fails because it causes an underflow.  

## Tools Used
VSCode

## Recommended Mitigation Steps
Add `totalLocked[_asset] += amount` to the `Lock.extendLock` function.  