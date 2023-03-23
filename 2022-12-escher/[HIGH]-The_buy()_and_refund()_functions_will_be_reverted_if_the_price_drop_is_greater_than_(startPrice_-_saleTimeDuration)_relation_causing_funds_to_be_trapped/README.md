# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/205
# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/LPDA.sol#L58
https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/LPDA.sol#L99
https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/LPDA.sol#L117


# Vulnerability details

## Impact

The user can set the ```dropPerSecond``` variable in the [LPDA Creation](https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/LPDAFactory.sol#L29) The problem is that if the user set a ```dropPerSecond``` number greater than the (startPrice / saleTimeDuration) the [LPDA.sol::buy()](https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/LPDA.sol#L58) and [LPDA.sol::refund()](https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/LPDA.sol#L99) functions will be reverted. Then if someone buys (deposit Ether) a NFT, the buyer's funds will be trapped in the contract.


## Proof of Concept

I made a test where you can see the problem:

1. Creates LPDA.Sale with a big ```dropPerSecond``` number.
2. After one day has passed. As a user address(13372) buys one NFT.
3. After two days has passed. As another user address(1337) try to buy one NFT.
    The buy function will be reverted by a arithmetic over/undeflow error.
4. The user address(13372) can not reclaim a refund.
    The refund function will be reverted by a arithmetic over/undeflow error.
5. The funds that address(13372) put at the beginning are trapped in the Sale Contract.
    No one can take them out.


```solidity
// LPDA.t.sol
// $ forge test -m "test_BigDropPerSecondWillRevertBuyAndRefund" -vvv
function test_BigDropPerSecondWillRevertBuyAndRefund() public {
    //
    // Buy and Refund function will be reverted if the priceDrop is grater than (startPrice / endTime) relation.
    // 1. Creates LPDA.Sale with a big dropPerSecond.
    // 2. After one day has passed. As a user address(13372) buys one NFT.
    // 3. After two days has passed. As another user address(1337) try to buy one NFT.
    //     The buy function will be reverted by a arithmetic over/undeflow error
    // 4. The user address(13372) can not reclaim a refund.
    //     The refund function will be reverted by a arithmetic over/undeflow error
    // 5. The funds that address(13372) put at the beginning are trapped in the Sale Contract.
    //     No one can take them out.
    //
    // 1. Creates LPDA.Sale with a big dropPerSecond.
    //
    LPDA.Sale memory lpdaCustomSale = LPDA.Sale({
        currentId: uint48(0),
        finalId: uint48(10),
        edition: address(edition),
        startPrice: uint80(uint256(1 ether)),
        finalPrice: uint80(uint256(0.1 ether)),
        dropPerSecond: uint80(uint256(0.50 ether) / 1 days), // The price drops 0.5 each day
        startTime: uint96(block.timestamp),
        saleReceiver: payable(address(69)),
        endTime: uint96(block.timestamp + 4 days)
    });
    // Create the LPDA implementation
    sale = LPDA(lpdaSales.createLPDASale(lpdaCustomSale));
    // Authorize the lpda sale implementation to mint tokens
    edition.grantRole(edition.MINTER_ROLE(), address(sale));
    //
    // 2.  After one day has passed. As a user address(13372) buys one NFT.
    //
    vm.warp(block.timestamp + 1 days);
    uint80 price = uint80(sale.getPrice());
    uint256 amountToDepositFirstUser = price;
    vm.deal(address(13372), amountToDepositFirstUser);
    vm.startPrank(address(13372));
    sale.buy{value: amountToDepositFirstUser}(1);
    assertEq(address(sale).balance, amountToDepositFirstUser);
    (,uint80 userSaleBalance) = sale.receipts(address(13372));
    assertEq(userSaleBalance, amountToDepositFirstUser);
    console.log("First sale after 1 day. Now Sale balance:", address(sale).balance);
    console.log("Address(13372) balance:", address(13372).balance);
    vm.stopPrank();
    //
    // 3. After two days has passed, as another user address(1337) try to buy one NFT.
    //     The buy transaction will be reverted by a arithmetic over/undeflow error
    console.log("User address(1337) try to buy but tha function will be reverted");
    vm.warp(block.timestamp + 2 days);
    uint256 amountToDepositSecondUser = 1 ether;
    vm.deal(address(1337), amountToDepositSecondUser);
    vm.startPrank(address(1337));
    vm.expectRevert(stdError.arithmeticError);  // <-- Over/UnderFlow error
    sale.buy{value: amountToDepositSecondUser}(1);
    vm.stopPrank();
    //
    // 4. The user address(13372) can not reclaim a refund.
    //     The refund function will be reverted by a arithmetic over/undeflow error
    console.log("User address(13372) try to refund but tha function will be reverted");
    vm.prank(address(13372));
    vm.expectRevert(stdError.arithmeticError);  // <-- Over/UnderFlow error
    sale.refund();
    //
    // 5. The funds that address(13372) put at the beginning are trapped in the Sale Contract.
    //     No one can take them out.
    //
    console.log("Balance trapped in the Sale Contract:", address(sale).balance);
    assertEq(address(sale).balance, amountToDepositFirstUser);
    assertEq(address(13372).balance, 0);
}
```


## Tools used
VsCode/Foundry

## Recommended Mitigation Steps

I think that the ```dropPerSecond``` should not be greater than the (startPrice / saleTimeDuration) and add that validation in the [LPDA creation](https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/LPDAFactory.sol#L29) in order to prevent trapped funds in the contract.