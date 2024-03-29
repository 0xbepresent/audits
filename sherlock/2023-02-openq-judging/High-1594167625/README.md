# Original link
https://github.com/sherlock-audit/2023-02-openq-judging/issues/220
0xbepresent

medium

# The ```DepositManagerV1.sol::fundBountyToken()``` must accept only whitelisted tokens.

## Summary

The [DepositManagerV1.sol::fundBountyToken()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L36) function accepts non whitelisted tokens if the bounty has not reached the token address limit which is dangerous because the bounty could be feeded with malicious ERC20 tokens.

## Vulnerability Detail

The ```DepositManagerV1.sol::fundBountyToken()``` accepts non whitelisted tokens if the Bounty has not reached the token address limit as you can see:

```solidity
File: DepositManagerV1.sol
45:         if (!isWhitelisted(_tokenAddress)) {
46:             require(
47:                 !tokenAddressLimitReached(_bountyAddress),
48:                 Errors.TOO_MANY_TOKEN_ADDRESSES
49:             );
50:         }
```

That behaivor could be potentially dangerous if there are deposits with a malicious tokens like:
- Rebase tokens
- Fee on transfer tokens

The [DepositManagerV1.sol::fundBountyNFT()](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#LL113-L123C79) checks if the NFT address is whitelisted beforehand, so the ```fundBountyToken()``` should have the same logic.

## Impact

Deposits with malicious tokens can be dangerous so the protocol should accept only whitelisted tokens.

I created a test in DepositManager.test.js where it is possible to deposit a non whitelisted token.

1. Create the atomic bounty.
2. Approve the blackListed token.
3. Set the token address limit in order to accept several tokens.
4. Fund the bounty and the blackListed token will be accepted.

```solidity
File: DepositManager.test.js
282: 		it('should revert if funded with a non-whitelisted token.', async () => {
283: 			// The bounty must accept only whitelisted tokens.
284: 			// 1. Create the atomic bounty.
285: 			// 2. Approve the blackListed token.
286: 			// 3. Set the token address limit in order to accept several tokens.
287: 			// 4. Fund the bounty and the blackListed token will be accepted.
288: 			//
289: 			// 1. Create the atomic bounty.
290: 			//
291: 			await openQProxy.mintBounty(Constants.bountyId, Constants.organization, atomicBountyInitOperation);
292: 			const bountyAddress = await openQProxy.bountyIdToAddress(Constants.bountyId);
293: 			//
294: 			// 2. Approve the blackListed token.
295: 			//
296: 			await blacklistedMockDai.approve(bountyAddress, 10000000);
297: 			//
298: 			// 3. Set the token address limit in order to accept several tokens.
299: 			//
300: 			await openQTokenWhitelist.setTokenAddressLimit(10);
301: 			//
302: 			// 4. Fund the bounty and the blackListed token will be accepted.
303: 			// 
304: 			await expect(depositManager.fundBountyToken(
305: 				bountyAddress,
306: 				blacklistedMockDai.address,
307: 				10000000,
308: 				1,
309: 				Constants.funderUuid)).to.not.be.revertedWith('TOO_MANY_TOKEN_ADDRESSES');
310: 		});
```

## Code Snippet

The ```fundBountyToken()``` accepts the non whitelisted token if the bounty has not reached the limit.

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

The ```fundBountyNFT()``` checks if the token address is whitelisted.

```solidity
File: DepositManagerV1.sol
113:     function fundBountyNFT(
114:         address _bountyAddress,
115:         address _tokenAddress,
116:         uint256 _tokenId,
117:         uint256 _expiration,
118:         bytes calldata _data
119:     ) external onlyProxy {
120:         IBounty bounty = IBounty(payable(_bountyAddress));
121: 
122:         require(isWhitelisted(_tokenAddress), Errors.TOKEN_NOT_ACCEPTED);
123:         require(bountyIsOpen(_bountyAddress), Errors.CONTRACT_ALREADY_CLOSED);
```

## Tool used

Vscode

## Recommendation

Follow the same logic used in the ```fundBountyNFT()```, checks if the token is whitelisted regardless of the token address limit.

Duplicate of #62
