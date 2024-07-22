# Original link
https://github.com/code-423n4/2023-01-astaria-findings/issues/222
# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L313-L351


# Vulnerability details

## Impact
Vault A can call buyoutLien to buy out Vault B's lien tokens, which calls LienToken.buyoutLien
```solidity
  function buyoutLien(
    ILienToken.Stack[] calldata stack,
    uint8 position,
    IAstariaRouter.Commitment calldata incomingTerms
  )
    external
    whenNotPaused
    returns (ILienToken.Stack[] memory, ILienToken.Stack memory)
  {
...
    return
      lienToken.buyoutLien(
        ILienToken.LienActionBuyout({
          position: position,
          encumber: ILienToken.LienActionEncumber({
            amount: owed,
            receiver: recipient(),
            lien: ROUTER().validateCommitment({
              commitment: incomingTerms,
              timeToSecondEpochEnd: _timeToSecondEndIfPublic()
            }),
            stack: stack
```
 In LienToken.buyoutLien, it will burn Vault B's lien token and mint a new lien token for Vault A
```solidity
  function _replaceStackAtPositionWithNewLien(
    LienStorage storage s,
    ILienToken.Stack[] calldata stack,
    uint256 position,
    Stack memory newLien,
    uint256 oldLienId
  ) internal returns (ILienToken.Stack[] memory newStack) {
    newStack = stack;
    newStack[position] = newLien;
    _burn(oldLienId);                        // @ audit: burn Vault B's lien token
    delete s.lienMeta[oldLienId];
  }
...
    newLienId = uint256(keccak256(abi.encode(params.lien)));
    Point memory point = Point({
      lienId: newLienId,
      amount: params.amount.safeCastTo88(),
      last: block.timestamp.safeCastTo40(),
      end: (block.timestamp + params.lien.details.duration).safeCastTo40()
    });
    _mint(params.receiver, newLienId); // @ audit: mint a new lien token for Vault A
    return (newLienId, Stack({lien: params.lien, point: point}));
  }
```

And, when Vault B is a public vault, the handleBuyoutLien function of Vault B will be called to decrease liensOpenForEpoch
However, when Vault A is a public vault, it does not increase the liensOpenForEpoch of Vault A
```solidity
    if (_isPublicVault(s, payee)) {
      IPublicVault(payee).handleBuyoutLien(
        IPublicVault.BuyoutLienParams({
          lienSlope: calculateSlope(params.encumber.stack[params.position]),
          lienEnd: params.encumber.stack[params.position].point.end,
          increaseYIntercept: buyout -
            params.encumber.stack[params.position].point.amount
        })
      );
    }
...
  function handleBuyoutLien(BuyoutLienParams calldata params)
    public
    onlyLienToken
  {
    VaultData storage s = _loadStorageSlot();

    unchecked {
      uint48 newSlope = s.slope - params.lienSlope.safeCastTo48();
      _setSlope(s, newSlope);
      s.yIntercept += params.increaseYIntercept.safeCastTo88();
      s.last = block.timestamp.safeCastTo40();
    }

    _decreaseEpochLienCount(s, getLienEpoch(params.lienEnd.safeCastTo64()));  // @audit: decrease liensOpenForEpoch 
    emit YInterceptChanged(s.yIntercept);
  }
```
Since the liensOpenForEpoch of the public vault decreases when the lien token is repaid, and since the liensOpenForEpoch of public vault A is not increased, then when that lien token is repaid, _payment will fail due to overflow when decreasing the liensOpenForEpoch.
```solidity
    } else {
      amount = stack.point.amount;
      if (isPublicVault) {
        // since the openLiens count is only positive when there are liens that haven't been paid off
        // that should be liquidated, this lien should not be counted anymore
        IPublicVault(lienOwner).decreaseEpochLienCount(  //  @audit: overflow here
          IPublicVault(lienOwner).getLienEpoch(end)
        );
      }
```
Consider the following case.
Public Vault B holds a lien token and B.liensOpenForEpoch == 1
Public Vault A buys out B's lien token for refinancing, B.liensOpenForEpoch == 0, A.liensOpenForEpoch == 0
borrower wants to repay the loan, in the _payment function, the decreaseEpochLienCount function of Vault A will be called, `A.liensOpenForEpoch-- `will trigger an overflow, resulting in borrower not being able to repay the loan, and borrower's collateral will be auctioned off, but in the call to updateVaultAfterLiquidation function will also fail in decreaseEpochLienCount due to the overflow
```solidity
  function updateVaultAfterLiquidation(
    uint256 maxAuctionWindow,
    AfterLiquidationParams calldata params
  ) public onlyLienToken returns (address withdrawProxyIfNearBoundary) {
    VaultData storage s = _loadStorageSlot();

    _accrue(s);
    unchecked {
      _setSlope(s, s.slope - params.lienSlope.safeCastTo48());
    }

    if (s.currentEpoch != 0) {
      transferWithdrawReserve();
    }
    uint64 lienEpoch = getLienEpoch(params.lienEnd);
    _decreaseEpochLienCount(s, lienEpoch); //  @audit: overflow here
```
As a result, the borrower cannot repay the loan and the borrower's collateral cannot be auctioned off, thus causing the depositor of the public vault to suffer a loss
## Proof of Concept
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L313-L351
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L835-L843
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L640-L655
## Tools Used
None
## Recommended Mitigation Steps
In LienToken.buyoutLien, when the caller is a public vault, increase the decreaseEpochLienCount of the public vault
