# Original link
https://github.com/code-423n4/2022-11-debtdao-findings/issues/201
# Lines of code

https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/modules/escrow/Escrow.sol#L86


# Vulnerability details

## Impact

The ```Escrow.sol``` contract can receive collateral to the position. The ```addCollateral()``` function receive the amount and the token. If the user add collateral with an ERC20 as a token AND send accidentally/intentionally native ETH value, the eth value will be lockout because the payable function does not control the native ```msg.value```.


## Proof of Concept

I wrote a test for the ```addCollateral``` function:

1. As a borrower addCollateral for 1 suportedToken1 AND ACCIDENTALLY/INTENTIONALLY send 10 Native ETH
2. As a borrower get the collateral value, it only counts the 1 supportedToken1
3. As a borrower the 10 native eth I sent are not accounted in the collateral value

**Test:**

```solidity
// Escrow.t.sol
// $ forge test -m "test_0xbepresent_can_add_token_and_ETH" -vvv

function test_0xbepresent_can_add_token_and_ETH() public {
    //
    // Native ETH could be trapped in the escrow contract.
    // 1. As a borrower addCollateral for 1 suportedToken1 AND ACCIDENTALLY/INTENTIONALLY send 10 Native ETH
    // 2. As a borrower get the collateral value, it only counts the 1 supportedToken1.

    //
    // 1. As a borrower addCollateral for 1 suportedToken1 AND ACCIDENTALLY/INTENTIONALLY send 10 Native ETH
    //
    uint escrowSupportedToken1BalanceBeforeDeposit = supportedToken1.balanceOf(address(escrow));
    uint escrowBalanceBeforeDeposit = address(escrow).balance;
    uint supportedToken1AmountToDeposit = 1 ether;
    uint nativeETHAmountToDeposit = 10 ether;
    escrow.addCollateral{value: nativeETHAmountToDeposit}(
        supportedToken1AmountToDeposit,
        address(supportedToken1));
    // Escrow has more supportedToken1
    assertEq(
        escrowSupportedToken1BalanceBeforeDeposit + supportedToken1AmountToDeposit,
        supportedToken1.balanceOf(address(escrow)));
    // Escrow balance has 10 more native ether
    assertEq(
        escrowBalanceBeforeDeposit + nativeETHAmountToDeposit,
        address(escrow).balance
    );
    //
    // 2. As a borrower get the collateral value, it only counts the 1 supportedToken1.
    //
    assertEq(escrow.getCollateralValue(),  (1000 * 1e8) * (supportedToken1AmountToDeposit / 1 ether));
}
```

## Tools used
Foundry/VisualStudio

## Recommended Mitigation Steps

If the user send to ```addCollateral``` a non Denominations.ETH token, the ```msg.value``` should be zero in order to avoid eth value lockout.