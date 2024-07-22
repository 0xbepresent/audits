# Original link
https://github.com/code-423n4/2023-03-asymmetry-findings/issues/812
# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L63
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/WstEth.sol#L73-L81
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L156-L204
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L170
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L120-L150


# Vulnerability details

## Impact
The users will not be able to stake their funds and there will be loss of reputation
## Proof of Concept
The `SafEth` contract's [`stake()`](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L63) function is the main entry point to add liquid Eth to the derivatives. Accordingly the `stake()` function takes the users' ETH and convert it into various derivatives based on their weights and mint an amount of safETH that represents a percentage of the total assets in the system.

The execution to deposit to the available derivative is done through iterating the derivatives mapping in [`SafEthStorage`](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEthStorage.sol#L22) contract.

```solidity
function stake() external payable {
    require(pauseStaking == false, "staking is paused");
    require(msg.value >= minAmount, "amount too low");
    require(msg.value <= maxAmount, "amount too high");

    uint256 underlyingValue = 0;

    // Getting underlying value in terms of ETH for each derivative
    for (uint i = 0; i < derivativeCount; i++)
        underlyingValue +=
            (derivatives[i].ethPerDerivative(derivatives[i].balance()) *
                derivatives[i].balance()) /
            10 ** 18;

    uint256 totalSupply = totalSupply();
    uint256 preDepositPrice; // Price of safETH in regards to ETH
    if (totalSupply == 0)
        preDepositPrice = 10 ** 18; // initializes with a price of 1
    else preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;

    uint256 totalStakeValueEth = 0; // total amount of derivatives worth of ETH in system
    for (uint i = 0; i < derivativeCount; i++) {
        uint256 weight = weights[i];
        IDerivative derivative = derivatives[i];
        if (weight == 0) continue;
        uint256 ethAmount = (msg.value * weight) / totalWeight;

        // This is slightly less than ethAmount because slippage
        uint256 depositAmount = derivative.deposit{value: ethAmount}();
        uint derivativeReceivedEthValue = (derivative.ethPerDerivative(
            depositAmount
        ) * depositAmount) / 10 ** 18;
        totalStakeValueEth += derivativeReceivedEthValue;
    }
    // mintAmount represents a percentage of the total assets in the system
    uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
    _mint(msg.sender, mintAmount);
    emit Staked(msg.sender, msg.value, mintAmount);
}
```

And for all the derivatives the stake function calls the derivative contract's `deposit()` function. Below is for [`WstEth`](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/WstEth.sol#L73-L81) contract's `deposit()` function to adapt to Lido staking;
```solidity
function deposit() external payable onlyOwner returns (uint256) {
    uint256 wstEthBalancePre = IWStETH(WST_ETH).balanceOf(address(this));
    // solhint-disable-next-line
    (bool sent, ) = WST_ETH.call{value: msg.value}("");
    require(sent, "Failed to send Ether");
    uint256 wstEthBalancePost = IWStETH(WST_ETH).balanceOf(address(this));
    uint256 wstEthAmount = wstEthBalancePost - wstEthBalancePre;
    return (wstEthAmount);
}
```
The Lido protocol implements a daily staking limit both for `stETH` and `WstETH` as per their [docs](https://docs.lido.fi/guides/steth-integration-guide/#staking-rate-limits)
Accordingly the daily rate is 150000 ETH and the `deposit()` function will revert if the limit is hit. From the docs:
>***Staking rate limits***
In order to handle the staking surge in case of some unforeseen market conditions, the Lido protocol implemented staking rate limits aimed at reducing the surge's impact on the staking queue & Lido’s socialized rewards distribution model. There is a sliding window limit that is parametrized with _maxStakingLimit and _stakeLimitIncreasePerBlock. This means it is only possible to submit this much ether to the Lido staking contracts within a 24 hours timeframe. Currently, the daily staking limit is set at ***150,000 ether***.
You can picture this as a health globe from Diablo 2 with a maximum of _maxStakingLimit and regenerating with a constant speed per block. When you deposit ether to the protocol, the level of health is reduced by its amount and the current limit becomes smaller and smaller. ***When it hits the ground, transaction gets reverted.***
To avoid that, you should check if `getCurrentStakeLimit() >= amountToStake`, and if it's not you can go with an alternative route. The staking rate limits are denominated in ether, thus, it makes no difference if the stake is being deposited for stETH or using the wstETH shortcut, the limits apply in both cases.

However this check was not done either in `SafEth::stake()` or `WstEth::deposit()` functions. So if the function reverts, the stake function will all revert and it will not be possible to deposit to the other derivatives as well. 



#### Another issue lies in the Reth contract having the same root cause below - Missing Validation & external tx dependency.

For all the derivatives the stake function calls the derivative contract's `deposit()` function. Below is `rETH` contract's [`deposit()`](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L156-L204) function;
```solidity
function deposit() external payable onlyOwner returns (uint256) {
    // Per RocketPool Docs query addresses each time it is used
    address rocketDepositPoolAddress = RocketStorageInterface(
        ROCKET_STORAGE_ADDRESS
    ).getAddress(
            keccak256(
                abi.encodePacked("contract.address", "rocketDepositPool")
            )
        );

    RocketDepositPoolInterface rocketDepositPool = RocketDepositPoolInterface(
            rocketDepositPoolAddress
        );

    if (!poolCanDeposit(msg.value)) {
        uint rethPerEth = (10 ** 36) / poolPrice();

        uint256 minOut = ((((rethPerEth * msg.value) / 10 ** 18) *
            ((10 ** 18 - maxSlippage))) / 10 ** 18);

        IWETH(W_ETH_ADDRESS).deposit{value: msg.value}();
        uint256 amountSwapped = swapExactInputSingleHop(
            W_ETH_ADDRESS,
            rethAddress(),
            500,
            msg.value,
            minOut
        );

        return amountSwapped;
    } else {
        address rocketTokenRETHAddress = RocketStorageInterface(
            ROCKET_STORAGE_ADDRESS
        ).getAddress(
                keccak256(
                    abi.encodePacked("contract.address", "rocketTokenRETH")
                )
            );
        RocketTokenRETHInterface rocketTokenRETH = RocketTokenRETHInterface(
            rocketTokenRETHAddress
        );
        uint256 rethBalance1 = rocketTokenRETH.balanceOf(address(this));
        rocketDepositPool.deposit{value: msg.value}();
        uint256 rethBalance2 = rocketTokenRETH.balanceOf(address(this));
        require(rethBalance2 > rethBalance1, "No rETH was minted");
        uint256 rethMinted = rethBalance2 - rethBalance1;
        return (rethMinted);
    }
}
```
At [Line#170](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L170) it checks the pools availability to deposit with `poolCanDeposit(msg.value)`;

[`PoolCanDeposit`](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L120-L150) function below;
```solidity
function poolCanDeposit(uint256 _amount) private view returns (bool) {
    address rocketDepositPoolAddress = RocketStorageInterface(
        ROCKET_STORAGE_ADDRESS
    ).getAddress(
            keccak256(
                abi.encodePacked("contract.address", "rocketDepositPool")
            )
        );
    RocketDepositPoolInterface rocketDepositPool = RocketDepositPoolInterface(
            rocketDepositPoolAddress
        );
    address rocketProtocolSettingsAddress = RocketStorageInterface(
        ROCKET_STORAGE_ADDRESS
    ).getAddress(
            keccak256(
                abi.encodePacked(
                    "contract.address",
                    "rocketDAOProtocolSettingsDeposit"
                )
            )
        );
    RocketDAOProtocolSettingsDepositInterface rocketDAOProtocolSettingsDeposit = RocketDAOProtocolSettingsDepositInterface(
            rocketProtocolSettingsAddress
        );
    return
        rocketDepositPool.getBalance() + _amount <=
        rocketDAOProtocolSettingsDeposit.getMaximumDepositPoolSize() &&
        _amount >= rocketDAOProtocolSettingsDeposit.getMinimumDeposit();
}
```

However, as per Rocket Pool's [`RocketDepositPool`](https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/deposit/RocketDepositPool.sol#L77) contract, there is an other check to confirm the availability of the intended deposit;
```solidity
require(rocketDAOProtocolSettingsDeposit.getDepositEnabled(), "Deposits into Rocket Pool are currently disabled");
```

The `Reth::deposit()` function doesn't check this requirement whether the deposits are disabled. As a result, the `SafEth::stake()` function will all revert and it will not be possible to deposit to the other derivatives as well. 



## Tools Used
Manual Review
## Recommended Mitigation Steps
1. For WstETH contract; checking the daily limit via `getCurrentStakeLimit() >= amountToStake`
2. For Reth contract; Checking the Rocket Pool's deposit availability
3. Wrap the `stake()` function's iteration inside `try/catch` block to make the transaction success until it reverts.