# Original link
https://github.com/sherlock-audit/2023-06-unstoppable-judging/issues/103
0xbepresent

medium

# The `Vault._update_debt()` function should be executed before admin sets new interest rate via `Vault.set_variable_interest_parameters()`

## Summary

The [Vault._update_debt()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1050) should be executed before the interests rate are modified by the [Vault.set_variable_interest_parameters()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1581C5-L1581C37) function.

## Vulnerability Detail

The [Vault._update_debt()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1050) function helps to accrue interest since the last update. The execution path:
1. It calls the function [Vault._debt_interest_since_last_update(_debt_token)](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1063) in order to calculate the token debt interest.
2. The `Vault._debt_interest_since_last_update()` function uses the [Vault._current_interest_per_second()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1072C16-L1072C44) function in order to get the interest per second.
3. The [Vault._current_interest_per_second()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1173) calls [Vault._interest_rate_by_utilization()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1175C35-L1175C64) in order to get the `interest rate`.
4. The [Vault._interest_rate_by_utilization()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1204C5-L1204C34) function calls the [Vault._dynamic_interest_rate_low_utilization()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1218) or [Vault._dynamic_interest_rate_high_utilization](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1220C21-L1220C60) depending on the `switch utilization`.
5. The [Vault._dynamic_interest_rate_low_utilization()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1225C5-L1225C43) is executed and it calls the [Vault._min_interest_rate()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1240).
6. Then, the [Vault._min_interest_rate()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1296) function gets the value from the [self.interest_configuration[_address][0]](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1300C17-L1300C39)

So in the step 6, it gets the interest from the `interest_configuration`, then the interests are used in order to calculate the token debt. The `Vault._update_debt()` accrue interest since the last update, the function should be executed before the admin change the interest configuration via [Vault.set_variable_interest_parameters()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1581C5-L1581C37) function. The reason is that the time that has already passed must be taken with the previous interest before the admin changes to the new interests rate.

## Impact

The protocol should accrue interest since the last update before change the interest rate configuration. If the admin change the interest rate, the new interest rate should be applied to the future time not to the time that has already passed. The time that has already passed should use the previous interest.

It is unfair for the users because users expects interests changes to be applied after they were changed not to the past time the `_update_debt` has not been executed.

## Code Snippet

- The [Vault._update_debt()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1050) function.
- The [Vault._debt_interest_since_last_update](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1069) function.
- The [Vault._current_interest_per_second()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1173) function.
- The [Vault._interest_rate_by_utilization()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1204C5-L1204C34) function.
- The [Vault._dynamic_interest_rate_low_utilization()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1225C5-L1225C43) function.
- The [Vault._dynamic_interest_rate_high_utilization()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1253) function.
- The [Vault._interest_configuration()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1282C5-L1282C28) function.
- The [Vault.set_variable_interest_parameters()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L1581C5-L1581C37) function.


## Tool used

Manual review

## Recommendation

Execute `self._update_debt(address)` before the interests are modified:

```diff
@external
def set_variable_interest_parameters(
    _address: address,
    _min_interest_rate: uint256,
    _mid_interest_rate: uint256,
    _max_interest_rate: uint256,
    _rate_switch_utilization: uint256,
):
    assert msg.sender == self.admin, "unauthorized"
++  self._update_debt(address)
    self.interest_configuration[_address] = [
        _min_interest_rate,
        _mid_interest_rate,
        _max_interest_rate,
        _rate_switch_utilization,
    ]
```