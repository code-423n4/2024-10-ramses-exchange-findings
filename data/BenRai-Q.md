## L-01: It is not possible to change the initial fee for a tick spacing once the tick spacing is enabled
In the `RamsesV3PoolFactory`, once an `initial fee` is set for a tick spacing, it is not possible to change it because there is no function for this. If for any reason the initial fee for a tick spacing needs to be changed, a new factory would need to be deployed which would require to deploy a new nfpManager since the `deployer` value in the nfpManager which is used for calculating pool addresses if immutable.

## L02: If a caller sends more tokens to the pool when swapping, this tokens will not be accounted for and will be stuck in the pool
When a swap happens, the amount the caller needs to pay is calculated, the current token balance of the pool is determined and the [uniswapV3SwapCallback]( https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L667C48-L667C69 ) is called on the caller.  
After that the new token balance of the pool is taken to check if the required amount of tokens was send.

```solidity
   uint256 balance0Before = balance0();
   IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
   if (balance0Before + uint256(amount0) > balance0()) revert IIA();
``` 

If enough tokens were sent, the function continues. 
The issue arises from the fact that if the caller send some additional funds, they are not accounted for and get stuck in the pool.
Consider adding any additional funds to the protocol fees like it is done in the [flash loan function]( https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L706-L724).

The same issue also arises in the `mint` function.  

## L03: The values `initialized` and `nfpManager` can be removed from the `PoolState` struct because they are never set and never used


## L04: The variable `votingEscrow` is never used in the `ClGaugeFactory` and can be removed

## L05: The `ClGaugeFactory` no longer has an owner therefore the `OwnerChanged` event should not be emitted in the constructor and can be removed from `IClGaugeFactory ` 

## L06: The constant `PRECISION` is never used in `GaugeV3` and can be removed

## L07: The function `earned` in the GaugeV3 only works for positions created by the nfpManager
To make the gauge better compatible with other protocols, consider adding a function which allows the lookup of `earned` for positions which were not created through the nfpManager.

## L08: `GaugeV3::getPeriodReward` allowes for claims for future periods
To prevent potential exploits like the recent on in RamsesV2 add a check if the desired period is in the future. If so, revert the function.

## L09: In `GaugeV3::_getRewards` the `lastClaimByToken` is set to currentPeriod - 1
This will result in an unnecessary iteration through the last finished period where all rewards have been claimed already when all fees are claimed next time. To avoid this unneeded iteration set `lastClaimByToken[tokens[i]][_positionHash]` to `currentPeriod`.

## L10: Remove the mentioning of Boost in line 69 of `IGaugeV3`

## L11: No need to have the line `pragma abicoder v2;` in `NonfungiblePositionManager`
`abicoder v2` is [enabled by default in v8.0 and above]( https://docs.soliditylang.org/en/v0.8.0/080-breaking-changes.html )

## L12: Calculating `tokensOwed0` and `tokensOwed1` in `NonfungiblePositionManager` can be removed
Just use the return values from the `pool.positions(positionKey)` which already returns the values for `tokensOwed0` and `tokensOwed1` saved in the pool
