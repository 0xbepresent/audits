# Original link
https://github.com/code-423n4/2023-03-wenwin-findings/issues/366
# Lines of code

https://github.com/code-423n4/2023-03-wenwin/blob/main/src/Lottery.sol#L170


# Vulnerability details

## Impact
If the ticket owner lists the winning ticket on the secondary market and initiates their own claiming transaction before the trade transaction takes place, the NFT buyer could lose funds as a result.

## Proof of Concept
Given that there are no restrictions on trading tickets on the secondary market, the following scenario could occur:

1) Alice acquires the winning ticket with 75 DAI worth of claimable rewards, and lists the ticket on the secondary market for 70 DAI.
2) Bob decides to purchase the ticket, recognizing that it is profitable to trade and claim the rewards associated with it.
3) Alice monitors the mempool and submits a `claimWinningTickets()` transaction just before Bob's purchase transaction.
4) Bob receives ticket NFT and calls to `claimWinningTickets()` but this transaction would revert as the rewards have already been claimed by Alice. Consequently, Alice receives a total of 75 + 70 DAI, while Bob is left with an empty ticket and no rewards.

The next test added to `/2023-03-wenwin/test/Lottery.t.sol` could demonstrate such a scenario:

```solidity
    function testClaimBeforeTransfer() public {
        uint128 drawId = lottery.currentDraw();
        uint256 ticketId = initTickets(drawId, 0x8E);

        // this will give winning ticket of 0x0F so 0x8E will have 3/4
        finalizeDraw(0);

        uint8 winTier = 3;
        checkTicketWinTier(drawId, 0x8E, winTier);
        
        address BUYER = address(456);

        vm.prank(USER);
        claimTicket(ticketId); // USER front-run trade transaction and claims rewards 
        vm.prank(USER);
        lottery.transferFrom(USER, BUYER, ticketId);
  
        vm.prank(BUYER);
        vm.expectRevert(abi.encodeWithSelector(NothingToClaim.selector, 0));
        claimTicket(ticketId); // BUYER tries to claim the ticket but it would revert since USER already claimed it
    }
```

## Recommended Mitigation Steps
Consider burning claimed ticket NFTs or the removal possibility to transfer NFTs that have already been claimed.