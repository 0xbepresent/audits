# Original link
https://github.com/sherlock-audit/2023-06-unstoppable-judging/issues/129
0xbepresent

high

# The `Vault.reduce_position()` function does not increment the account's margin `Vault.margin[account][debt_token]`

## Summary

The `Vault.reduce_position()` decrease the positions values but it does not increment the account's margin `Vault.margin[account][debt_token]`.

## Vulnerability Detail

The [Vault.reduce_position()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L290) function helps to reduce the position by a certain amount. The function reduces the [position.margin_amount](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L326) needed, the problem is that this margin amount is lost because it is reduced to the `position.margin_amount` but it not added to the account's margin.

When the position is created via the [Vault.open_position()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L157), the account's margin is [decreased](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L178) and that [margin is added to the position](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L192). The `reduce_position()` should behave in the opposite way, the position margin should be reduced and that margin should be added to the account's margin.

I created a test where the position is reduced but the account's margin is not increased. Test steps:

1. Open the position using `300 * 10**6` as a position margin. Initial account's margin is `1000 * 10**6`
2. Assert the owner margin is `700 * 10**6` becuase `300 * 10**6` were used as a margin amount.
3. User reduces his position to 1/3.
4. The vault.margin() is still `700 * 10**6`
5. Close the position. Now the owner margin is `896666667`, there are `100 * 10**6` lost aprox because the initial margin was `1000 * 10**6`

```python
# tests/vault/test_reduce_position.py
# $ pytest -k "test_reduce_position_margin_account_is_not_updated"
def test_reduce_position_margin_account_is_not_updated(
    vault, owner, weth, usdc, mock_router, eth_usd_oracle
):
    """
    The reduce_position() function does not increment the account's margin
    1. Open the position. Initial account's margin 1000 * 10**6
    2. Assert the owner margin is 700 * 10**6. It uses 300 * 10**6 as a margin amount.
    3. User reduces his position to 1/3.
    4. The vault.margin() is still 700 * 10**6
    5. Close the position. There are 100 * 10**6 which are lost because the initial margin was 1000 * 10**6
    """
    eth_usd_oracle.set_answer(9300_00000000)
    assert vault.swap_router() == mock_router.address
    #
    # 1. Open the position. Initial account's margin 1000 * 10**6
    #
    assert vault.margin(owner, usdc) == 1000 * 10**6
    assert vault.margin(owner, weth) == 0
    uid, amount_bought = vault.open_position(
        owner,  # account
        weth,  # position_token
        1 * 10**18,  # min_position_amount_out
        usdc,  # debt_token
        9000 * 10**6,  # debt_amount
        300 * 10**6,  # margin_amount
    )
    position_before = vault.positions(uid)
    position_amount_before = position_before[6]
    assert position_amount_before == 1 * 10**18
    margin_amount_before = position_before[3]
    assert margin_amount_before == 300 * 10**6
    debt_amount_before = vault.debt(uid)
    assert debt_amount_before == 9000 * 10**6
    assert vault.debt(uid) > 0
    #
    # 2. Assert the owner margin is 700 * 10**6. It uses 300 * 10**6 as a margin amount.
    #
    assert vault.margin(owner, usdc) == 700 * 10**6
    assert vault.margin(owner, weth) == 0
    #
    # 3. User reduces his position to 1/3.
    #
    vault.reduce_position(uid, int(position_amount_before / 3), 3100 * 10**6)
    position_after = vault.positions(uid)
    debt_amount_after = vault.debt_shares_to_amount(usdc.address, position_after[4])
    assert debt_amount_after == pytest.approx(6000 * 10**6, 10000000)
    margin_amount_after = position_after[3]
    assert margin_amount_after == pytest.approx(200 * 10**6, 1000000)
    position_amount_after = position_after[6]
    assert position_amount_after == pytest.approx(int(1 * 10**18 / 3 * 2), 1000)
    #
    # 4. The vault.margin() is still 700 * 10**6
    #
    assert vault.margin(owner, usdc) == 700 * 10**6
    assert vault.margin(owner, weth) == 0
    #
    # 5. Close the position. There are 100 * 10**6 which are lost because the initial margin was 1000 * 10**6
    #
    vault.close_position(uid, 0)
    assert vault.margin(owner, usdc) == 896666667
    assert vault.margin(owner, weth) == 0
```

## Impact

The reduced margin is lost in the accounting because the `Vault.reduce_position()` does not increment the account's margin. So the reduced margin is lost forever.

## Code Snippet

- The [Vault.reduce_position()](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L290) function.


## Tool used

Manual review

## Recommendation

Increment the account's margin in the `Vault.reduce_position()` function:

```diff
def reduce_position(
    _position_uid: bytes32, _reduce_by_amount: uint256, _min_amount_out: uint256
) -> uint256:
...
...
    debt_amount: uint256 = self._debt(_position_uid)
    margin_debt_ratio: uint256 = position.margin_amount * PRECISION / debt_amount

    amount_out_received: uint256 = self._swap(
        position.position_token, position.debt_token, _reduce_by_amount, min_amount_out
    )

    # reduce margin and debt, keep leverage as before
    reduce_margin_by_amount: uint256 = (
        amount_out_received * margin_debt_ratio / PRECISION
    )
    reduce_debt_by_amount: uint256 = amount_out_received - reduce_margin_by_amount

    position.margin_amount -= reduce_margin_by_amount
++  self.margin[position.account][position.debt_token] += reduce_margin_by_amount
...
...
```

Duplicate of #143
