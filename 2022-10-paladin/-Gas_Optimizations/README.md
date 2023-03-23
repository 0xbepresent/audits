# Original link
https://github.com/code-423n4/2022-10-paladin-findings/issues/104
1 - Use immutable variables can save gas
==

Variables that are assigned only in the constructor can be immutable

**Instances of this issue:**

- [contracts/WardenPledge.sol#L60](https://github.com/code-423n4/2022-10-paladin/blob/d6d0c0e57ad80f15e9691086c9c7270d4ccfe0e6/contracts/WardenPledge.sol#L60)
- [contracts/WardenPledge.sol#L62](https://github.com/code-423n4/2022-10-paladin/blob/d6d0c0e57ad80f15e9691086c9c7270d4ccfe0e6/contracts/WardenPledge.sol#L62)

2 - Use constant variable can save gas
==

The variable can be constant in order to save gas

**Instances of this issue:**
- [contracts/WardenPledge.sol#L79](https://github.com/code-423n4/2022-10-paladin/blob/d6d0c0e57ad80f15e9691086c9c7270d4ccfe0e6/contracts/WardenPledge.sol#L79)