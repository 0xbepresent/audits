# Findings for 2022-12-gogopool 

- [[HIGH]]([HIGH]-Initial_'''ERC4626x.sol::deposit()'''_can_manipulate_share_prices/README.md) - Initial ```ERC4626x.sol::deposit()``` can manipulate share prices
- [[MEDIUM]]([MEDIUM]-MinipoolManager.sol::recordStakingEnd()_reverts_when_there_are_no_rewards_and_duration_is_zero_causing_funds_not_to_be_returned/README.md) - MinipoolManager.sol::recordStakingEnd() reverts when there are no rewards and duration is zero causing funds not to be returned
- [[MEDIUM]]([MEDIUM]-MinipoolManager.sol::recreateMinipool()_should_verify_if_the_multisig_is_still_enabled/README.md) - MinipoolManager.sol::recreateMinipool() should verify if the multisig is still enabled
- [[MEDIUM]]([MEDIUM]-The_Staking.sol::restakeGGP()_function_does_not_have_the_whenNotPaused_modifier./README.md) - The Staking.sol::restakeGGP() function does not have the whenNotPaused modifier.
- [[MEDIUM]]([MEDIUM]-NodeOp_funds_may_be_trapped_by_a_invalid_state_transition/README.md) - NodeOp funds may be trapped by a invalid state transition
- [[MEDIUM]]([MEDIUM]-NodeOp_can_get_rewards_even_if_there_was_an_error_in_registering_the_node_as_a_validator/README.md) - NodeOp can get rewards even if there was an error in registering the node as a validator
- [[MEDIUM]]([MEDIUM]-Users_may_not_be_able_to_redeem_their_shares_due_to_underflow/README.md) - Users may not be able to redeem their shares due to underflow
- [[MEDIUM]]([MEDIUM]-The_minipools_creation_could_be_compromised_if_is_not_possible_to_register_more_multisigs_and_all_of_them_are_disabled/README.md) - The minipools creation could be compromised if is not possible to register more multisigs and all of them are disabled
- [[MEDIUM]]([MEDIUM]-MinipoolManager.sol::recreateMinipool()_uses_a_wrong_reward_for_the_avaxLiquidStakerAmt_causing_to_re-stake_more_Avax_than_it_should_be/README.md) - MinipoolManager.sol::recreateMinipool() uses a wrong reward for the avaxLiquidStakerAmt causing to re-stake more Avax than it should be
- [[MEDIUM]]([MEDIUM]-NodeOp_with_error_in_registering_the_node_as_a_validator_can_withdraw_his_funds_and_still_have_an_active_minipool./README.md) - NodeOp with error in registering the node as a validator can withdraw his funds and still have an active minipool.
