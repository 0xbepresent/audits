# Original link
https://github.com/code-423n4/2023-01-astaria-findings/issues/25
# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/Vault.sol#L65


# Vulnerability details

## Impact
It is possible to make a deposit even when `_loadVISlot().isShutdown` is true. This check is done in `whenNotPaused` modifier and is already done for Public Vault but is missing for Private Vault, allowing unexpected deposits

## Proof of Concept
1. Observe the deposit function at https://github.com/code-423n4/2023-01-astaria/blob/main/src/Vault.sol#L59

```
function deposit(uint256 amount, address receiver)
    public
    virtual
    returns (uint256)
  {
    VIData storage s = _loadVISlot();
    require(s.allowList[msg.sender] && receiver == owner());
    ERC20(asset()).safeTransferFrom(msg.sender, address(this), amount);
    return amount;
  }
```

2. Observe there is no `whenNotPaused` modifier in deposit which confirm that shutdown has not been called yet

```
modifier whenNotPaused() {
    if (ROUTER().paused()) {
      revert InvalidRequest(InvalidRequestReason.PAUSED);
    }

    if (_loadVISlot().isShutdown) {
      revert InvalidRequest(InvalidRequestReason.SHUTDOWN);
    }
    _;
  }
```

## Recommended Mitigation Steps
Revise the deposit function as below:

```
function deposit(uint256 amount, address receiver)
    public
    virtual 
    whenNotPaused
    returns (uint256)
  {
    VIData storage s = _loadVISlot();
    require(s.allowList[msg.sender] && receiver == owner());
    ERC20(asset()).safeTransferFrom(msg.sender, address(this), amount);
    return amount;
  }
```