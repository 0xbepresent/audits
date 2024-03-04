# Original link
https://github.com/code-423n4/2022-09-quickswap-findings/issues/283
1 - Internal functions only called once can be inlined to save gas
==

Not inlining costs more gas because of extra ```JUMP``` instructions and additional stack operations needed for function calls.

Instances of this issue:

```
contracts/AlgebraFactory.sol#L122    function computeAddress(address token0, address token1) internal view returns (address pool) {
```

2 - Using private rather than public for constants save gas
==

If needed, the values can be read from the verified contract source code.

Instances of this:

```
contracts/DataStorageOperator.sol#L15           uint256 constant UINT16_MODULO = 65536;
contracts/DataStorageOperator.sol#L16           uint128 constant MAX_VOLUME_PER_LIQUIDITY = 100000 << 64; // maximum meaningful ratio of volume to liquidity
core/contracts/libraries/DataStorage.sol#L12    uint32 public constant WINDOW = 1 days;
```

3 - More efficient struct packing
==

Pack non-256 bit type into single 256-bit storage slot may save gas.

Instances of this issue:

```
contracts/libraries/AdaptiveFee.sol#L10         struct Configuration {
contracts/AlgebraPool.sol#L41                   struct Position {
contracts/libraries/TickManager.sol#L17         struct Tick {
contracts/AlgebraPool.sol#L675                  struct SwapCalculationCache {
contracts/AlgebraPool.sol#L692                  struct PriceMovementCache {
```

4 - Multiple require statements instead of &&
==

Require statements including conditions with the && operator can be broken down in multiple require statements to save gas.

Instances of this issue:

```
contracts/AlgebraPool.sol#L953         require((communityFee0 <= Constants.MAX_COMMUNITY_FEE) && (communityFee1 <= Constants.MAX_COMMUNITY_FEE));
contracts/AlgebraPool.sol#L739         require(limitSqrtPrice < currentPrice && limitSqrtPrice > TickMath.MIN_SQRT_RATIO, 'SPL');
contracts/AlgebraPool.sol#L968         require(newLiquidityCooldown <= Constants.MAX_LIQUIDITY_COOLDOWN && liquidityCooldown != newLiquidityCooldown);
```

5 - Cache Array Length Outside of Loop
==

Caching the array length outside a loop saves reading it on each iteration, as long as the array's length is not changed during the loop.

Instances of this issue:

```
contracts/libraries/DataStorage.sol#L307    for (uint256 i = 0; i < secondsAgos.length; i++) {
```

6 - Use immutable variables can save gas
==

Variables that are assigned only in the constructor can be immutable

Instances of this issue:

```
contracts/AlgebraPoolDeployer.sol#L19    address private owner;
```

