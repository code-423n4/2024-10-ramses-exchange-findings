# `periodSecondsPerLiquidityOutsideX128[period]` should be initialized in `update()`

## Links to affected code

https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/libraries/Tick.sol#L118-L124

## Details

The `RamsesV3` contract uses the same tick algorithms from UniswapV3 and introduces new variables based on these algorithms. One such variable is `periodSecondsPerLiquidityOutsideX128`, which tracks the seconds per liquidity generated on the "other" side of a given tick, capped based on the period's information.

This variable is relevant in two key functions:

- `cross()`:
    ```solidity
    function cross(/* ... */) /* ... */ {
        unchecked {
            // ...
            uint256 periodSecondsPerLiquidityOutsideX128;
            uint256 periodSecondsPerLiquidityOutsideBeforeX128 = info.periodSecondsPerLiquidityOutsideX128[period];
            if (tick <= periodStartTick && periodSecondsPerLiquidityOutsideBeforeX128 == 0) {
                periodSecondsPerLiquidityOutsideX128 =
                    secondsPerLiquidityCumulativeX128 -
                    /* periodSecondsPerLiquidityOutsideBeforeX128 - */
                    endSecondsPerLiquidityPeriodX128;
            } else {
                periodSecondsPerLiquidityOutsideX128 =
                    secondsPerLiquidityCumulativeX128 -
                    periodSecondsPerLiquidityOutsideBeforeX128;
            }
            info.periodSecondsPerLiquidityOutsideX128[period] = periodSecondsPerLiquidityOutsideX128;
        }
    }
    ```
 
- `periodCumulativesInside()`:
    ```solidity
    function periodCumulativesInside(/* ... */) /* ... */ {
        // ...

        {
            int24 startTick = $.periods[period].startTick;
            uint256 previousPeriod = $.periods[period].previousPeriod;

            snapshot.secondsPerLiquidityOutsideLowerX128 = uint160(lower.periodSecondsPerLiquidityOutsideX128[period]);

            if (tickLower <= startTick && snapshot.secondsPerLiquidityOutsideLowerX128 == 0) {
                snapshot.secondsPerLiquidityOutsideLowerX128 = $
                    .periods[previousPeriod]
                    .endSecondsPerLiquidityPeriodX128;
            }

            snapshot.secondsPerLiquidityOutsideUpperX128 = uint160(upper.periodSecondsPerLiquidityOutsideX128[period]);
            if (tickUpper <= startTick && snapshot.secondsPerLiquidityOutsideUpperX128 == 0) {
                snapshot.secondsPerLiquidityOutsideUpperX128 = $
                    .periods[previousPeriod]
                    .endSecondsPerLiquidityPeriodX128;
            }
        }
        // ...
    }
    ```
    
These functions follow UniswapV3's logic by assuming all growth happens below the tick before it is initialized. However, note that while these functions initialize `periodSecondsPerLiquidityOutsideX128` when the tick is crossed or checked in `periodCumulativesInside()`, the value is not checkpointed in `update()` when the tick is first initialized:

```solidity
function update(/* ... */) /* ... */ {
    // ...
    if (liquidityGrossBefore == 0) {
        /// @dev by convention, we assume that all growth before a tick was initialized happened _below_ the tick
        if (tick <= tickCurrent) {
            info.feeGrowthOutside0X128 = feeGrowthGlobal0X128;
            info.feeGrowthOutside1X128 = feeGrowthGlobal1X128;
            info.secondsPerLiquidityOutsideX128 = secondsPerLiquidityCumulativeX128;
            info.tickCumulativeOutside = tickCumulative;
            info.secondsOutside = time;
        }
        info.initialized = true;
    }
    // ...
}
```

As a result, if a tick is newly initialized during a period, the logic in `cross()` and `periodCumulativesInside()` assumes that fee growth has occurred since the start of the period, even if it actually began in the middle of the period. Fortunately, this does not cause issues in the current codebase due to the behavior of `initializeSecondsStart()`, specifically the `secondsPerLiquidityPeriodStartX128` value that is recorded when each position is initialized.

## Recommended Mitigation Steps

Consider adding logic to `update()` to "checkpoint" the fee growth for the current period when the tick is initialized:

```diff
function update(/* ... */) /* ... */ {
    // ...
    if (liquidityGrossBefore == 0) {
        /// @dev by convention, we assume that all growth before a tick was initialized happened _below_ the tick
        if (tick <= tickCurrent) {
            info.feeGrowthOutside0X128 = feeGrowthGlobal0X128;
            info.feeGrowthOutside1X128 = feeGrowthGlobal1X128;
            info.secondsPerLiquidityOutsideX128 = secondsPerLiquidityCumulativeX128;
            info.tickCumulativeOutside = tickCumulative;
            info.secondsOutside = time;
+           info.periodSecondsPerLiquidityOutsideX128[currentPeriod] = secondsPerLiquidityCumulativeX128;
        }
        info.initialized = true;
    }
    // ...
}
```

# `burn()` does not consider unclaimed gauge rewards

## Links to affected code

https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/periphery/NonfungiblePositionManager.sol#L398

## Details

In the `NonfungiblePositionManager` contract, the `burn()` function permanently burns the LP position ERC721 token:

```solidity
function burn(uint256 tokenId) external payable override isAuthorizedForToken(tokenId) {
    Position storage position = _positions[tokenId];
    if (position.liquidity > 0 || position.tokensOwed0 > 0 || position.tokensOwed1 > 0) revert NotCleared();
    delete _positions[tokenId];
    _burn(tokenId);
}
```

This function ensures the token is "empty" by checking that the `liquidity` and `tokensOwed0`/`tokensOwed1` values are zero. However, it does not account for unclaimed gauge rewards associated with the token. As a result, a user could burn their token while still having unclaimed gauge rewards.

## Recommended Mitigation Steps

Consider documenting this behavior in the comments above `burn()`, or alternatively, remove the `burn()` function to prevent potential loss of gauge rewards.


# `_advancePeriod()` can be called before `initialize()`

## Links to affected code

https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L749-L778

## Details

In normal circumstances, a `RamsesV3` pool is deployed and then immediately initialized by calling `initialize()`, which has the following implementation:

```solidity
function initialize(uint160 sqrtPriceX96) external {
    PoolStorage.PoolState storage $ = PoolStorage.getStorage();

    if ($.slot0.sqrtPriceX96 != 0) revert AI();

    int24 tick = TickMath.getTickAtSqrtRatio(sqrtPriceX96);

    (uint16 cardinality, uint16 cardinalityNext) = Oracle.initialize($.observations, 0);

    _advancePeriod();

    $.slot0 = Slot0({
        sqrtPriceX96: sqrtPriceX96,
        tick: tick,
        observationIndex: 0,
        observationCardinality: cardinality,
        observationCardinalityNext: cardinalityNext,
        feeProtocol: 0,
        unlocked: true
    });

    emit Initialize(sqrtPriceX96, tick);
}
```

However, note that the `RamsesV3Factory` does not explicitly require that `initialize()` be called on deployment. The following logic only executes `initialize()` when a non-zero `sqrtPriceX96` is provided:

```solidity
function createPool(/* ... */) /* ... */ {
    // ...
    if (sqrtPriceX96 > 0) {
        IRamsesV3Pool(pool).initialize(sqrtPriceX96);
    }
}
```

As a result, it is possible to deploy a `RamsesV3` pool without calling `initialize()`. This means that someone could independently call `_advancePeriod()` outside of the normal control flow where it is called automatically by `initialize()`.

While this behavior does not appear exploitable, it may be preferable to prevent it entirely.

## Recommended Mitigation Steps

Consider preventing `_advancePeriod()` from being called before `initialize()` by making `_advancePeriod()` internal, introducing an external `advancePeriod()` function, and adding the `lock` modifier to the new external function.


# Combination of overflows and `uint160()` casting is dangerous

## Links to affected code

https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/libraries/Oracle.sol#L470-L549

## Details

The `periodCumulativesInside()` function is designed to calculate the seconds per liquidity inside a tick range for a specific period. However, due to factors such as how `endSecondsPerLiquidityPeriodX128` is initialized on crossing and how `initialize()` calls `_advancePeriod()` (which sets the `endTick` and `startTick` before `initialize()` sets `$.slot0.tick`), the return value of `periodCumulativesInside()` can sometimes overflow.

Despite the potential for overflow, this issue does not appear exploitable in the current codebase. The function is only used by individual positions that track changes in `periodCumulativesInside()` over time, and this works correctly even if the individual values overflow.

However, note that the overflowed values are typecast into `uint160`, and later into `int160`. For example, in the following `unchecked` block, an overflow can occur, and the return value will be implicitly cast into `uint160`:

```solidity
function periodCumulativesInside(/* ... */) external view returns (uint160 secondsPerLiquidityInsideX128) {
    // ...
    unchecked {
        if (lastTick < tickLower) {
            return snapshot.secondsPerLiquidityOutsideLowerX128 - snapshot.secondsPerLiquidityOutsideUpperX128;
        } else if (lastTick < tickUpper) {
            // ...
            return
                cache.secondsPerLiquidityCumulativeX128 -
                snapshot.secondsPerLiquidityOutsideLowerX128 -
                snapshot.secondsPerLiquidityOutsideUpperX128;
        } else {
            return snapshot.secondsPerLiquidityOutsideUpperX128 - snapshot.secondsPerLiquidityOutsideLowerX128;
        }
    }
}
```

This behavior seems dangerous, as the overflow leads to calculations with extremely large values, relying on future overflows to correct them. The `uint160` and `int160` introduces further complexity and essentially makes all calculations become modulo 160 bits, which is difficult to reason about.

While no specific exploit was identified during the contest, this behavior may be best to eliminate for extra safety.

## Recommended Mitigation Steps

Consider removing the `uint160` typecast and instead use `uint256` and `int256` for `secondsPerLiquidityInsideX128`. This would reduce complexity and make the code easier to reason about.
