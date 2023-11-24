# Sparkn  - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. The `organizer` can only distribute one token which is incorrect because the `Distributor` contract can have multiple tokens, so some tokens may be stuck in the contract](#H-01)
    - ### [H-02. The same signature can be used in different `distribution` implementation causing that the caller who owns the signature, can distribute on unauthorized implementations](#H-02)

- ## Low Risk Findings
    - ### [L-01. Organizers are not incentivized to deploy and distribute to winners causing that winners may not to be rewarded for a long time and force the protocol owner to manage the distribution](#L-01)
    - ### [L-02. The signature does not have an expiration time](#L-02)
    - ### [L-03. There is no feature that helps to cancel a given signature](#L-03)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: CodeFox Inc.

### Dates: Aug 20th, 2023 - Aug 28th, 2023

[See more contest details here](https://www.codehawks.com/contests/cllcnja1h0001lc08z7w0orxx)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 0
   - Low: 3


# High Risk Findings

## <a id='H-01'></a>H-01. The `organizer` can only distribute one token which is incorrect because the `Distributor` contract can have multiple tokens, so some tokens may be stuck in the contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Distributor.sol#L92

https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L127

https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L152

## Summary

The `organizer` can distribute only one token using the `deployProxyAndDistribute()` function, that is incorrect because the sponsor can deposit to the Distributor contract multiple whitelisted tokens causing that some tokens may be stuck in the contract.

## Vulnerability Details

The `organizer` can use the [deployProxyAndDistribute()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L127) to deploy the [Distributor contract](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Distributor.sol#L92) and then [execute the distribution](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L136) to winners. Additionally, the sponsor can deposit to the `Distributor contract` the tokens that [are whitelisted](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L85), so those tokens can be distributed to the winners.

The problem is that the `organizer` only have an option to distribute one token because the [Distributor::distribute()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Distributor.sol#L92) only accept one token at a time and if was not enough, the `organizer` can't call again the `deployProxyAndDistribute()` function with another toker because it will be reverted.

I created a test where the `organizer` calls `deployProxyAndDistribute()` for the token `jpycv2Address` and the distribution to winners is correct but the `organizer` wants to distribute the token `jpycv1Address` and the `deployProxyAndDistribute()` will be reverted. Test steps:

1. Sponsor deposits `jpycv1Address` and `jpycv2Address`.
2. Organizer calls `deployProxyAndDistribute()` in order to distributes the token `jpycv2Address`.
3. Organizer calls `deployProxyAndDistribute()` in order to distributes the token `jpycv1Address` but the transaction will be reverted.
4. The amount of `jpycv1Address` will be stuck in the `Distributor` contract.


```solidity
// test/integration/ProxyFactoryTest.t.sol:ProxyFactoryTest
// $ forge test --match-test "testDistributeIsPossibleJustForOneToken" -vvv
//
    function testDistributeIsPossibleJustForOneToken() public {
        // The organizer can distribute just for one token which is incorrect because
        // the sponsor can deposit to the Distributor contract all the whitelisted tokens.
        //
        // before
        assertEq(MockERC20(jpycv2Address).balanceOf(user1), 0 ether);
        assertEq(MockERC20(jpycv2Address).balanceOf(stadiumAddress), 0 ether);
        vm.startPrank(factoryAdmin);
        bytes32 randomId = keccak256(abi.encode("Jason", "001"));
        proxyFactory.setContest(organizer, randomId, block.timestamp + 8 days, address(distributor));
        vm.stopPrank();
        bytes32 salt = keccak256(abi.encode(organizer, randomId, address(distributor)));
        address proxyAddress = proxyFactory.getProxyAddress(salt, address(distributor));
        //
        // 1. Sponsor deposits jpycv1Address and jpycv2Address
        //
        vm.startPrank(sponsor);
        MockERC20(jpycv1Address).transfer(proxyAddress, 100 ether);
        MockERC20(jpycv2Address).transfer(proxyAddress, 10000 ether);
        vm.stopPrank();
        assertEq(MockERC20(jpycv1Address).balanceOf(proxyAddress), 100 ether);
        assertEq(MockERC20(jpycv2Address).balanceOf(proxyAddress), 10000 ether);
        bytes32 randomId_ = keccak256(abi.encode("Jason", "001"));
        address[] memory winners = new address[](1);
        winners[0] = user1;
        uint256[] memory percentages_ = new uint256[](1);
        percentages_[0] = 9500;
        vm.warp(9 days); // 9 days later
        vm.startPrank(organizer);
        //
        // 2. Organizer calls deployProxyAndDistribute() in order to distributes the token jpycv2Address
        //
        bytes memory data = abi.encodeWithSelector(Distributor.distribute.selector, jpycv2Address, winners, percentages_, "");
        proxyFactory.deployProxyAndDistribute(randomId_, address(distributor), data);
        //
        // 3. Organizer calls deployProxyAndDistribute() in order to distributes the token jpycv1Address but
        // the transaction will be reverted.
        //
        data = abi.encodeWithSelector(Distributor.distribute.selector, jpycv1Address, winners, percentages_, "");
        vm.expectRevert();
        proxyFactory.deployProxyAndDistribute(randomId_, address(distributor), data);
        vm.stopPrank();
        // after
        assertEq(MockERC20(jpycv2Address).balanceOf(user1), 9500 ether);
        assertEq(MockERC20(jpycv2Address).balanceOf(stadiumAddress), 500 ether);
        //
        // 4. The amount of jpycv1Address will be stuck in the Distributor contract.
        //
        assertEq(MockERC20(jpycv1Address).balanceOf(proxyAddress), 100 ether);
        assertEq(MockERC20(jpycv2Address).balanceOf(proxyAddress), 0);
    }
```

The `owner` can call the [distributeByOwner()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L205C14-L205C31) function but the purpose of the function is not to assign winners, [the purpose](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L196C6-L197) is to rescue funds if token is stuck after the deployment and contest is over for a while.

```
    * @notice Owner can rescue funds if token is stuck after the deployment and contest is over for a while
    * @dev only owner can call this function and it is supposed not to be called often
```

## Impact

The `organizer` can deploy and distribute only one token, that is a problem because sponsors can deposit to the `Distributor` contract multiple tokens (the whitelisted tokens) causing that those tokens that the `organizer` did not distribute to get trapped in the contract.

## Tools used

Manual review

## Recommendations

Add the feature that helps the `organizer` to distribute all the whitelisted tokens.
## <a id='H-02'></a>H-02. The same signature can be used in different `distribution` implementation causing that the caller who owns the signature, can distribute on unauthorized implementations            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L159

## Summary

The same signature can be used in different `distribute` implementations causing that the caller who owns the signature, to distribute on unauthorized implementations.

## Vulnerability Details

The [ProxyFactory::setContest()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L105) function helps to configure a `closeTime` to specific `organizer`, `contestId` and `implementation`.

```solidity
File: ProxyFactory.sol
105:     function setContest(address organizer, bytes32 contestId, uint256 closeTime, address implementation)
106:         public
107:         onlyOwner
...
...
113:         bytes32 salt = _calculateSalt(organizer, contestId, implementation);
114:         if (saltToCloseTime[salt] != 0) revert ProxyFactory__ContestIsAlreadyRegistered();
115:         saltToCloseTime[salt] = closeTime;
```

The caller who owns the signature, can distributes to winners using the [deployProxyAndDistributeBySignature()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L152) function. The problem is that the hash in the code line ([#159](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L159)) does not consider the `implementation` parameter. 

```solidity
File: ProxyFactory.sol
152:     function deployProxyAndDistributeBySignature(
153:         address organizer,
154:         bytes32 contestId,
155:         address implementation,
156:         bytes calldata signature,
157:         bytes calldata data
158:     ) public returns (address) {
159:         bytes32 digest = _hashTypedDataV4(keccak256(abi.encode(contestId, data)));
160:         if (ECDSA.recover(digest, signature) != organizer) revert ProxyFactory__InvalidSignature();
161:         bytes32 salt = _calculateSalt(organizer, contestId, implementation);
162:         if (saltToCloseTime[salt] == 0) revert ProxyFactory__ContestIsNotRegistered();
163:         if (saltToCloseTime[salt] > block.timestamp) revert ProxyFactory__ContestIsNotClosed();
164:         address proxy = _deployProxy(organizer, contestId, implementation);
165:         _distribute(proxy, data);
166:         return proxy;
167:     }
```

For some reason, there could be a different `distribution` implementation to the same `contestId`. Then the caller who owns the signature can distribute even if the organizer does not authorize a signature to the new implementation.

I created a test where the caller who owns a signature can distribute to new `distribute implementation` using the same signature. Test steps:

1. Owner setContest using the implementation `address(distributor)`
2. Organizer creates a signature.
3. Caller distributes prizes using the signature.
4. For some reason there is a new distributor implementation. The Owner set the new distributor for the same `contestId`.
5. The caller can distribute prizes using the same signature created in the step 2 in different distributor implementation.

```solidity
// test/integration/ProxyFactoryTest.t.sol:ProxyFactoryTest
// $ forge test --match-test "testSignatureCanBeUsedToNewImplementation" -vvv
//
    function testSignatureCanBeUsedToNewImplementation() public {
        address organizer = TEST_SIGNER;
        bytes32 contestId = keccak256(abi.encode("Jason", "001"));
        //
        // 1. Owner setContest using address(distributor)
        vm.startPrank(factoryAdmin);
        proxyFactory.setContest(organizer, contestId, block.timestamp + 8 days, address(distributor));
        vm.stopPrank();
        bytes32 salt = keccak256(abi.encode(organizer, contestId, address(distributor)));
        address proxyAddress = proxyFactory.getProxyAddress(salt, address(distributor));
        vm.startPrank(sponsor);
        MockERC20(jpycv2Address).transfer(proxyAddress, 10000 ether);
        vm.stopPrank();
        assertEq(MockERC20(jpycv2Address).balanceOf(proxyAddress), 10000 ether);
        // before
        assertEq(MockERC20(jpycv2Address).balanceOf(user1), 0 ether);
        assertEq(MockERC20(jpycv2Address).balanceOf(stadiumAddress), 0 ether);
        //
        // 2. Organizer creates a signature
        (bytes32 digest, bytes memory sendingData, bytes memory signature) = createSignatureByASigner(TEST_SIGNER_KEY);
        assertEq(ECDSA.recover(digest, signature), TEST_SIGNER);
        vm.warp(8.01 days);
        //
        // 3. Caller distributes prizes using the signature
        proxyFactory.deployProxyAndDistributeBySignature(
            TEST_SIGNER, contestId, address(distributor), signature, sendingData
        );
        // after
        assertEq(MockERC20(jpycv2Address).balanceOf(user1), 9500 ether);
        assertEq(MockERC20(jpycv2Address).balanceOf(stadiumAddress), 500 ether);
        //
        // 4. For some reason there is a new distributor implementation.
        // The Owner set the new distributor for the same contestId
        Distributor new_distributor = new Distributor(address(proxyFactory), stadiumAddress);
        vm.startPrank(factoryAdmin);
        proxyFactory.setContest(organizer, contestId, block.timestamp + 8 days, address(new_distributor));
        vm.stopPrank();
        bytes32 newDistributorSalt = keccak256(abi.encode(organizer, contestId, address(new_distributor)));
        address proxyNewDistributorAddress = proxyFactory.getProxyAddress(newDistributorSalt, address(new_distributor));
        vm.startPrank(sponsor);
        MockERC20(jpycv2Address).transfer(proxyNewDistributorAddress, 10000 ether);
        vm.stopPrank();
        //
        // 5. The caller can distribute prizes using the same signature in different distributor implementation
        vm.warp(20 days);
        proxyFactory.deployProxyAndDistributeBySignature(
            TEST_SIGNER, contestId, address(new_distributor), signature, sendingData
        );
    }
```

## Impact

The caller who owns the signature, can distribute the prizes for a new distribution implementation using the same signature which was created for an old implementation. 
The `organizer` must create a new signature if there is a new implementation for the same `contestId`. The authorized signature is for one distribution implementation not for the future distribution implementations.

## Tools used

Manual review

## Recommendations

Include the `distribution implementation` in the [signature hash](https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/ProxyFactory.sol#L159).

```diff
    function deployProxyAndDistributeBySignature(
        address organizer,
        bytes32 contestId,
        address implementation,
        bytes calldata signature,
        bytes calldata data
    ) public returns (address) {
--      bytes32 digest = _hashTypedDataV4(keccak256(abi.encode(contestId, data)));
++      bytes32 digest = _hashTypedDataV4(keccak256(abi.encode(contestId, implementation, data)));
```
		


# Low Risk Findings

## <a id='L-01'></a>L-01. Organizers are not incentivized to deploy and distribute to winners causing that winners may not to be rewarded for a long time and force the protocol owner to manage the distribution            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L127

https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L152

## Summary

The organizer can deploy and distribute to winners at any time without restriction about the contest expiration time `EXPIRATION_TIME` causing that the winners to be unable to receive their rewards for a long time.

## Vulnerability Details

The organizer can execute the [deployProxyAndDistribute()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L127C14-L127C38) function to deploy the `distribute` contract and execute the [distribution](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L136) to winners. The only restriction is that the current time should be [greater than contest close time](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L134) (code line 134).

```solidity
File: ProxyFactory.sol
127:     function deployProxyAndDistribute(bytes32 contestId, address implementation, bytes calldata data)
128:         public
129:         returns (address)
130:     {
131:         bytes32 salt = _calculateSalt(msg.sender, contestId, implementation);
132:         if (saltToCloseTime[salt] == 0) revert ProxyFactory__ContestIsNotRegistered();
133:         // can set close time to current time and end it immediately if organizer wish
134:         if (saltToCloseTime[salt] > block.timestamp) revert ProxyFactory__ContestIsNotClosed();
...
...
```

In the other hand, the `owner` can execute [deployProxyAndDistributeByOwner()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L179C14-L179C45) function after the [contest expiration time](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L187) (code line [187](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L187)).

```solidity
File: ProxyFactory.sol
179:     function deployProxyAndDistributeByOwner(
180:         address organizer,
181:         bytes32 contestId,
182:         address implementation,
183:         bytes calldata data
184:     ) public onlyOwner returns (address) {
185:         bytes32 salt = _calculateSalt(organizer, contestId, implementation);
186:         if (saltToCloseTime[salt] == 0) revert ProxyFactory__ContestIsNotRegistered();
187:         if (saltToCloseTime[salt] + EXPIRATION_TIME > block.timestamp) revert ProxyFactory__ContestIsNotExpired();
...
...
```

The problem is that the `organizer` can execute the [deployProxyAndDistribute()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L127C14-L127C38) function after the [contest close time](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L134) without restriction of time. The `organizer` can wait indefinitely causing the winners not to be rewarded for a long time and force the owner to execute the distribution manually via `deployProxyAndDistributeByOwner()`.

Additionally, the `organizers` are not incentivized to deploy and distribute to winners.

## Impact

The malicious organizer can wait indefinitely until the `owner` calls [deployProxyAndDistributeByOwner()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L179C14-L179C45). The bad/malicious behaivour of the `organizer` can cause the winners to be unable receive rewards for a long time AND force the `owner` to execute manually `deployProxyAndDistributeByOwner()`. That affects the protocol because rewards are not assigned in time AND the protocol owner needs to manage manually the deploy and distribution in order to not affect the protocol's reputation and winners.

Additionally the `organizers` are not incentivized to deploy and distribute to winners causing to the protocol owner to execute manually the `deployProxyAndDistributeByOwner()`.

## Tools used

Manual review

## Recommendations

Add a validation that the `organizer` distribution must be between the `saltToCloseTime` and the `EXPIRATION_TIME`. Same in [deployProxyAndDistributeBySignature()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L152C14-L152C49)

```diff
    function deployProxyAndDistribute(bytes32 contestId, address implementation, bytes calldata data)
        public
        returns (address)
    {
        bytes32 salt = _calculateSalt(msg.sender, contestId, implementation);
        if (saltToCloseTime[salt] == 0) revert ProxyFactory__ContestIsNotRegistered();
        // can set close time to current time and end it immediately if organizer wish
        if (saltToCloseTime[salt] > block.timestamp) revert ProxyFactory__ContestIsNotClosed();
++      if (saltToCloseTime[salt] + EXPIRATION_TIME < block.timestamp) revert();
        address proxy = _deployProxy(msg.sender, contestId, implementation);
        _distribute(proxy, data);
        return proxy;
    }
```

Additionally, there should be a penalization to the `organizer` or an incentive to deploy and distribute in time to winners.
## <a id='L-02'></a>L-02. The signature does not have an expiration time            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L159

## Summary

The signature does not use a expiration time, allowing to the caller who has the signature to call [deployProxyAndDistributeBySignature()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L152) at any time. 

## Vulnerability Details

The [deployProxyAndDistributeBySignature()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L152)  function helps to distribute prizes to winners by the caller who has the correct signature.

The problem is that the [signature does not have an expiration time](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L159) causing that the caller, who has the signature, is able to call the `deployProxyAndDistributeBySignature()` at any time. That could be a problem because if for some reason no one call the [_distribute()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L249) function and the owner distributes the prizes using the [deployProxyAndDistributeByOwner()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L179) function after the [expiration time](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L187), the signature owner can frontrun the owner `deployProxyAndDistributeByOwner()` execution. Please see the next scenario:

1. The `organizer` creates a signature to the `winner1` and `winner2`.
2. No one calls [deployProxyAndDistribute()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L127) or [deployProxyAndDistributeBySignature()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L152) functions.
3. The time goes and the owner can call the [deployProxyAndDistributeByOwner()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L179C14-L179C45) function because the [expiration time](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L187) is left behind. The owner wants to distribute prizes to `winner1`, `winner2` and `winner3`. He adds the `winner3`.
4. The caller who has the signature execute the function before the owner execution (frontrun). Now the distribution is made by the caller who owns the signature and it distributes to `winner1` and `winner2`.

## Impact

The signature does not have an expiration time. The signature can be used for the end of life.

## Tools used

Manual review

## Recommendations

Add expiration time to the signature.
## <a id='L-03'></a>L-03. There is no feature that helps to cancel a given signature            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L152

## Summary

The [ProxyFactory::deployProxyAndDistributeBySignature()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L152) function helps to deploy proxy contract and distribute prize on behalf of organizer. The problem is that there is no implementation that helps to cancel the given signature.

## Vulnerability Details

The [ProxyFactory::deployProxyAndDistributeBySignature()](https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/ProxyFactory.sol#L152) function helps to deploy proxy contract and distribute prize on behalf of organizer.

The hash contains the next parameters `keccak256(abi.encode(contestId, data))` (code line 159) and the `data` parameter is the distribute selector and the function parameters `data = abi.encodeWithSelector(Distributor.distribute.selector, jpycv2Address, winners, percentages_, "");`.

```solidity
File: ProxyFactory.sol
152:     function deployProxyAndDistributeBySignature(
153:         address organizer,
154:         bytes32 contestId,
155:         address implementation,
156:         bytes calldata signature,
157:         bytes calldata data
158:     ) public returns (address) {
159:         bytes32 digest = _hashTypedDataV4(keccak256(abi.encode(contestId, data)));
160:         if (ECDSA.recover(digest, signature) != organizer) revert ProxyFactory__InvalidSignature();
161:         bytes32 salt = _calculateSalt(organizer, contestId, implementation);
162:         if (saltToCloseTime[salt] == 0) revert ProxyFactory__ContestIsNotRegistered();
163:         if (saltToCloseTime[salt] > block.timestamp) revert ProxyFactory__ContestIsNotClosed();
164:         address proxy = _deployProxy(organizer, contestId, implementation);
165:         _distribute(proxy, data);
166:         return proxy;
167:     }
```

The problem is that the sighature can not be cancelled by the `organizer` therfore the caller can distribute the prizes wrongly. Please see the next scenario:
1. `Organizer` creates the signature using the `winner1 percentage 90%` and `winner2 percentage 5%`.
2. The caller waits until the `close time` is reached. Meanwhile for some reason, the `organizer` wants to change the winners percentages but since there is no any feature that helps to cancel the signatures, the caller who owns the signature, can still call with the wrong winners.
3. `Caller` distributes to undesired winners.

## Impact

The signature can not be cancelled by the `organizer`. There could be situations where the `organizer` may want to change the winners/percentages before the caller who owns the signature calls `distribute` function.

## Tools used

Manual review

## Recommendations

Implements a feature that helps to cancel a given signature by the `organizer`.


