# Original link
https://github.com/code-423n4/2022-08-nounsdao-findings/issues/111

1 - Consider two-phase vetoer transfer
==

The vetoer calls ```_setVetoer``` in order to tranfers the ownership to the new address directly. There is a risk that the ownership is transferred to an invalid address, thus causing the contract to be without vetoer.

Recommendation
--

Consider a two step process where the vetoer nominates an prevetoer account and the nominated prevetoer account need to call an ```acceptVetoOwnership()``` function for the transfer of vetoer to fully succeed. This ensures the nominated EOA account is a valid and active account


2 - The logic is not according to the comment
==

The require [here](https://github.com/code-423n4/2022-08-nounsdao/blob/main/contracts/governance/NounsDAOLogicV1.sol#L617) should validates ```pendingAdmin != address(0)``` instead  ```msg.sender != address(0)```

```
// Check caller is pendingAdmin and pendingAdmin ≠ address(0)
require(msg.sender == pendingAdmin && msg.sender != address(0), 'NounsDAO::_acceptAdmin: pending admin only');
```

Recommendation
--
Consider use ```pendingAdmin != address(0)``` in the ```require()``` logic.