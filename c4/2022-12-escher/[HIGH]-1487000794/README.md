# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/392
# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L117-L124


# Vulnerability details

The dutch auction in the `LPDA` contract is implemented by configuring a start price and price drop per second.

A bad set of settings can cause an issue where the elapsed duration of the sale multiplied by the drop per second gets bigger than the start price and underflows the current price calculation.

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L117

```solidity
function getPrice() public view returns (uint256) {
    Sale memory temp = sale;
    (uint256 start, uint256 end) = (temp.startTime, temp.endTime);
    if (block.timestamp < start) return type(uint256).max;
    if (temp.currentId == temp.finalId) return temp.finalPrice;

    uint256 timeElapsed = end > block.timestamp ? block.timestamp - start : end - start;
    return temp.startPrice - (temp.dropPerSecond * timeElapsed);
}
```

This means that if `temp.dropPerSecond * timeElapsed > temp.startPrice` then the unsigned integer result will become negative and underflow, leading to potentially bricking the contract and an eventual loss of funds.

## Impact

Due to Solidity 0.8 default checked math, the subtraction of the start price and the drop will cause a negative value that will generate an underflow in the unsigned integer type and lead to a transaction revert.

Calls to `getPrice` will revert, and since this function is used in the `buy` to calculate the current NFT price it will also cause the buy process to fail. The price drop will continue to increase as time passes, making it impossible to recover from this situation and effectively bricking the contract.

This will eventually lead to a loss of funds because currently the only way to end a sale and transfer funds to the sale and fee receiver is to buy the complete set of NFTs in the sale (i.e. buy everything up to the `sale.finalId`) which will be impossible if the `buy` function is bricked.

## PoC

In the following test, the start price is 1500 and the duration is 1 hour (3600 seconds) with a drop of 1 per second. At about ~40% of the elapsed time the price drop will start underflowing the price, reverting the calls to both `getPrice` and `buy`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "forge-std/Test.sol";
import {FixedPriceFactory} from "src/minters/FixedPriceFactory.sol";
import {FixedPrice} from "src/minters/FixedPrice.sol";
import {OpenEditionFactory} from "src/minters/OpenEditionFactory.sol";
import {OpenEdition} from "src/minters/OpenEdition.sol";
import {LPDAFactory} from "src/minters/LPDAFactory.sol";
import {LPDA} from "src/minters/LPDA.sol";
import {Escher721} from "src/Escher721.sol";

contract AuditTest is Test {
    address deployer;
    address creator;
    address buyer;

    FixedPriceFactory fixedPriceFactory;
    OpenEditionFactory openEditionFactory;
    LPDAFactory lpdaFactory;

    function setUp() public {
        deployer = makeAddr("deployer");
        creator = makeAddr("creator");
        buyer = makeAddr("buyer");

        vm.deal(buyer, 1e18);

        vm.startPrank(deployer);

        fixedPriceFactory = new FixedPriceFactory();
        openEditionFactory = new OpenEditionFactory();
        lpdaFactory = new LPDAFactory();

        vm.stopPrank();
    }
    
    function test_LPDA_getPrice_NegativePrice() public {
        // Setup NFT and create sale
        vm.startPrank(creator);

        Escher721 nft = new Escher721();
        nft.initialize(creator, address(0), "Test NFT", "TNFT");

        // Duration is 1 hour (3600 seconds), with a start price of 1500 and a drop of 1, getPrice will revert and brick the contract at about 40% of the elapsed duration
        uint48 startId = 0;
        uint48 finalId = 1;
        uint80 startPrice = 1500;
        uint80 dropPerSecond = 1;
        uint96 startTime = uint96(block.timestamp);
        uint96 endTime = uint96(block.timestamp + 1 hours);

        LPDA.Sale memory sale = LPDA.Sale(
            startId, // uint48 currentId;
            finalId, // uint48 finalId;
            address(nft), // address edition;
            startPrice, // uint80 startPrice;
            0, // uint80 finalPrice;
            dropPerSecond, // uint80 dropPerSecond;
            endTime, // uint96 endTime;
            payable(creator), // address payable saleReceiver;
            startTime // uint96 startTime;
        );
        LPDA lpdaSale = LPDA(lpdaFactory.createLPDASale(sale));

        nft.grantRole(nft.MINTER_ROLE(), address(lpdaSale));

        vm.stopPrank();

        // simulate we are in the middle of the sale duration
        vm.warp(startTime + 0.5 hours);

        vm.startPrank(buyer);

        // getPrice will revert due to the overflow caused by the price becoming negative
        vm.expectRevert();
        lpdaSale.getPrice();

        // This will also cause the contract to be bricked, since buy needs getPrice to check that the buyer is sending the correct amount
        uint256 amount = 1;
        uint256 price = 1234;
        vm.expectRevert();
        lpdaSale.buy{value: price * amount}(amount);

        vm.stopPrank();
    }
}
```

## Recommendation

Add a validation in the `LPDAFactory.createLPDASale` function to ensure that the given duration and drop per second settings can't underflow the price.

```solidity
require((sale.endTime - sale.startTime) * sale.dropPerSecond <= sale.startPrice, "MAX DROP IS GREATER THAN START PRICE");
```
