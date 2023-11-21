
- [Medium](Medium-1789944694/README.md) - 0xbepresent - The `Vault._update_debt()` function should be executed before admin sets new interest rate via `Vault.set_variable_interest_parameters()`

- [Medium](Medium-1789946964/README.md) - 0xbepresent - The `Vault._to_usd_oracle_price()` function uses the same `ORACLE_FRESHNESS_THRESHOLD` for all token prices feeds which is incorrect

- [High](High-1789982999/README.md) - 0xbepresent - The `Vault.reduce_position()` function does not increment the account's margin `Vault.margin[account][debt_token]`
