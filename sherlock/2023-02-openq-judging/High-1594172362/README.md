# Original link
https://github.com/sherlock-audit/2023-02-openq-judging/issues/296
0xbepresent

high

# User claim is compromised if the deposited NFT is refunded by the funder.

## Summary

If one NFT is refunded before the user claim his bounty the [ClaimManagerV1.sol::_claimAtomicBounty()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L123), [ClaimManagerV1.sol::_claimTieredPercentageBounty()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L203) and [ClaimManagerV1.sol::_claimTieredFixedBounty()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L278) functions will be reverted because the NFT Deposits array is not decreased in the refund action.

## Vulnerability Detail

The funder can fund the bounty with a NFT via [DepositManagerV1.sol::fundBountyNFT()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L113) function. Then when the funder wants to refund his NFT he can do it with the [DepositManagerV1.sol::refundDeposit()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L152) function. The problem is that [BountyCore.sol::refundDeposit()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L64) function does not decrease the [nftDeposits](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/AtomicBountyV1.sol#L149) array.

The ```nftDeposits``` array is important because if there is an inconsistency in the [FOR statement](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L320) in the claim functions the claim functions will be reverted.

## Impact

The user can not claim his bounties that are still available in the contract.

I created a test in ```ClaimManager.test.js```. Basically the funder put two NFT to the same tier winner, then the funder refunds one of his NFT and the winner can not claim one of the NFT that is still available.

Test steps:

1. Create Tiered percentage bounty
2. Fund the bounty with 2 NFTs for the first place
3. Funder refund one NFT assigned the first place winner
4. First place claim his bounty but it will be reverted by "ERC721: caller is not token owner nor approved"
5. All other bounties are nor claimable by the winner

```javascript
it('NFT transfer reverts when the NFT is refunded', async () => {
    //User claim could be compromised if NFT funder refunds his NFT
    // 1. Create Tiered percentage bounty
    // 2. Fund the bounty with 2 NFTs for the first place
    // 3. Funder refund one NFT assigned the first place winner
    // 4. First place claim his bounty but it will be reverted by "ERC721: caller is not token owner nor approved"
    const FIRST_PLACE_NFT = 1;
    const FIRST_PLACE_NFT_SECOND_NFT = 3;
    const SECOND_PLACE_NFT = 2;
    expect(await mockNft.ownerOf(FIRST_PLACE_NFT)).to.equal(owner.address);
    expect(await mockNft.ownerOf(FIRST_PLACE_NFT_SECOND_NFT)).to.equal(owner.address);
    expect(await mockNft.ownerOf(SECOND_PLACE_NFT)).to.equal(owner.address);
    //
    // 1. Create Tiered percentage bounty
    //
    await openQProxy.mintBounty(
        Constants.bountyId,
        Constants.organization,
        tieredPercentageBountyInitOperation_permissionless);
    const bountyAddress = await openQProxy.bountyIdToAddress(Constants.bountyId);
    //
    // 2. Fund the bounty with 2 NFTs for the first place
    //
    await mockNft.approve(bountyAddress, FIRST_PLACE_NFT);
    await mockNft.approve(bountyAddress, FIRST_PLACE_NFT_SECOND_NFT);
    await mockNft.approve(bountyAddress, SECOND_PLACE_NFT);
    await depositManager.fundBountyNFT(
        bountyAddress, mockNft.address, FIRST_PLACE_NFT, 1, zeroTier);
    await depositManager.fundBountyNFT(
        bountyAddress, mockNft.address, FIRST_PLACE_NFT_SECOND_NFT, 1, zeroTier);
    await depositManager.fundBountyNFT(
        bountyAddress, mockNft.address, SECOND_PLACE_NFT, 1, oneTier);
    expect(await mockNft.ownerOf(FIRST_PLACE_NFT)).to.equal(bountyAddress);
    expect(await mockNft.ownerOf(FIRST_PLACE_NFT_SECOND_NFT)).to.equal(bountyAddress);
    expect(await mockNft.ownerOf(SECOND_PLACE_NFT)).to.equal(bountyAddress);
    //
    // 3. Funder refund one NFT from the first place
    //
    const bountyDepositId = generateDepositId(Constants.bountyId, 0);
    await expect(
        depositManager.refundDeposit(bountyAddress, bountyDepositId));
    expect(await mockNft.ownerOf(FIRST_PLACE_NFT)).to.equal(owner.address);//refund NFT to owner
    //
    // 4. First place claim his bounty but it will be reverted by
    // "ERC721: caller is not token owner nor approved"
    //
    let abiEncodedTieredCloserDataFirstPlace = abiCoder.encode(
        ['address', 'string', 'address', 'string', 'uint256'],
        [owner.address, "FlacoJones", owner.address, "https://github.com/OpenQDev/OpenQ-Frontend/pull/398", 0]);
    await expect(claimManager.connect(oracle).claimBounty(
        bountyAddress,
        claimant.address,
        abiEncodedTieredCloserDataFirstPlace)).to.be.revertedWith("ERC721: caller is not token owner nor approved");
});
```


## Code Snippet

The ```ClaimManagerV1.sol::_claimAtomicBounty()``` FOR statement:

```solidity
File: ClaimManagerV1.sol
150:         for (uint256 i = 0; i < _bounty.getNftDeposits().length; i++) {
151:             _bounty.claimNft(_closer, _bounty.nftDeposits(i));
152: 
153:             emit NFTClaimed(
154:                 _bounty.bountyId(),
155:                 address(_bounty),
156:                 _bounty.organization(),
157:                 _closer,
158:                 block.timestamp,
159:                 _bounty.tokenAddress(_bounty.nftDeposits(i)),
160:                 _bounty.tokenId(_bounty.nftDeposits(i)),
161:                 _bounty.bountyType(),
162:                 _closerData,
163:                 VERSION_1
164:             );
165:         }
```

The ```ClaimManagerV1.sol::_claimTieredPercentageBounty()``` FOR statement:

```solidity
File: ClaimManagerV1.sol
251:         for (uint256 i = 0; i < _bounty.getNftDeposits().length; i++) {
252:             bytes32 _depositId = _bounty.nftDeposits(i);
253:             if (_bounty.tier(_depositId) == _tier) {
254:                 _bounty.claimNft(_closer, _depositId);
255: 
256:                 emit NFTClaimed(
257:                     _bounty.bountyId(),
258:                     address(_bounty),
259:                     _bounty.organization(),
260:                     _closer,
261:                     block.timestamp,
262:                     _bounty.tokenAddress(_depositId),
263:                     _bounty.tokenId(_depositId),
264:                     _bounty.bountyType(),
265:                     _closerData,
266:                     VERSION_1
267:                 );
268:             }
269:         }
```

The ```ClaimManagerV1.sol::_claimTieredFixedBounty()``` FOR statement:

```solidity
File: ClaimManagerV1.sol
320:         for (uint256 i = 0; i < _bounty.getNftDeposits().length; i++) {
321:             bytes32 _depositId = _bounty.nftDeposits(i);
322:             if (_bounty.tier(_depositId) == _tier) {
323:                 _bounty.claimNft(_closer, _depositId);
324: 
325:                 emit NFTClaimed(
326:                     _bounty.bountyId(),
327:                     address(_bounty),
328:                     _bounty.organization(),
329:                     _closer,
330:                     block.timestamp,
331:                     _bounty.tokenAddress(_depositId),
332:                     _bounty.tokenId(_depositId),
333:                     _bounty.bountyType(),
334:                     _closerData,
335:                     VERSION_1
336:                 );
337:             }
338:         }
```

The ```BountyCore.sol::refundDeposit()``` where the ```nftDeposits``` array is not decreased.

```solidity
File: BountyCore.sol
64:     function refundDeposit(
65:         bytes32 _depositId,
66:         address _funder,
67:         uint256 _volume
68:     ) external virtual onlyDepositManager nonReentrant {
69:         require(!refunded[_depositId], Errors.DEPOSIT_ALREADY_REFUNDED);
70:         require(funder[_depositId] == _funder, Errors.CALLER_NOT_FUNDER);
71:         require(
72:             block.timestamp >= depositTime[_depositId] + expiration[_depositId],
73:             Errors.PREMATURE_REFUND_REQUEST
74:         );
75: 
76:         refunded[_depositId] = true;
77: 
78:         if (tokenAddress[_depositId] == address(0)) {
79:             _transferProtocolToken(funder[_depositId], _volume);
80:         } else if (isNFT[_depositId]) {
81:             _transferNft(
82:                 tokenAddress[_depositId],
83:                 funder[_depositId],
84:                 tokenId[_depositId]
85:             );
86:         } else {
87:             _transferERC20(
88:                 tokenAddress[_depositId],
89:                 funder[_depositId],
90:                 _volume
91:             );
92:         }
93:     }
```

## Tool used

Vscode

## Recommendation

Decrease the ```nftDeposits``` array when there is a refund action.


Duplicate of #263 
