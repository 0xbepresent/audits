# DittoETH - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. When the short record is created the collateral amount is not counted/saved correctly causing the user to receive more collateral when exits his `short` position](#H-01)
- ## Medium Risk Findings
    - ### [M-01. The `LibAsset::minShortErc` and `LibAsset::minAskEth` can be bypassed causing that the created `short orders` with the minimum values would not incentivize the liquidators](#M-01)
    - ### [M-02. The liquidation action won't be available in the last hour specified in the `resetLiquidationTime()` time frame](#M-02)
- ## Low Risk Findings
    - ### [L-01. Malicious trader can intentionally obtain `dittoMatchedShares` in some edges cases](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Ditto

### Dates: Sep 7th, 2023 - Oct 8th, 2023

[See more contest details here](https://www.codehawks.com/contests/clm871gl00001mp081mzjdlwc)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 2
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. When the short record is created the collateral amount is not counted/saved correctly causing the user to receive more collateral when exits his `short` position            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L763

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BidOrdersFacet.sol#L290

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BidOrdersFacet.sol#L301

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L75

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/DataTypes.sol#L74

## Summary

The `shortRecord.collateral` amount is not counted/saved correctly causing that the `shorter` receives more collateral when he exits his short position using the `ExitShortFacet::exitShortWallet()` function.

## Vulnerability Details

When the `shorter` match his order a [`shortRecord` is created](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L759) using the [LibShortRecord::createShortRecord()](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L71C14-L71C31) function. The problem is that the short record creation is using the [`matchTotal.fillEth`](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L763) as the [collateral parameter](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L75) for the `LibShortRecord::createShortRecord()` function. Causing that user [receives more collateral](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ExitShortFacet.sol#L68) when the `shorter` exits his position using [ExitShortFacet::exitShortWallet()](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ExitShortFacet.sol#L43C14-L43C29) function. Please see the next scenario:

**Match orders:**
```
InitialMargin: 500
Short order: Price: 10 ercAmount: 5
Bid order: Price: 10, ercAmount: 3. There is only one bid order in the order book.
```

1. Shorter opens his position and its matched with a bid order, then the [fillErc](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L793) will be `3` and the [fillEth](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L796) will be `30` (highestBid * fillErc == 10 * 3). 
2. The [collateral used is calculated](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L801) as `incomingSell.price * fillErc * initialMargin == 10 * 3 * 500 == 15000`
3. [matchTotal.fillEth](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L806) == 30 and [matchTotal.colUsed](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L801C13-L801C31) == 15000
4. Now, since there is not more `bid` orders, the [matchIncomingShort()](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L744) is executed, the [collateral used (15000) is deacreased](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L754) from the shorter and the collateral used is increased on the [matchTotal.fillEth](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L755) variable, so now the `matchTotal.fillEth == 15030 (15000 collateral + 30 fillEth)`.
5. Then, the [shortRecord](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L759) is created and it uses the [matchTotal.fillEth](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L763) as the collateral amount, the [LibShortRecord::createShortRecord() uses that parameter as the collateral amount](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L75), then it saves the [collateral amount](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L95).
6. The problem is exactly that, the user is saving more collateral in his `short record` (15030 instead of 15000). When user exists his position he will [receive more collateral (15030)](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ExitShortFacet.sol#L68) than the used (15000).

Additionally, the ShortRecord struct specifies that the [collateral parameter](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/DataTypes.sol#L74) should be the calculation of `price * ercAmount * initialMargin` NOT `price * ercAmount * initialMargin + ethFilled`

## Impact

Shorter will receive more collateral than used if he decides to exit form his position.

## Tools used

Manual review

## Recommendations

The `short record creation` should use the collateral used:

```diff
File: LibOrders.sol
    function matchIncomingShort(
        address asset,
        STypes.Order memory incomingShort,
        MTypes.Match memory matchTotal
    ) private returns (uint8 shortRecordId) {
        AppStorage storage s = appStorage();
        STypes.Asset storage Asset = s.asset[asset];
        uint256 vault = Asset.vault;
        STypes.Vault storage Vault = s.vault[vault];

        s.vaultUser[vault][incomingShort.addr].ethEscrowed -= matchTotal.colUsed;
        matchTotal.fillEth += matchTotal.colUsed;

        SR status = incomingShort.ercAmount == 0 ? SR.FullyFilled : SR.PartialFill;

        shortRecordId = LibShortRecord.createShortRecord(
            asset,
            incomingShort.addr,
            status,
--          matchTotal.fillEth,
++          matchTotal.collUsed,
            matchTotal.fillErc,
            Asset.ercDebtRate,
            Vault.zethYieldRate,
            0
        );

        Vault.dittoMatchedShares += matchTotal.dittoMatchedShares;
--      Vault.zethCollateral += matchTotal.fillEth;
--      Asset.zethCollateral += matchTotal.fillEth;
++      Vault.zethCollateral += matchTotal.collUsed;
++      Asset.zethCollateral += matchTotal.collUsed;
        Asset.ercDebt += matchTotal.fillErc;
    }
```

```diff
File: BidOrdersFacet.sol
    function matchlowestSell(
        address asset,
        STypes.Order memory lowestSell,
        STypes.Order memory incomingBid,
        MTypes.Match memory matchTotal
    ) private {
        uint88 fillErc = incomingBid.ercAmount > lowestSell.ercAmount
            ? lowestSell.ercAmount
            : incomingBid.ercAmount;
        uint88 fillEth = lowestSell.price.mulU88(fillErc);

        if (lowestSell.orderType == O.LimitShort) {
            // Match short
            uint88 colUsed = fillEth.mulU88(LibOrders.convertCR(lowestSell.initialMargin));
            LibOrders.increaseSharesOnMatch(asset, lowestSell, matchTotal, colUsed);
            uint88 shortFillEth = fillEth + colUsed;
            matchTotal.shortFillEth += shortFillEth;
++          matchTotal.colUsed += colUsed;
            // Saves gas when multiple shorts are matched
            if (!matchTotal.ratesQueried) {
                STypes.Asset storage Asset = s.asset[asset];
                matchTotal.ratesQueried = true;
                matchTotal.ercDebtRate = Asset.ercDebtRate;
                matchTotal.zethYieldRate = s.vault[Asset.vault].zethYieldRate;
            }
            // Default enum is PartialFill
            SR status;
            if (incomingBid.ercAmount >= lowestSell.ercAmount) {
                status = SR.FullyFilled;
            }
            if (lowestSell.shortRecordId > 0) {
                // shortRecord has been partially filled before
                LibShortRecord.fillShortRecord(
                    asset,
                    lowestSell.addr,
                    lowestSell.shortRecordId,
                    status,
--                  shortFillEth,
++                  colUsed,
                    fillErc,
                    matchTotal.ercDebtRate,
                    matchTotal.zethYieldRate
                );
            } else {
                // shortRecord newly created
                lowestSell.shortRecordId = LibShortRecord.createShortRecord(
                    asset,
                    lowestSell.addr,
                    status,
--                  shortFillEth,
++                  colUsed,
                    fillErc,
                    matchTotal.ercDebtRate,
                    matchTotal.zethYieldRate,
                    0
                );
            }
        } else {
            // Match ask
            s.vaultUser[s.asset[asset].vault][lowestSell.addr].ethEscrowed += fillEth;
            matchTotal.askFillErc += fillErc;
        }

        matchTotal.fillErc += fillErc;
        matchTotal.fillEth += fillEth;
    }
```

```diff
File: BidOrdersFacet.sol
    function matchIncomingBid(
        address asset,
        STypes.Order memory incomingBid,
        MTypes.Match memory matchTotal,
        MTypes.BidMatchAlgo memory b
    ) private returns (uint88 ethFilled, uint88 ercAmountLeft) {
        if (matchTotal.fillEth == 0) {
            return (0, incomingBid.ercAmount);
        }

        STypes.Asset storage Asset = s.asset[asset];
        uint256 vault = Asset.vault;

        LibOrders.updateSellOrdersOnMatch(asset, b);

        // If at least one short was matched
        if (matchTotal.shortFillEth > 0) {
            STypes.Vault storage Vault = s.vault[vault];

            // Matched Shares
            Vault.dittoMatchedShares += matchTotal.dittoMatchedShares;
            // Yield Accounting
--          Vault.zethCollateral += matchTotal.shortFillEth;
--          Asset.zethCollateral += matchTotal.shortFillEth;
++          Vault.zethCollateral += matchTotal.colUsed;
++          Asset.zethCollateral += matchTotal.colUsed;
            Asset.ercDebt += matchTotal.fillErc - matchTotal.askFillErc;

            //@dev Approximates the startingShortId after bid is fully executed
            O shortOrderType = s.shorts[asset][b.shortId].orderType;
            O prevShortOrderType = s.shorts[asset][b.prevShortId].orderType;
            uint80 prevShortPrice = s.shorts[asset][b.prevShortId].price;

            if (shortOrderType != O.Cancelled && shortOrderType != O.Matched) {
                Asset.startingShortId = b.shortId;
            } else if (
                prevShortOrderType != O.Cancelled && prevShortOrderType != O.Matched
                    && prevShortPrice >= b.oraclePrice
            ) {
                Asset.startingShortId = b.prevShortId;
            } else {
                if (b.isMovingFwd) {
                    Asset.startingShortId = s.shorts[asset][b.shortId].nextId;
                } else {
                    Asset.startingShortId = s.shorts[asset][b.shortHintId].nextId;
                }
            }
        }

        // Match bid
        s.vaultUser[vault][incomingBid.addr].ethEscrowed -= matchTotal.fillEth;
        s.assetUser[asset][incomingBid.addr].ercEscrowed += matchTotal.fillErc;
        return (matchTotal.fillEth, incomingBid.ercAmount);
    }
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. The `LibAsset::minShortErc` and `LibAsset::minAskEth` can be bypassed causing that the created `short orders` with the minimum values would not incentivize the liquidators            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ShortOrdersFacet.sol#L53

## Summary

The `short` can be created with values less than the `minAskEth` and `minShortErc` causing that the created order with minimum values would not be attractive to be liquidated if it is the case.

## Vulnerability Details

The `short order` creation is possible using the [ShortOrdersFacet::createLimitShort()](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ShortOrdersFacet.sol#L34C14-L34C30) function. It [checks for `minAskEth` and `minShortErc`](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ShortOrdersFacet.sol#L53) minimum values before everything else. The problem is that the number of `bids orders` can not be enough to complete the `short order` causing the creation of the short order with minimum values. Please consider the next scenario:

```
Current system status:
minBidEth: 1
minAskEth: 2
minShortErc: 2

Current orderbook status:
Bid orders - ercAmount: 1, price: 1
Short orders: 0
```

1. The `new short order` is executed with `ercAmount: 2, price: 1, initialMargin: 500`, the [next validation](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ShortOrdersFacet.sol#L53) does not rever the transaction (2 < 2 || 2 < 2 is False): 

```solidity
File: ShortOrdersFacet.sol
53:         if (ercAmount < p.minShortErc || p.eth < p.minAskEth) {
54:             revert Errors.OrderUnderMinimumSize();
55:         }
```
2. The `new short order` is matched with the unique bid order avaliable. The [`fillErc = 1`](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L793) and [`fillEth = 1`](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L796). The [`matchTotal.colUsed`](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L801) is `1 (incomingSellPrice) * 1 (fillErc) * 500 (initialMargin) = 500`
3. Since there is only one bid order, the [matching process ends](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L662-L670) and the [next validation in the code line 666](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L666) is not True so the leftover incomingSell.ercAmount (1 amount) is not assigned to the orderbook (`1 [left ercAmount] * 1 [incomingSellPrice] >= 2 is False`).
```solidity
File: LibOrders.sol
666:                         if (incomingAsk.ercAmount.mul(incomingAsk.price) >= minAskEth) {
667:                             addSellOrder(incomingAsk, asset, orderHintArray);
668:                         }
```
4. The [user short record](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L759) creation ends with the next fills:
```
matchTotal.fillErc = 1
matchTotal.fillEth = 1
matchTotal.colUsed = 500
```

That is less than the required by [minAskEth](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibAsset.sol#L146) and [minShortErc](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibAsset.sol#L156).


## Impact

Shorts can be created with the less than the minimum value required by `minAskEth` and `minShortErc` causing that the `short order` would not be attractive to be liquidated by the liquidatiors since the collateral attached is not enough to cover the liquidation transaction.

## Tools used

Manual review

## Recommendations

Verify the minimum required values `minAskEth` and `minShortErc` in the short record creation [LibShortRecord::createShortRecord()](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L71) function.
## <a id='M-02'></a>M-02. The liquidation action won't be available in the last hour specified in the `resetLiquidationTime()` time frame            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallPrimaryFacet.sol#L387

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallPrimaryFacet.sol#L395

## Summary

The liquidation action won't be available when the current time is in the last hour of the `resetLiquidationTime()` time frame. Causing that the time frame for liquidation will be less than expected by the `resetLiquidationTime()`.

## Vulnerability Details

The [MarginCallPrimaryFacet::_canLiquidate()](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallPrimaryFacet.sol#L351C14-L351C27) function helps to determinate if the `short` is able to be liquidated or not. If no one has liquidated during 16 hours ([resetLiquidationTime()](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibAsset.sol#L19)) the flag will be reset and the flagging process begins anew. As the [documentation says](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallPrimaryFacet.sol#L343): 

```
* @dev Shorter has 10 hours after initial flag to bring cRatio up above maintainence margin...
* @dev ...After that, the flagger has 2 hours to liquidate the shorter. If short is not liquidated by shorter within that time, ANYBODY can then liquidate...
* @dev ...After 16 total hours have passed and the short has not been liquidated, the flag gets reset and the flagging process begins anew
```

The problem is that the `short` can't be liquidated exactly at the `hour 16` because in the [code line 387](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallPrimaryFacet.sol#L387) the if statement uses a `>=` so the [code line 395](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallPrimaryFacet.sol#L395) `timeDiff <= resetLiquidationTime` is not possible to be executed:

```solidity
File: MarginCallPrimaryFacet.sol
383: 
384:         uint256 timeDiff = LibOrders.getOffsetTimeHours() - m.short.updatedAt;
385:         uint256 resetLiquidationTime = LibAsset.resetLiquidationTime(m.asset);
386: 
387:         if (timeDiff >= resetLiquidationTime) {
388:             return false;
389:         } else {
390:             uint256 secondLiquidationTime = LibAsset.secondLiquidationTime(m.asset);
391:             bool isBetweenFirstAndSecondLiquidationTime = timeDiff
392:                 > LibAsset.firstLiquidationTime(m.asset) && timeDiff <= secondLiquidationTime
393:                 && s.flagMapping[m.short.flaggerId] == msg.sender;
394:             bool isBetweenSecondAndResetLiquidationTime =
395:                 timeDiff > secondLiquidationTime && timeDiff <= resetLiquidationTime;
396:             if (
397:                 !(
398:                     (isBetweenFirstAndSecondLiquidationTime)
399:                         || (isBetweenSecondAndResetLiquidationTime)
400:                 )
401:             ) {
402:                 revert Errors.MarginCallIneligibleWindow();
403:             }
404: 
405:             return true;
406:         }
```

## Impact

The liquidaton won't be possible at the last hour specified in the `resetLiquidationTime()` time frame. The `short` should possible to be liquidated between the time frame from hour 11 to hour 16. Liquidators have less time to be able liquidate `shorts`.

## Tools used

Manual review

## Recommendations

Change the next line so liquidation is possible in the last hour of the `resetLiquidationTime()` time frame:

```diff
function _canLiquidate(MTypes.MarginCallPrimary memory m)
        private
        view
        returns (bool)
    {
        ...
        ...

        uint256 timeDiff = LibOrders.getOffsetTimeHours() - m.short.updatedAt;
        uint256 resetLiquidationTime = LibAsset.resetLiquidationTime(m.asset);

--      if (timeDiff >= resetLiquidationTime) {
++      if (timeDiff > resetLiquidationTime) {
            return false;
        } else {
            uint256 secondLiquidationTime = LibAsset.secondLiquidationTime(m.asset);
            bool isBetweenFirstAndSecondLiquidationTime = timeDiff
                > LibAsset.firstLiquidationTime(m.asset) && timeDiff <= secondLiquidationTime
                && s.flagMapping[m.short.flaggerId] == msg.sender;
            bool isBetweenSecondAndResetLiquidationTime =
                timeDiff > secondLiquidationTime && timeDiff <= resetLiquidationTime;
            if (
                !(
                    (isBetweenFirstAndSecondLiquidationTime)
                        || (isBetweenSecondAndResetLiquidationTime)
                )
            ) {
                revert Errors.MarginCallIneligibleWindow();
            }

            return true;
        }
    }
```

# Low Risk Findings

## <a id='L-01'></a>L-01. Malicious trader can intentionally obtain `dittoMatchedShares` in some edges cases            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L39

## Summary

Malicious trader can intentionally obtain `dittoMatchedShares` by creating a bid order using a low price that nobody will ask, then wait for more than 14 days and the same malicious trader create an ask order using the same bid's low price causing the increase of `dittoMatchedShares`.

## Vulnerability Details

Malicious trader can create a bid order using the [BidOrdersFacet::createBid()](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BidOrdersFacet.sol#L39C14-L39C23) function at very low price, then the same malicious trader can wait [some days](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/Constants.sol#L14) until the minumum required in [order to get `dittoMatchedShares`](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L49) and set a `ask` order using the bid's low price. Please consider the next scenario:

```
Market status:
assetX: current price 100
```

1. Malicious trader [creates](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BidOrdersFacet.sol#L39C14-L39C23) the `bid order` for the `assetX` using the `price: 10` (low price compared to the current 100 price) and `ercAmount 10`. The low price is because nobody wants to sell at that price so the order can stay there without be matched.
2. The bid order will be submitted to the order book because there are not `asks/sells` to fill at that price.
3. Malicious trader waits for more than 14 days. Additionally the malicious trader needs to wait until there are not `asks/sells` in the order book.
4. Once the step 3 is ok, the Malicious trader creates the `ask order` at `price 10 and ercAmount10` (the bid's order price from step 1). The [order is matched](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L798) with the `bid order` from the step 1 and `dittoMatchedShares` are [assigned to the malicious trader](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L55).

It is a very edge case because the malicious trader needs an empty `ask/sells` orderbook so he can put his own `ask order` at the malicious bid order price but in conditions where the asset is not very trader the malicious actor can benefit from this.

## Impact

Malicious actor can intentionally obtain `dittoMatchedShares` using `bid/asks` orders that he intentionally crafts. The `bid/ask` orders are created by the same malicious actor, so he won't lose assets.

## Tools used

Manual review

## Recommendations

Verify that the address from the `bid order` is not the same address who is creating the `ask` order.


