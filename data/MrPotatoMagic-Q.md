# Quality Assurance Report

The report consists of low-severity risk issues, governance risk issues arising from centralization vectors and a few non-critical issues in the end. The governance risks section describes scenarios in which the actors could misuse their powers.

 - [Low-severity issues](#low-severity-issues)
 - [Governance risks](#governance-risks)

# Low-severity issues

| ID     | Issue                                                                                                       |
|--------|-------------------------------------------------------------------------------------------------------------|
| [L-01] | Initialize function can be re-initialized in RamsesV3Factory.sol contract                                   |
| [L-02] | Incorrect check for `$.fee` in function setFee()                                                            |
| [L-03] | Use two step mechanism to transfer treasury ownership                                                       |
| [L-04] | Collection of protocol fees can be bricked permanently due to incorrect authorization in FeeDistributor.sol |
| [L-05] | cachePeriodEarned() could revert when period is not advanced                                                |
| [L-06] | Late protocol users could face high gas consumption and possibly out-of-gas errors when claiming rewards    |
| [L-07] | Redundant owner event emission in ClGaugeFactory constructor or missing ownership access                    |

## [L-01] Initialize function can be re-initialized in RamsesV3Factory.sol contract

The function initialize() below is restricted to the `accessManager` to set the ramsesV3PoolDeployer. But the function does not add a one-time initialization check to ensure the function can only be called once. This allows the accessManager to change the ramsesV3PoolDeployer anytime in the future after it has been set. As mentioned in the contest README, the `accessManager` is expected to initially be a multisig of core contributors/stakeholders, but the team intends on moving all controls to a decentralized governance model over time. This could pose a problem in the decentralized governance model since if a proposal passes, it would be possible to change the pool deployer.
```solidity
File: RamsesV3Factory.sol
65:     function initialize(address _ramsesV3PoolDeployer) external restricted {
66:         require(ramsesV3PoolDeployer == address(0));
67:         ramsesV3PoolDeployer = _ramsesV3PoolDeployer;
68:     }
```

Solution: Add OZ's initializer() modifier from Initializable.sol or simply implement a custom one-time initialization check. If the intended behaviour is to allow the pool deployer to be modified in the future as well, consider renaming the function to something like setRamsesV3PoolDeployer().

## [L-02] Incorrect check for `$.fee` in function setFee()

On Line 78, the `_fee` parameter is checked to be less than 1e5 instead of 1e6. This is a problem since it restricts the `accessManager` from setting a specific pool's `$.fee` value in the range [100000, 1000000].

The issue is being reported as low-severity since the bug is present in the ProtocolActions.sol contract, which is out-of-scope for this contest. Nonetheless, the issue is worth pointing out to fix.
```solidity
File: ProtocolActions.sol
75:     function setFee(uint24 _fee, address factory) external {
76:         PoolStorage.PoolState storage $ = PoolStorage.getStorage();
77:         if (msg.sender != factory) revert Unauthorized();
78:         if (_fee > 100000) revert InvalidFee();
79:         uint24 _oldFee = $.fee;
80:         $.fee = _fee;
81:         emit FeeAdjustment(_oldFee, _fee);
82:     }
```

Solution: Use 1000000 instead of 100000 on Line 78.

## [L-03] Use two step mechanism to transfer treasury ownership

The treasury is an important variable that determines the fees paid to the fee distributor, which is paid out to the voters. If the parameter to the setter function setTreasury() below is passed as an incorrect value, the role will be lost and all fees assigned to the treasury in the future will be lost as well as the ability to update the fee percentage assigned to the voters.
```solidity
File: FeeCollector.sol
53:     function setTreasury(address _treasury) external override onlyTreasury {
54:         emit TreasuryChanged(treasury, _treasury);
55: 
56:         treasury = _treasury;
57:     }
```

## [L-04] Collection of protocol fees can be bricked permanently due to incorrect authorization in FeeDistributor.sol

**Note:** Since the root cause is in the FeeDistributor.sol contract, which is out-of-scope for this contest, the issue is being reported as low-severity. The issue should be fixed though.

In function collectProtocolFees() in FeeCollector.sol, we call the notifyRewardAmount() function on the FeeDistributor.sol contract. In function [notifyRewardAmount()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/FeeDistributor.sol#L111) though,  over [here](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/FeeDistributor.sol#L111), only the PairFees.sol contract is authorized to call the function. Due to this, the call would revert and the collection of fees would never be possible.

```solidity
File: FeeCollector.sol
144:         if (amount0 > 0) {
145:             token0.approve(address(feeDist), amount0);
146:             feeDist.notifyRewardAmount(address(token0), amount0);
147:         }
148:         if (amount1 > 0) {
149:             token1.approve(address(feeDist), amount1);
150:             feeDist.notifyRewardAmount(address(token1), amount1);
151:         }
```

Since the collection of fees is not possible, the [GaugeV3.notifyRewardAmount()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L147C1-L150C40) function will always revert since it has the pushFees() modifier on it that attempts to call collectProtocolFees() on the FeeCollector.sol contract. Due to this, the gauge will not be able to notify current period reward amounts. 

This breaks one of the main invariants defined the README [here](https://github.com/code-423n4/2024-10-ramses-exchange#main-invariants) - `Gauges should never be "bricked"`.

Solution: In FeeDistributor.sol's notifyRewardAmount() function, authorize the FeeCollector.sol contract to call it.

## [L-05] cachePeriodEarned() could revert when period is not advanced

The function cachePeriodEarned() is used by functions in the GaugeV3.sol contract to retrieve the rewards to transfer to the users.

In function cachePeriodEarned(), we make a staticcall to function positionPeriodSecondsInRange() on the RamsesV3Pool.sol contract. 
```solidity
File: GaugeV3.sol
303:             (bool success, bytes memory data) = address(pool).staticcall(
304:                 abi.encodeCall(
305:                     IRamsesV3PoolState.positionPeriodSecondsInRange,
306:                     (period, owner, index, tickLower, tickUpper)
307:                 )
308:             );
```

The [positionPeriodSecondsInRange()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L930) function exposes the library function [positionPeriodSecondsInRange()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/libraries/Position.sol#L215) to retrieve the amount of seconds the position was in range.

In Position.positionPeriodSecondsInRange(), we check if the params.period is greater than the currentPeriod. In short, we ensure the period we are retrieving the seconds for is not greater than the current period as seen on Line 227-228 below.
```solidity
File: Position.sol
222:     function positionPeriodSecondsInRange(
223:         PositionPeriodSecondsInRangeParams memory params
224:     ) public view returns (uint256 periodSecondsInsideX96) {
225:         PoolStorage.PoolState storage $ = PoolStorage.getStorage();
226: 
227:         uint256 currentPeriod = $.lastPeriod;
228:         if (params.period > currentPeriod) revert FTR();
```

The issue is that before we made the positionPeriodSecondsInRange() static call from cachePeriodEarned(), we did not call the advancePeriod() function on the RamsesV3Pool contract. Due to this, if a user tries to claim rewards for the current period at the start of that period through the GaugeV3 contract, we revert. This is because the `currentPeriod` variable stores the previous period's value and not the current period's value (assuming no swaps or flash loans occurred or no advancePeriod was directly called). Since the static call was requesting the current period's (let's say X) seconds in range, the check evaluated as X > X - 1, which is true and we revert.

Now you might ask why would a user even retrieve the rewards at the start of a period when: (1) There were no swaps or flash loans that generated fees for that period (2) There were no rewards notified through Voter.sol for the current period. 

The user does so since the GaugeV3 has two functions, notifyRewardAmountNextPeriod() and notifyRewardAmountForPeriod(), which are not used by any contracts but are instead called directly by external protocols/teams to provide incentives for long programs (as per sponsor). Due to this, there is a reason for a user to call and claim rewards for the current period.

The issue is more suited for low-severity though since the team can just call advancePeriod() on the RamsesV3Pool contract directly if it's not called for a long time from the start of the current period (e.g. for low-liquidity pools that do not receive much interactions). As such this does not pose a threat to the functionality or even rewards since we revert only until the period is advanced. 

It could pose a threat if the totalTokenSupply (rewards for that period) are low and the user misses out on claiming those rewards to another user due to the revert. But this too is an edge case scenario that only means a small loss of rewards.

Solution: Consider advancing period by calling _advancePeriod() before positionPeriodSecondsInRange() is static called.

## [L-06] Late protocol users could face high gas consumption and possibly out-of-gas errors when claiming rewards

In the internal function _getAllRewards() below, a new user of the protocol attempting to get rewards from all periods, loops through all the periods starting from the firstPeriod i.e. 0. This process consumes excessive gas for a user who is new to the protocol (e.g. 1 or 2 years since RamsesV3 launch). The new user would loop need to loop through around 48-96 periods to be able to claim all their recent rewards.
```solidity
File: GaugeV3.sol
520:             lastClaim = Math.max(
521:                 lastClaimByToken[tokens[i]][_positionHash],
522:                 firstPeriod
523:             ); 
524:            
525:             for (
526:                 uint256 period = lastClaim;
527:                 period <= currentPeriod;
528:                 ++period
529:             ) {
```

The issue is not major since the recently joined users could just utilize the [getPeriodReward()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L385) function to retrieve rewards from a specific period one by one. While this issue could be a problem for users in some cases, it's being pointed out to be aware of and is rated as low-severity due to no major functionality being affected.

## [L-07] Redundant owner event emission in ClGaugeFactory constructor or missing ownership access

The ClGaugeFactory's constructor below emits an event that mentions that the owner of the contract was changed to the msg.sender. But in reality, we can see that there is no owner of the contract. 

This means that either the event is redundant or the team has overlooked adding the owner role to control changing the storage variables in it.

Depending on the intended behaviour, consider either removing the event emission or adding ownership to the contract.
```solidity
File: ClGaugeFactory.sol
22:     constructor(
23:         address _nfpManager,
24:         address _votingEscrow,
25:         address _voter,
26:         address _feeCollector
27:     ) {
28:         nfpManager = _nfpManager;
29:         votingEscrow = _votingEscrow;
30:         voter = _voter;
31:         feeCollector = _feeCollector;
32: 
33:         emit OwnerChanged(address(0), msg.sender); 
34:     }
```

# Governance Risks

## Actors/Roles in the system
 - Access Manager
 - Treasury 
 - Voter (mainly for adding and removing reward tokens in gauge, which is indirectly controlled by Voter.sol's access manager)
 - External protocols

## Powers of each role (highest to lowest)

### Access Manager
 - The `accessManager` mainly holds power in the RamsesV3Factory.sol contract. It has the ability to update the RamsesV3PoolDeployer, enable fees for specific tick spacings, set the default protocol fee, set a pool specific protocol fee, set the FeeCollector and update the swap/flash fees. The more concerning aspects are the ability to change the pool deployer and updating the protocol fees and swap/flash fees at any time. While the team has mentioned in the README that the `accessManager` is initially a multisig of core contributors/stakeholders, but they intend on moving all controls to a decentralized governance model over time, we should still be concerned on how often these values can be updated. Finding [L-01] in this report describes how the deployer should not be updatable due to the possible risk it carries if updated to a malicious pool deployer by frontrunning a user's createPool() function call. As such here are my final suggestions: (1) Disallow updating the pool deployer after first initialization (2) Introduce a rate limit to how often atleast the protocol fees can be updated for a specific pool. I'm assuming if we move to a decentralized model, swap/flash fees would be updated frequently as per where the community decides the fee incentives to align, so introducing a rate limit for setFee() is not required.

### Treasury
 - The `treasury` holds power in the FeeCollector.sol contract. Other than having the power to transfer its own ownership, the treasury has full control over how the fees collected from the pools should be distributed among the treasury itself and the FeeDistributor (i.e. voters). The README though does not mention anything about the treasury being a trusted actor and atleast being a multisig of core contributors/stakeholders. Providing control over fee percentages is important for the voters to be interested in the protocol. As such, the treasury should also be a multisig model.

### Voter
 - The Voter contract is responsible for updating the reward tokens in the GaugeV3.sol contract. This is done through the voter's functions [whitelistGaugeRewards() and removeGaugeRewardWhitelist()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/Voter.sol#L279C1-L299C6). Ultimately, these are controlled by the Voter.sol contract's `accessManager`. This role is being pointed out since there was no mention of the out of scope Voter.sol contract's `accessManager` being a single trusted entity or a multisig transitioning to a decentralized governance model. A possible risk this carries is that reward tokens can be removed at any time, even when there are existing rewards pending in the GaugeV3.sol contract for past or future periods. My suggestion would be to track a global mapping that stores (address token => uint256 totalBalance). That way if there are existing tokens above a certain threshold in the GaugeV3.sol contract, the token cannot be removed. If this recommendation makes the procedure more complex, this could only be handled by the multisig manually in that case by re-adding the removed token to allow pending users to claim their rewards but would be a problem in case of a decentralized governance model, where the proposal could either pass or not to re-add the removed token.

### External protocols
 - In Gaugev3.sol, the team has introduced two functions, notifyRewardAmountNextPeriod() and notifyRewardAmountForPeriod(), which allow external protocols to notify reward tokens as an external incentive. This means external protocols have the power to influence the possible rewards earnable per period. While this does not seem to provide a direct risk to the RamsesV3 protocol, influencing the rewards earnable per period is a behaviour that the Ramses team needs to be aware of in case it could be misused in any off-chain or on-chain component that was not part of this audit scope.
