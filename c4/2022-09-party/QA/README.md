# Original link
https://github.com/code-423n4/2022-09-party-findings/issues/342

1 - Consider two-phase host transfer
==

The host calls ```abdicate``` in order to tranfers the host to the new address directly. There is a risk that the ownership is transferred to an invalid address, thus causing the contract to be without host.

Recommendation
--

Consider a two step process where the host nominates an pre-host account and the nominated pre-host account need to call an ```acceptHostOwnership()``` function for the transfer of host to fully succeed. This ensures the nominated EOA account is a valid and active account.

https://github.com/PartyDAO/party-contracts-c4/blob/d129d64779/contracts/party/PartyGovernance.sol#L458