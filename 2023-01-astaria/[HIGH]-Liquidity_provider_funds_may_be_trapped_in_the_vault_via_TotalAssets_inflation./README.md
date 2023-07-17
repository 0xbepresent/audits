# Original link
https://github.com/code-423n4/2023-01-astaria-findings/issues/319
# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L332
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L117
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L275
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L359
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L490


# Vulnerability details

## Impact

The liquidity provider can lend to public vaults in order to finance vaults. The problem is that the lender can call the [redeem()](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L117) function, then the vault support/refinance a new lien from another vault, then the borrower repay his debt and then the [processEpoch()](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L275) function will be reverted by arithmetic under/overflow error.

The ```processEpoch()``` will be reverted because there is a problem in the calculation with the ```totalAssets()``` so the [s.yIntercept will be less than the totalAssets and the substract](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L332) will be reverted by underflow error.

The liquidity provider can not get his money because ```processEpoch()``` will not accumulate the ```withdrawReserve``` then ```transferWithdrawReserve()``` function will not transfer the liquidity provider funds to the ```WithdrawProxy```.

In the next test you can see that after the borrower repayment the ```totalAssets()``` will be inflated.

## Proof of Concept

I created a test in ```AstariaTest.t.sol```:

1. Create a vault and lend it 50 Ether.
2. Borrower get 10 ether for 10 days.
3. Create another public vault and lend it 11 ether.
4. The lender redeem 10 ether from the new public vault.
5. Borrower from the first vault refinance his terms and transfer his debt to the new public vault.
6. Borrower repay 10 ether of his debt to the new public vault.
7. Pass the time to the epoch end and call processEpoch(), the function will be reverted by arithmetic over/underflow.

As you can see in the logs, the ```totalAssets()``` increments more than ```yIntercept``` causing the underflow.

```solidity
function testProcessEpochAfterRedeemAndRefinance() public {
    // Lender funds may be trapped in the vault after lender redeem and the vault refinance a new lien.
    // 1. Create a vault and lend it 50 Ether.
    // 2. Borrower get 10 ether for 10 days.
    // 3. Create another public vault and lend it 11 ether.
    // 4. The lender redeem 10 ether.
    // 5. Borrower from the first vault refinance his terms and transfer his debt to the new public vault.
    // 6. Borrower repay 10 ether of his debt.
    // 7. Pass the time to the epoch end and call processEpoch(), the function will be reverted by arithmetic over/underflow.
    TestNFT nft = new TestNFT(2);
    address tokenContract = address(nft);
    uint256 tokenId = uint256(0);
    uint256 tokenIdTwo = uint256(1);

    uint256 initialBalance = WETH9.balanceOf(address(this));
    //
    // 1. Create a vault and lend it 50 Ether.
    //
    address publicVault = _createPublicVault({
        strategist: strategistOne,
        delegate: strategistTwo,
        epochLength: 15 days
    });
    // lend 50 ether to the PublicVault as address(1)
    _lendToVault(
        Lender({addr: address(1), amountToLend: 50 ether}),
        publicVault
    );
    console.log("");
    console.log("Public vault balance:               ", WETH9.balanceOf(publicVault));
    //
    // 2. Borrower get 10 ether for 10 days.
    //
    console.log("");
    console.log("address(this) ask for a 10 Ether borrow at 1.5% rate and 10 days duration");
    (uint256[] memory liens, ILienToken.Stack[] memory stack) = _commitToLien({
        vault: publicVault,
        strategist: strategistOne,
        strategistPK: strategistOnePK,
        tokenContract: tokenContract,
        tokenId: tokenId,
        lienDetails: standardLienDetails,
        amount: 10 ether,
        isFirstLien: true
    });
    console.log("");
    console.log("Public vault balance now:           ", WETH9.balanceOf(publicVault));
    vm.warp(block.timestamp + 3 days);
    console.log("");
    console.log("Jump 3 days after Public Vault creation...");
    console.log("");
    console.log("Borrower owed at 3 day:             ", LIEN_TOKEN.getOwed(stack[0]));
    console.log("Borrower lien interest:             ", LIEN_TOKEN.getInterest(stack[0]));
    console.log("Borrower lien duration:             ", stack[0].lien.details.duration);
    console.log("Borrower lien last:                 ", stack[0].point.last);
    //
    // 3. Create another public vault and lend it 11 ether.
    //
    address newPublicVault = _createPublicVault({
        strategist: strategistOne,
        delegate: strategistOne,
        epochLength: 15 days
    });
    _lendToVault(
        Lender({addr: address(7), amountToLend: 11 ether}),
        newPublicVault
    );
    console.log("");
    console.log("New Public vault creation and lend it 11 ether...");
    console.log("New Public vault balance:           ", WETH9.balanceOf(newPublicVault));
    console.log("New Public vault totalAssets:       ", PublicVault(newPublicVault).totalAssets());
    //
    // 4. The lender redeem 10 ether from the new public vault.
    //
    vm.startPrank(address(7));
    uint256 assets = PublicVault(newPublicVault).redeem(10 ether, address(7), address(7));
    console.log("");
    console.log("As a LP address(7) redeem 10 Ether shares from the new public vault...");
    console.log("Current epoch:                      ", PublicVault(newPublicVault).getCurrentEpoch());
    console.log("Assets for redeem in future epoch:  ", assets);
    WithdrawProxy withdrawProxy = PublicVault(newPublicVault).getWithdrawProxy(
        PublicVault(newPublicVault).getCurrentEpoch());
    console.log("WithdrawProxy shares:               ", IERC20(withdrawProxy).balanceOf(address(7)));
    console.log("WithdrawProxy eth balance:          ", WETH9.balanceOf(address(withdrawProxy)));
    console.log("Actual timeToEpochEnd():            ", PublicVault(newPublicVault).timeToEpochEnd());
    console.log("Actual withDrawReserve():           ", PublicVault(newPublicVault).getWithdrawReserve());
    console.log("New Public vault totalAssets:       ", PublicVault(newPublicVault).totalAssets());
    vm.stopPrank();
    //
    // 5. Borrower from the first vault refinance his terms and transfer his debt to the new public vault.
    //
    console.log("");
    console.log("Borrower refinance terms increase 3 days duaration. New duration is 13 days...");
    IAstariaRouter.Commitment memory refinanceTerms = _generateValidTerms({
        vault: newPublicVault,
        strategist: strategistOne,
        strategistPK: strategistOnePK,
        tokenContract: tokenContract,
        tokenId: tokenId,
        lienDetails: refinanceLienDetails3DaysDurationIncrease,
        amount: 10 ether,
        stack: stack
    });
    (, ILienToken.Stack memory newStack) = VaultImplementation(newPublicVault).buyoutLien(
        stack,
        uint8(0),
        refinanceTerms
    );
    console.log("");
    console.log("Borrower after refinance his debt");
    console.log("Borrower owed at 3 day:             ", LIEN_TOKEN.getOwed(newStack));
    console.log("Borrower lien interest:             ", LIEN_TOKEN.getInterest(newStack));
    console.log("Borrower lien duration:             ", newStack.lien.details.duration);
    console.log("Borrower lien last:                 ", newStack.point.last);

    console.log("");
    console.log("PublicVault balance after refinance:", WETH9.balanceOf(publicVault));
    console.log("New public vault balance after refinance:", WETH9.balanceOf(newPublicVault));

    console.log("");
    console.log("Current epoch:                      ", PublicVault(newPublicVault).getCurrentEpoch());
    WithdrawProxy withdrawProxyNewVault = PublicVault(newPublicVault).getWithdrawProxy(
        PublicVault(newPublicVault).getCurrentEpoch());
    console.log("withdrawProxyNewVault eth balance:  ", WETH9.balanceOf(address(withdrawProxyNewVault)));
    console.log("Actual timeToEpochEnd():            ", PublicVault(newPublicVault).timeToEpochEnd());
    console.log("Actual withDrawReserve():           ", PublicVault(newPublicVault).getWithdrawReserve());
    console.log("New Public vault totalAssets:       ", PublicVault(newPublicVault).totalAssets());
    //
    // 6. Borrower repay 10 ether of his debt to the new public vault.
    //
    console.log("");
    console.log("Address(this) repay 10 ether debt...");
    ILienToken.Stack[] memory newStackTuple = new ILienToken.Stack[](1);
    newStackTuple[0] = newStack;
    ILienToken.Stack[] memory newStackAfterRepay = _repay(
        newStackTuple, 0, 10 ether, address(this));
    console.log("New public vault balance after repay:", WETH9.balanceOf(newPublicVault));
    console.log("");
    console.log("Borrower after repaying his debt");
    console.log("Borrower owed at 3 day:             ", LIEN_TOKEN.getOwed(newStackAfterRepay[0]));
    console.log("Borrower lien interest:             ", LIEN_TOKEN.getInterest(newStackAfterRepay[0]));
    console.log("Borrower lien duration:             ", newStackAfterRepay[0].lien.details.duration);
    console.log("Borrower lien last:                 ", newStackAfterRepay[0].point.last);
    console.log("New Public vault totalAssets:       ", PublicVault(newPublicVault).totalAssets());
    //
    // 7. Pass the time to the epoch end and call processEpoch(), the function will be reverted
    //
    console.log("");
    console.log("Pass 15 days in order to process the Epoch...");
    vm.warp(block.timestamp + 15 days);
    console.log("");
    console.log("New Public vault yIntercept:        ", PublicVault(newPublicVault).getYIntercept());
    console.log("New Public vault totalAssets:       ", PublicVault(newPublicVault).totalAssets());
    console.log("");
    console.log("processEpoch() will be reverted by arithmetic over/underflow");
    vm.expectRevert();
    PublicVault(newPublicVault).processEpoch();
}
```

Output:

```bash
Logs:
  
  Public vault balance:                50000000000000000000
  
  address(this) ask for a 10 Ether borrow at 1.5% rate and 10 days duration
  Generating valid terms
  0x5e04a85fb1777f5deb6b8af74cebeb4739e76986d53836d25de402a5751a297342aaea5c801a1d6ec85ba43aa8fb3d047f0ecd06d228f32a3a6141ac461f11281b
  signature length: 65
  
  Public vault balance now:            40000000000000000000
  
  Jump 3 days after Public Vault creation...
  
  Borrower owed at 3 day:              10123287671231200000
  Borrower lien interest:              123287671231200000
  Borrower lien duration:              864000
  Borrower lien last:                  1668027215
  
  New Public vault creation and lend it 11 ether...
  New Public vault balance:            11000000000000000000
  New Public vault totalAssets:        11000000000000000000
  
  As a LP address(7) redeem 10 Ether shares from the new public vault...
  Current epoch:                       0
  Assets for redeem in future epoch:   10000000000000000000
  WithdrawProxy shares:                10000000000000000000
  WithdrawProxy eth balance:           0
  Actual timeToEpochEnd():             1296000
  Actual withDrawReserve():            0
  New Public vault totalAssets:        11000000000000000000
  
  Borrower refinance terms increase 3 days duaration. New duration is 13 days...
  0xdbbfd5784f191d17cc7884dbc848d926f2d13d07989f651199ba8d767c34a1df50d1264f36a698e50d405520e417d11ce5135c2998f239f458eff4a1d039d8261b
  signature length: 65
  
  Borrower after refinance his debt
  Borrower owed at 3 day:              10123287671231200000
  Borrower lien interest:              0
  Borrower lien duration:              1123200
  Borrower lien last:                  1668286415
  
  PublicVault balance after refinance: 50152054794518480000
  New public vault balance after refinance: 847945205481520000
  
  Current epoch:                       0
  withdrawProxyNewVault eth balance:   0
  Actual timeToEpochEnd():             1296000
  Actual withDrawReserve():            0
  New Public vault totalAssets:        11000000000000000000
  
  Address(this) repay 10 ether debt...
  New public vault balance after repay: 10847945205481520000
  
  Borrower after repaying his debt
  Borrower owed at 3 day:              123287671231200000
  Borrower lien interest:              0
  Borrower lien duration:              1123200
  Borrower lien last:                  1668286415
  New Public vault totalAssets:        11000000000000000000
  
  Pass 15 days in order to process the Epoch...
  
  New Public vault yIntercept:         11000000000000000000
  New Public vault totalAssets:        375175131460854176000
  
  processEpoch() will be reverted by arithmetic over/underflow
```

## Tools used

Foundry/Vscode

## Recommended Mitigation Steps

Review the [totalAssets()](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L490) calculation. After the borrower repayment the totalAssets() is inflated.