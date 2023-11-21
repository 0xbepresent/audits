# Original link
https://github.com/code-423n4/2023-01-astaria-findings/issues/303
# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L596
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L608
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L840


# Vulnerability details

## Impact

The borrower cannot repay his debt after transferring to another vault and the borrower can lost his NFT. When the borrower transfer his debt to another vault the LienCount is not added, so when the borrower wants to pay all his debt the [decreaseEpochLienCount](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L840) will be reverted.

Borrower can not get his NFT even when the borrower wants to pay his debt.

## Proof of Concept

I created a test for this situation in ```AstariaTest.t.sol```, following the next steps:

1. Create a public vault and lent it 50 Ether.
2. Borrower get 10 Ether from the vault.
3. After 3 days another public vault is created.
4. Borrower refinance the terms and transfer the debt to the New Public Vault.
5. Borrower wants to repay all his debt but is not possible by Arithmetic over/under flow error.

```solidity
function testRefinanceDontIncreaseLiensOpenForEpoch() public {
    // Borrower can not repay his debt after refinance it.
    // 1. Create a public vault and lent it 50 Ether.
    // 2. Borrower get 10 Ether from the vault.
    // 3. After 3 days another public vault is created.
    // 4. Borrower refinance the terms and transfer the debt to the New Public Vault.
    // 5. Borrower wants to repay all his debt but is not possible by Arithmetic over/under flow error.
    TestNFT nft = new TestNFT(2);
    address tokenContract = address(nft);
    uint256 tokenId = uint256(0);
    uint256 tokenIdTwo = uint256(1);

    uint256 initialBalance = WETH9.balanceOf(address(this));
    //
    // 1. Create a public vault and lent it 50 Ether.
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
    // 2. Borrower get 10 Ether from the vault.
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

    //
    // 3. After 3 days another public vault is created.
    //
    vm.warp(block.timestamp + 3 days);
    console.log("");
    console.log("Jump 3 days after Public Vault creation...");

    console.log("");
    console.log("Borrower owed at 3 day:             ", LIEN_TOKEN.getOwed(stack[0]));
    console.log("Borrower lien amount:               ", stack[0].point.amount);
    console.log("Borrower lien last:                 ", stack[0].point.last);
    console.log("Borrower lien end:                  ", stack[0].point.end);
    console.log("Borrower lien interest:             ", LIEN_TOKEN.getInterest(stack[0]));
    console.log("Borrower lien rate:                 ", stack[0].lien.details.rate);
    console.log("Borrower lien duration:             ", stack[0].lien.details.duration);
    console.log("Borrower lien maxPotentialDebt:     ", stack[0].lien.details.maxPotentialDebt);
    console.log("Borrower lien maxAmount:            ", stack[0].lien.details.maxAmount);
    console.log("Borrower lien liquidationInitialAsk:", stack[0].lien.details.liquidationInitialAsk);
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
    console.log("New Public vault creation and lend it 11 Ether...");
    console.log("New Public vault balance:           ", WETH9.balanceOf(newPublicVault));
    //
    // 4. Borrower refinance the terms and pass the debt to the New Public Vault.
    //
    console.log("");
    console.log("Borrower refinance terms incrase 3 days duration. New duration is 13 days...");
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
    console.log("Borrower after refinance his debt...");
    console.log("Borrower owed at 3 day:             ", LIEN_TOKEN.getOwed(newStack));
    console.log("Borrower lien amount:               ", newStack.point.amount);
    console.log("Borrower lien last:                 ", newStack.point.last);
    console.log("Borrower lien end:                  ", newStack.point.end);
    console.log("Borrower lien interest:             ", LIEN_TOKEN.getInterest(newStack));//((1.5% anual /100) / 31536000 seconds in one year) * 60 * 60 * 24 un dia * 9 ether
    console.log("Borrower lien rate:                 ", newStack.lien.details.rate);
    console.log("Borrower lien duration:             ", newStack.lien.details.duration);
    console.log("Borrower lien maxPotentialDebt:     ", newStack.lien.details.maxPotentialDebt);
    console.log("Borrower lien maxAmount:            ", newStack.lien.details.maxAmount);
    console.log("Borrower lien liquidationInitialAsk:", newStack.lien.details.liquidationInitialAsk);

    console.log("");
    console.log("PublicVault balance after refinance:", WETH9.balanceOf(publicVault));
    console.log("New public vault balance after refinance:", WETH9.balanceOf(newPublicVault));
    //
    // 5. Borrower wants to repay all his debt but is not possible by Arithmetic over/under flow error.
    //
    // repay debt
    console.log("");
    console.log("Address(this) repay all his debt in the new vault...");
    ILienToken.Stack[] memory newStackTuple = new ILienToken.Stack[](1);
    newStackTuple[0] = newStack;
    uint256 amountRepay = LIEN_TOKEN.getOwed(newStack);
    vm.deal(address(this), amountRepay * 3);
    WETH9.deposit{value: amountRepay * 2}();
    WETH9.approve(address(TRANSFER_PROXY), amountRepay * 2);
    WETH9.approve(address(LIEN_TOKEN), amountRepay * 2);
    vm.expectRevert(); //<--Arithmetic over/underflow
    LIEN_TOKEN.makePayment(
        newStackTuple[0].lien.collateralId,
        newStackTuple,
        0,
        amountRepay
    );
}
```


## Tools used

Foundry/Vscode

## Recommended Mitigation Steps

Increase the Lien count after transferring to another vault.