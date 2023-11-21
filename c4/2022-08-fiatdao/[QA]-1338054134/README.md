# Original link
https://github.com/code-423n4/2022-08-fiatdao-findings/issues/45
1 - Not initializing the blocklist contract in VotingEscrow.sol constructor
==

The [blocklist contract address](https://github.com/code-423n4/2022-08-fiatdao/blob/main/contracts/VotingEscrow.sol#L53) is not initialized in the VotingEscrow.sol constructor.

The modifier ```checkBlocklist``` uses the blocklist contract address therefore the functions that use this modifier could lead to unexpected behaivors.

**Recommendation**

Initialize the blocklist contract address in the constructor so the creator/owner do not forget to setting up.

2 - Consider two-phase ownershop transfer
==

The owner calls ```transferOwnership``` in order to tranfers the ownership to the new address directly. There is a risk that the ownership is transferred to an invalid address, thus causing the contract to be without owner.

https://github.com/code-423n4/2022-08-fiatdao/blob/main/contracts/VotingEscrow.sol#L141

**Recommendation**

Consider a two step process where the owner nominates an account and the nominated account need to call an ```acceptOwnership()``` function for the transfer of owner to fully succeed. This ensures the nominated EOA account is a valid and active account.