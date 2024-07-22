# Original link
https://github.com/code-423n4/2023-03-asymmetry-findings/issues/770
# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L71-L91
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L113-L119
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L140-L153
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L66-L127
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L60-L106
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/WstEth.sol#L61


# Vulnerability details

## Impact

When `stake()/unstake()/rebalanceToWeights()`, if any one of the derivatives fails to `deposit()/withdraw()`, the whole function will revert, causing DoS. The impacts include:
- users' fund would be locked for a period.
- contract become inoperable until the external call resumes


## Proof of Concept

In `stake()`, each derivative iteration, `ethPerDerivative()` and `deposit()` will be called:
```solidity
File: contracts/SafEth/SafEth.sol
63:     function stake() external payable {

71:         for (uint i = 0; i < derivativeCount; i++)
72:             underlyingValue +=
73:                 (derivatives[i].ethPerDerivative(derivatives[i].balance()) *
74:                     derivatives[i].balance()) /
75:                 10 ** 18;

84:         for (uint i = 0; i < derivativeCount; i++) {

91:             uint256 depositAmount = derivative.deposit{value: ethAmount}();

```

In `unstake()`, each derivative is iterated to `withdraw()`:
```solidity
File: contracts/SafEth/SafEth.sol
108:     function unstake(uint256 _safEthAmount) external {

113:         for (uint256 i = 0; i < derivativeCount; i++) {

118:             derivatives[i].withdraw(derivativeAmount);
119:         }
```

In `rebalanceToWeights()`, each derivative is iterated to `withdraw()` and then `deposit()`:
```solidity
138:     function rebalanceToWeights() external onlyOwner {
139:         uint256 ethAmountBefore = address(this).balance;
140:         for (uint i = 0; i < derivativeCount; i++) {
141:             if (derivatives[i].balance() > 0)
142:                 derivatives[i].withdraw(derivatives[i].balance());
143:         }

147:         for (uint i = 0; i < derivativeCount; i++) {
148:             if (weights[i] == 0 || ethAmountToRebalance == 0) continue;
149:             uint256 ethAmount = (ethAmountToRebalance * weights[i]) /
150:                 totalWeight;
151:             // Price will change due to slippage
152:             derivatives[i].deposit{value: ethAmount}();
153:         }

```

For each of the current derivatives, there are several different scenarios where the `ethPerDerivative()/deposit()/withdraw()` could fail.

#### SfrxEth.sol

- `redeem()` could fail due to not enough allowance.
```solidity
File: contracts/SafEth/derivatives/SfrxEth.sol
60:     function withdraw(uint256 _amount) external onlyOwner {

61:         IsFrxEth(SFRX_ETH_ADDRESS).redeem(
62:             _amount,
63:             address(this),
64:             address(this)
65:         );
```

Below is sFrxEth contract code:
```solidity
// https://etherscan.io/address/0xac3E018457B222d93114458476f3E3416Abbe38F
// line 691-700
    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) public virtual returns (uint256 assets) {
        if (msg.sender != owner) {
            uint256 allowed = allowance[owner][msg.sender]; // Saves gas for limited approvals.

            if (allowed != type(uint256).max) allowance[owner][msg.sender] = allowed - shares;
        }
```

- FrxEthEthPool `exchange()` could fail due to `minOut` requirement.
```solidity
File: contracts/SafEth/derivatives/SfrxEth.sol
60:     function withdraw(uint256 _amount) external onlyOwner {

77:         IFrxEthEthPool(FRX_ETH_CRV_POOL_ADDRESS).exchange(
78:             1,
79:             0,
80:             frxEthBalance,
81:             minOut
82:         );
```

- `deposit()` could fail because `submitAndDeposit()` -> `_submit()` can be paused.

```solidity
File: contracts/SafEth/derivatives/SfrxEth.sol
094:     function deposit() external payable onlyOwner returns 

101:         frxETHMinterContract.submitAndDeposit{value: msg.value}(address(this));
```

Below is the frxETHMinter contract:
```solidity
// https://etherscan.io/address/0xbAFA44EFE7901E04E39Dad13167D089C559c1138
// frxETHMinter.sol: 70-101
    function submitAndDeposit(address recipient) external payable returns (uint256 shares) {

        _submit(address(this));

    }

    function _submit(address recipient) internal nonReentrant {

        require(!submitPaused, "Submit is paused");

    }

```

If `submitPaused` is turned on, this deposit function will revert.

#### Reth.sol

- `rethAddress()` and `getAddress()`

`rethAddress()` is called in multiple places: 
```solidity
File: contracts/SafEth/derivatives/Reth.sol
66:     function rethAddress() private view returns (address) {
67:         return
68:             RocketStorageInterface(ROCKET_STORAGE_ADDRESS).getAddress(
69:                 keccak256(
70:                     abi.encodePacked("contract.address", "rocketTokenRETH")
71:                 )
72:             );
73:     }
```

But it could return wrong address or `addr(0)`, since the referred `getAddress()` could return unexpected result. `addressStorage[_key]` can be reset or deleted. Then the whole function call will revert.

Below is the RocketStorage.sol:
```solidity
// https://etherscan.io/address/0x1d8f8f00cfa6758d7bE78336684788Fb0ee0Fa46
// 179-181
    function getAddress(bytes32 _key) override external view returns (address r) {
        return addressStorage[_key];
    }

// 215-217
    function setAddress(bytes32 _key, address _value) onlyLatestRocketNetworkContract override external {
        addressStorage[_key] = _value;
    }

// 251-253
    function deleteAddress(bytes32 _key) onlyLatestRocketNetworkContract override external {
        delete addressStorage[_key];
    }    
```

`rethAddress()` is referred in `withdraw()/deposit()/ethPerDerivative()/balance()`:
```solidity
File: contracts/SafEth/derivatives/Reth.sol
107:     function withdraw(uint256 amount) external onlyOwner {
108:         RocketTokenRETHInterface(rethAddress()).burn(amount);

156:     function deposit() external payable onlyOwner returns (uint256) {

176:             IWETH(W_ETH_ADDRESS).deposit{value: msg.value}();
177:             uint256 amountSwapped = swapExactInputSingleHop(
178:                 W_ETH_ADDRESS,
179:                 rethAddress(),
180:                 500,
181:                 msg.value,
182:                 minOut
183:             );

211:     function ethPerDerivative(uint256 _amount) public view returns (uint256) {
212:         if (poolCanDeposit(_amount))
213:             return
214:                 RocketTokenRETHInterface(rethAddress()).getEthValue(10 ** 18);
215:         else return (poolPrice() * 10 ** 18) / (10 ** 18);
216:     }

221:     function balance() public view returns (uint256) {
222:         return IERC20(rethAddress()).balanceOf(address(this));
223:     }
```

`getAddress()` will also influence `poolCanDeposit()`, which could revert `deposit()` and the view function `ethPerDerivative()`:
```solidity
120:     function poolCanDeposit(uint256 _amount) private view returns (bool) {
121:         address rocketDepositPoolAddress = RocketStorageInterface(
122:             ROCKET_STORAGE_ADDRESS
123:         ).getAddress(
124:                 keccak256(
125:                     abi.encodePacked("contract.address", "rocketDepositPool")
126:                 )
127:             );

156:     function deposit() external payable onlyOwner returns (uint256) {

170:         if (!poolCanDeposit(msg.value)) {

211:     function ethPerDerivative(uint256 _amount) public view returns (uint256) {
212:         if (poolCanDeposit(_amount))
```



- `burn()` 

There is no guarantee that the function `burn()` will succeed.
```solidity
File: contracts/SafEth/derivatives/Reth.sol
107:     function withdraw(uint256 amount) external onlyOwner {
108:         RocketTokenRETHInterface(rethAddress()).burn(amount);
```

Because in Reth contract code below, the execution may fail in several cases:
- `burn()` -> `getTotalCollateral()` -> `getContractAddress()` -> `getAddress()` might fail due to the same reason above
- the `require(ethBalance >= ethAmount)` could fail due to low balance
- `burn()` -> `withdrawDepositCollateral()` -> `getContractAddress()`, `withdrawExcessBalance()` both could fail for same reason as above
- `msg.sender.transfer()` could fail due to not enough gas (2300 limit)
```solidity
// https://etherscan.io/address/0xae78736Cd615f374D3085123A210448E74Fc6393
// RocketTokenRETH.sol: 131-146
    // Burn rETH for ETH
    function burn(uint256 _rethAmount) override external {
        // Check rETH amount
        require(_rethAmount > 0, "Invalid token burn amount");
        require(balanceOf(msg.sender) >= _rethAmount, "Insufficient rETH balance");
        // Get ETH amount
        uint256 ethAmount = getEthValue(_rethAmount);
        // Get & check ETH balance
        uint256 ethBalance = getTotalCollateral();
        require(ethBalance >= ethAmount, "Insufficient ETH balance for exchange");
        // Update balance & supply
        _burn(msg.sender, _rethAmount);
        // Withdraw ETH from deposit pool if required
        withdrawDepositCollateral(ethAmount);
        // Transfer ETH to sender
        msg.sender.transfer(ethAmount);
        // Emit tokens burned event
        emit TokensBurned(msg.sender, _rethAmount, ethAmount, block.timestamp);
    }

// RocketTokenRETH.sol: 98-101
    function getTotalCollateral() override public view returns (uint256) {
        RocketDepositPoolInterface rocketDepositPool = RocketDepositPoolInterface(getContractAddress("rocketDepositPool"));
        return rocketDepositPool.getExcessBalance().add(address(this).balance);
    }

// RocketBase.sol: 112-119
    function getContractAddress(string memory _contractName) internal view returns (address) {
        // Get the current contract address
        address contractAddress = getAddress(keccak256(abi.encodePacked("contract.address", _contractName)));
        // Check it
        require(contractAddress != address(0x0), "Contract not found");
        // Return
        return contractAddress;
    }

// RocketTokenRETH.sol: 152-159
    function withdrawDepositCollateral(uint256 _ethRequired) private {
        // Check rETH contract balance
        uint256 ethBalance = address(this).balance;
        if (ethBalance >= _ethRequired) { return; }
        // Withdraw
        RocketDepositPoolInterface rocketDepositPool = RocketDepositPoolInterface(getContractAddress("rocketDepositPool"));
        rocketDepositPool.withdrawExcessBalance(_ethRequired.sub(ethBalance));
    }
```


### WstEth.sol

StEthEthPool function `exchange()` could fail due to `minOut` requirement.
```solidity
File: contracts/SafEth/derivatives/WstEth.sol
56:     function withdraw(uint256 _amount) external onlyOwner {

60:         uint256 minOut = (stEthBal * (10 ** 18 - maxSlippage)) / 10 ** 18;
```

As long as any one of the above code failed, the whole `stake()/unstake()/rebalanceToWeights()` will revert, and users' fund would be locked until the external dependency is resolved, the contract will lose the core functionality.


## Tools Used
Manual analysis.

## Recommended Mitigation Steps

Use `try/catch` to skip the failed function call, then the contract will be more robust to unexpected situations. In case of `deposit()`, redistribute the fund into the other derivatives according to the weights might be an option, since re-balance will be done regularly. For `withdraw()`, maybe temporarily record the missed amount, and give the user opportunity to retrieve later.



