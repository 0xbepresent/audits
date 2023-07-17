# Original link
https://github.com/code-423n4/2022-08-foundation-findings/issues/55
1 - Unnecessary checked arithmetic in for loop.
--

There is no risk that the loop counter can overflow, using solidity's unchecked block saves gas.

There are 5 instances of this issue:

```
./mixins/shared/MarketFees.sol:126: for (uint256 i = 0; i < creatorRecipients.length; ++i) {
./mixins/shared/MarketFees.sol:198: for (uint256 i = 0; i < creatorShares.length; ++i) {
./mixins/shared/MarketFees.sol:484: for (uint256 i = 0; i < creatorRecipients.length; ++i) {
./libraries/BytesLibrary.sol:25:      for (uint256 i = 0; i < 20; ++i) {
./libraries/BytesLibrary.sol:44:      for (uint256 i = 0; i < 4; ++i) {
```

Gas Report using [this gist](https://gist.github.com/0xbepresent/707eefd3ead1b0a297b0f17d3dc54c7f):

```
╭─────────────────────────────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ src/test/Unchecked.t.sol:Contract0 contract ┆                 ┆      ┆        ┆      ┆         │
╞═════════════════════════════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost                             ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 55105                                       ┆ 307             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                               ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withOutUnChecked                            ┆ 2068            ┆ 2068 ┆ 2068   ┆ 2068 ┆ 1       │
╰─────────────────────────────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
╭─────────────────────────────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ src/test/Unchecked.t.sol:Contract1 contract ┆                 ┆      ┆        ┆      ┆         │
╞═════════════════════════════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost                             ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 53705                                       ┆ 300             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                               ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withUnchecked                               ┆ 1408            ┆ 1408 ┆ 1408   ┆ 1408 ┆ 1       │
╰─────────────────────────────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
```

2 - Cache array length outside of loop
--

Caching the array length outside a loop saves reading it on each iteration, as long as the array's length is not changed during the loop.

There are 4 instances of this issue:

```
./mixins/shared/MarketFees.sol:126: for (uint256 i = 0; i < creatorRecipients.length; ++i) {
./mixins/shared/MarketFees.sol:198: for (uint256 i = 0; i < creatorShares.length; ++i) {
./mixins/shared/MarketFees.sol:484: for (uint256 i = 0; i < creatorRecipients.length; ++i) {
./mixins/shared/MarketFees.sol:503: for (uint256 i = 1; i < creatorRecipients.length; ) {
```

Gas Report using [this gist](https://gist.github.com/0xbepresent/3b009070d671349351a4301a494d3d8f):

```
╭───────────────────────────────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ src/test/ArrayLength.t.sol:Contract0 contract ┆                 ┆      ┆        ┆      ┆         │
╞═══════════════════════════════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost                               ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 137640                                        ┆ 412             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                                 ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withArrayLengthInside                         ┆ 3108            ┆ 3108 ┆ 3108   ┆ 3108 ┆ 1       │
╰───────────────────────────────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
╭───────────────────────────────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ src/test/ArrayLength.t.sol:Contract1 contract ┆                 ┆      ┆        ┆      ┆         │
╞═══════════════════════════════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost                               ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 137840                                        ┆ 413             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name                                 ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ withArrayLengthOutside                        ┆ 2813            ┆ 2813 ┆ 2813   ┆ 2813 ┆ 1       │
╰───────────────────────────────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
```



3 - Use custom errors rather than revert()/require() strings to save gas
--

[Custom errors](https://blog.soliditylang.org/2021/04/21/custom-errors/) are available since solidity version 0.8.4. Custom errors save ~50 gas avoiding having to allocate and store the revert string. Custom errors are been used in the [NFTDropMarketFixedPriceSale contract](https://github.com/code-423n4/2022-08-foundation/blob/main/contracts/mixins/nftDropMarket/NFTDropMarketFixedPriceSale.sol#L16)

There are 13 instances of this issue:

```
contracts/NFTCollectionFactory.sol::262 => require(bytes(symbol).length != 0, "NFTCollectionFactory: Symbol is required");
contracts/NFTCollection.sol::158 => require(tokenCreatorPaymentAddress != address(0), "NFTCollection: tokenCreatorPaymentAddress is required");
contracts/NFTCollection.sol::263 => require(bytes(tokenCID).length != 0, "NFTCollection: tokenCID is required");
contracts/NFTCollection.sol::264 => require(!cidToMinted[tokenCID], "NFTCollection: NFT was already minted");
contracts/NFTCollection.sol::268 => require(maxTokenId == 0 || tokenId <= maxTokenId, "NFTCollection: Max token count has already been minted");
contracts/NFTCollection.sol::327 => require(_exists(tokenId), "NFTCollection: URI query for nonexistent token");
contracts/NFTDropCollection.sol::88 => require(bytes(_baseURI).length > 0, "NFTDropCollection: `_baseURI` must be set");
contracts/NFTDropCollection.sol::93 => require(postRevealBaseURIHash != bytes32(0), "NFTDropCollection: Already revealed");
contracts/NFTDropCollection.sol::130 => require(bytes(_symbol).length > 0, "NFTDropCollection: `_symbol` must be set");
contracts/NFTDropCollection.sol::131 => require(_maxTokenId > 0, "NFTDropCollection: `_maxTokenId` must be set");
contracts/NFTDropCollection.sol::172 => require(count != 0, "NFTDropCollection: `count` must be greater than 0");
contracts/NFTDropCollection.sol::179 => require(latestTokenId <= maxTokenId, "NFTDropCollection: Exceeds max tokenId");
contracts/NFTDropCollection.sol::238 => require(_postRevealBaseURIHash != bytes32(0), "NFTDropCollection: use `reveal` instead");
```