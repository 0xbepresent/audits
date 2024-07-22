# Original link
https://github.com/code-423n4/2022-12-tigris-findings/issues/377
# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/Trading.sol#L222-L230
https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/StableVault.sol#L78-L83
https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/StableToken.sol#L38-L46
https://github.com/code-423n4/2022-12-tigris/blob/496e1974ee3838be8759e7b4096dbee1b8795593/contracts/PairsContract.sol#L48


# Vulnerability details

The project heavily relies on nodes/oracles, which are EOAs that sign the current price.
Since all functions (including withdrawing) require a recently-signed price, the owner(s) of those EOA can freeze all activity by not providing signed prices.

I got from the sponsor that the owner of the contract is going to be a timelock contract.
However, once the owner holds the power to pause withdrawals - that nullifies the timelock. The whole point of the timelock is to allow users to withdraw their funds when they see a pending malicious tx before it's executed. If the owner has the power to freeze users' funds in the contract, they wouldn't be able to do anything while the owner executes his malicious activity.

Besides that, there are also LP funds, which are locked to a certain period, and also can't withdraw their funds when they see a pending malicious timelock tx.
## Impact
The owner (or attacker who steals the owner's wallet) can steal all user's funds.

## Proof of Concept
* The fact that the protocol relies on EOA signatures is pretty clear from the code and docs
* The whole project relies on the 'StableVault' and 'StableToken'
    * The value of the 'StableToken' comes from the real stablecoin that's locked in 'StableVault', if someone manages to empty the 'StableVault' from the deposited stablecoins the 'StableToken' would become worthless
* The owner has a few ways to drain all funds:
    * Replace the minter via `StableToken.setMinter()`, mint more tokens, and redeem them via `StableVault.withdraw()`
    * List a fake token at `StableVault`, deposit it and withdraw real stablecoin
    * List a new fake asset for trading with a fake chainlink oracle, fake profit with trading with fake prices, and then withdraw
        * They can prevent other users from doing the same by setting `maxOi` and opening position in the same tx
    * Replace the MetaTx forwarder and execute tx on behalf of users (e.g. transferring bonds, positions and StableToken from their account)

## Recommended Mitigation Steps
* Rely on a contract (chainlink/Uniswap) solely as an oracle
* Alternately, add functionality to withdraw funds at the last given price in case no signed data is given for a certain period
    * You can do it by creating a challenge in which a user requests to close his position at a recent price, if no bot executes it for a while it can be executed at the last recorded price.
* As for LPs' funds, I don't see an easy way around it (besides doing significant changes to the architecture of the protocol), this a risk LPs should be aware of and decide if they're willing to accept