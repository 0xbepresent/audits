# Original link
https://github.com/code-423n4/2023-03-wenwin-findings/issues/315
# Lines of code

https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/Ticket.sol#L12
https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/Lottery.sol#L170


# Vulnerability details

## Impact
The [Ticket](https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/Ticket.sol#L12) contract is an ERC721 which helps to claim a reward if it is a winning ticket. The [documentation](https://docs.wenwin.com/wenwin-lottery/the-game/tickets) says: *Can be traded on the secondary market before or after the draw* but the problem is that the Ticket can be traded even when the rewards are already claimed.

A malicious winner can market his already claimed ticket.

## Proof of Concept

I created a basic test in ```Lottery.t.sol``` where it is possible to transfer a claimed ticket.

1. USER buys a ticketId.
2. The ticketId is selected as a winner.
3. The USER claim his rewards via [claimWinningTickets()](https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/Lottery.sol#L170).
4. The USER transfer ticketId to address(1337).
5. address(1337) can not claim an already claimed ticketId.

```solidity
function testTransferClaimedTicket() public {
    // Transfer claimed ticket
    // 1. USER buys a ticketId
    // 2. The ticketId is selected as a winner
    // 3. The USER claim his rewards
    // 4. The USER transfer ticketId to address(1337)
    // 5. address(1337) can not claim an already claimed ticketId
    //
    // 1. USER buys a ticket
    //
    uint128 drawId = lottery.currentDraw();
    vm.startPrank(USER);
    rewardToken.mint(TICKET_PRICE);
    rewardToken.approve(address(lottery), TICKET_PRICE);
    uint256 ticketId = buyTicket(lottery.currentDraw(), uint120(0x0F), address(0));
    vm.stopPrank();
    // Execute draw.
    vm.warp(block.timestamp + 60 * 60 * 24);
    lottery.executeDraw();
    //
    // 2. The ticketId is selected as a winner
    //
    uint256 randomNumber = 0x00000000;
    vm.prank(randomNumberSource);
    lottery.onRandomNumberFulfilled(randomNumber);
    //
    // 3. The USER claim his rewards
    //
    vm.prank(USER);
    claimTicket(ticketId);
    assertEq(rewardToken.balanceOf(USER), lottery.winAmount(drawId, SELECTION_SIZE));
    //
    // 4. The USER transfer ticketId to address(1337)
    //
    vm.prank(USER);
    lottery.transferFrom(USER, address(1337), ticketId);
    assertEq(lottery.ownerOf(ticketId), address(1337));
    //
    // 5. address(1337) can not claim an already claimed ticketId
    //
    vm.prank(address(1337));
    vm.expectRevert(abi.encodeWithSelector(NothingToClaim.selector, ticketId));
    claimTicket(ticketId);
    assertEq(rewardToken.balanceOf(address(1337)), 0);
}
```

## Tools used

Foundry/Vscode

## Recommended Mitigation Steps

Burn the ticket once the rewards are claimed.