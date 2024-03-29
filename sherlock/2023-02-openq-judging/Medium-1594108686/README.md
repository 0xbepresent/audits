# Original link
https://github.com/sherlock-audit/2023-02-openq-judging/issues/153
0xbepresent

high

# ```tokenAddresses``` count is not decreased on refunds causing a limitation in deposits.

## Summary

The ```tokenAddresses``` count is not decreased on the refund action causing a bad validation in the ```fundBountyToken() ``` function because the function checks the token addresses count with the ```tokenAddressLimitReached()``` function.

## Vulnerability Detail

When there is a deposit to the Bounty a [tokenAddress count is increased](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L55). If the funder [refunds his deposit](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L64), the ```tokenAddresses``` count is not decreased, so if another funder wants to deposit the [tokenAddressLimitReached()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L47) function will get bad information because one token address was refunded previously by the funder but the tokenAddress count was not decreased.

## Impact

Various funders can reach the [token address limit](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L216) then the funders withdraw his money and [tokenAddressLimitReached()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L207) function will still return ```true``` because the token addresses count were not decreased when the funders withdrew his money. Causing others funders can not deposit.

I created a test in DepositMAnager.test.js where it can seen that the funder deposit a tokenAddress, then the funder withdraw his money, then the funder wants to deposit again but the function will be reverted becuase "TOO_MANY_TOKEN_ADDRESSES" error.

1. Create the atomic bounty
2. Fund the bounty. The tokenAddress is set with the mockLink address.
3. Refund the mockLink token and the volume is refunded.
4. Fund the bounty again with a blacklisted token, it will be reverted because "TOO_MANY_TOKEN_ADDRESSES"

```javascript
it('TokenAddresses count is not decreased after refund', async () => {
    // The Bounty could be limited by deposits if the token address is not decreased
    // 1. Create the atomic bounty
    // 2. Fund the bounty. The tokenAddress is set with the mockLink address.
    // 3. Refund the mockLink token and the volume is refunded.
    // 4. Fund the bounty again with a blacklisted token, it will be reverted because "TOO_MANY_TOKEN_ADDRESSES"
    //
    // 1. Create the atomic bounty.
    //
    await openQProxy.mintBounty(
        Constants.bountyId, Constants.organization, atomicBountyInitOperation);
    const bountyAddress = await openQProxy.bountyIdToAddress(Constants.bountyId);
    const volume = 100;
    await mockLink.approve(bountyAddress, 10000000);
    await AtomicBountyV1.attach(bountyAddress);
    await openQTokenWhitelist.setTokenAddressLimit(1);
    //
    // 2. Fund the bounty. The tokenAddress is set with the mockLink address
    //
    await depositManager.fundBountyToken(
        bountyAddress,
        mockLink.address,
        volume,
        1,
        Constants.funderUuid);
    expect((await mockLink.balanceOf(bountyAddress)).toString()).to.equal('100');
    //
    // 3. Refund the mockLink and the volume is refunded.
    //
    const bountyDepositId = generateDepositId(Constants.bountyId, 0);
    const expectedTimestamp = await setNextBlockTimestamp(2764800);
    await expect(
        depositManager.refundDeposit(bountyAddress, bountyDepositId))
        .to.emit(depositManager, 'DepositRefunded')
        .withArgs(
            bountyDepositId,
            Constants.bountyId,
            bountyAddress,
            Constants.organization,
            expectedTimestamp,
            mockLink.address,
            volume,
            0,
            [],
            Constants.VERSION_1);
    expect((await mockLink.balanceOf(bountyAddress)).toString()).to.equal('0');
    //
    // 4. Fund the bounty again with a blacklisted token, it will be reverted because "TOO_MANY_TOKEN_ADDRESSES"
    //
    await blacklistedMockDai.approve(bountyAddress, 10000000);
    await expect(depositManager.fundBountyToken(
        bountyAddress,
        blacklistedMockDai.address,
        10000000,
        1,
        Constants.funderUuid)).to.be.revertedWith('TOO_MANY_TOKEN_ADDRESSES');
});
```

## Code Snippet

In the [receiveFunds()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L21) function the ```tokenAddresses``` is increased.

```solidity
File: BountyCore.sol
53: 
54:         deposits.push(depositId);
55:         tokenAddresses.add(_tokenAddress);
56: 
57:         return (depositId, volumeReceived);
```

In the [refundDeposit()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L64) the ```tokenAddresses``` is not decreased.

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

The [tokenAddressLimitReached()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L207) function will return a wrong value.

```solidity
File: DepositManagerV1.sol
207:     function tokenAddressLimitReached(address _bountyAddress)
208:         public
209:         view
210:         returns (bool)
211:     {
212:         IBounty bounty = IBounty(payable(_bountyAddress));
213: 
214:         return
215:             bounty.getTokenAddressesCount() >=
216:             openQTokenWhitelist.TOKEN_ADDRESS_LIMIT();
217:     }
```

In the [fundBountyToken()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L36) the ```tokenAddressLimitReached()``` function will limit the deposits.

```solidity
File: DepositManagerV1.sol
36:     function fundBountyToken(
37:         address _bountyAddress,
38:         address _tokenAddress,
39:         uint256 _volume,
40:         uint256 _expiration,
41:         string memory funderUuid
42:     ) external payable onlyProxy {
43:         IBounty bounty = IBounty(payable(_bountyAddress));
44: 
45:         if (!isWhitelisted(_tokenAddress)) {
46:             require(
47:                 !tokenAddressLimitReached(_bountyAddress),
48:                 Errors.TOO_MANY_TOKEN_ADDRESSES
49:             );
50:         }
```



## Tool used

Vscode

## Recommendation

Decrease the ```tokenAddresses``` count when the funder refunds his deposit.

Duplicate of #530
