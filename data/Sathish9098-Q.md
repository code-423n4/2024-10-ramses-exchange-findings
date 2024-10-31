# QA Report

##

## [L-1] Compromised Treasury Address Can Permanently Block Updates and steal protocol funds

## Impact

Permanent ``fund loss`` and the protocol being stuck in an ``unrecoverable state``.

If the treasury address gets compromised (e.g., private key stolen or malicious insider), the attacker gains full control over the treasury operations and steal the protocol fee indefinitely. 

If an attacker controls the treasury, they can block updates by refusing to change the address or setting it to a malicious address that locks further changes.

The protocol's ability to recover control is lost since only the compromised treasury has the power to update itself.

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/gauge
/FeeCollector.sol


 /// @inheritdoc IFeeCollector
    function setTreasury(address _treasury) external override onlyTreasury {
        emit TreasuryChanged(treasury, _treasury);

        treasury = _treasury;
    }

```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/FeeCollector.sol#L49-L54

### POC

- ``Alice`` is the initial treasury of a smart contract, with control over treasury-related functions.
- The ``onlyTreasury`` modifier ensures only the treasury address can update itself or perform restricted actions.
- ``Bob`` (a malicious actor) steals Alice’s private key through phishing or malware.
- Using the stolen key, Bob changes the treasury address to his own wallet (0xBob).
- Alice can no longer restore control because only ``Bob's`` address is now authorized.
- ``Bob`` can steal protocol fee funds fully or collect fees as the new treasury.
- ``Bob`` might set the treasury to a black-hole address (e.g., 0xdead), permanently locking the contract.
- Without an admin or backup mechanism, the team cannot recover control.

### Recommended Mitigation
Implement only the admins (onlyOwner) can change new treasury address instead of old treasury itself.

##

## [L-2] Reverts Occur When ``staticcall`` Is Used on Non-View/Non-Pure Functions

The ``staticcall`` may reverts because ``cachePeriodEarned()`` is not marked as view or pure. The EVM enforces that only read-only functions can be called via ``staticcall``. 

As per [docs]
(https://github.com/ethereum/EIPs/blob/master/EIPS/eip-214.md) the staticcall is only suitable for read-only functions. But the function ``cachePeriodEarned `` is declared as external which is not advisable as per docs.

> Any attempts to make state-changing operations inside an execution instance with STATIC set to true will instead throw an exception. These operations include CREATE, CREATE2, LOG0, LOG1, LOG2, LOG3, LOG4, SSTORE, and SELFDESTRUCT. They also include CALL with a non-zero value. As an exception, CALLCODE is not considered state-changing, even with a non-zero value.

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/gauge
/GaugeV3.sol

(bool success, bytes memory data) = address(this).staticcall(
            abi.encodeCall(
                this.cachePeriodEarned,
                (period, token, owner, index, tickLower, tickUpper, false)
            )
        );


 function cachePeriodEarned(
        uint256 period,
        address token,
        address owner,
        uint256 index,
        int24 tickLower,
        int24 tickUpper,
        bool caching
    ) external returns (uint256);

```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L262-L267

### Recommended Mitigation
Use regular ``call ``

##

## [L-3] Incorrect Protocol Fee Calculation in ``flash()`` Function

The issue is related to how protocol fees are calculated. Specifically, applying fees to the already-paid fees (``fee-on-fee logic``) can create several risks that affect both the protocol’s revenue and user interactions. 

### Problem
The ``flash()`` function attempts to collect protocol fees based on the total amount repaid (``paid0`` and ``paid1``), which includes both the original loan amount and the loan fee. This introduces a ``compounding fee effect``, where the protocol effectively taxes itself by charging fees on both the base amount and the fee amount. Specifically, applying fees to the already-paid fees (``fee-on-fee logic``) .

## Impact

- This results in less revenue than expected because fees are being deducted twice—once from the original transaction and again from the protocol’s portion.
- Over time, this fee compounding could ``erode the protocol’s revenue``, especially in high-frequency operations such as flash loans.

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/core
/RamsesV3Pool.sol

 /// @dev sub is safe because we know balanceAfter is gt balanceBefore by at least fee
            uint256 paid0 = balance0After - balance0Before;
            uint256 paid1 = balance1After - balance1Before;

            uint8 feeProtocol = $.slot0.feeProtocol;
            if (paid0 > 0) {
                uint256 pFees0 = feeProtocol == 0 ? 0 : paid0 / feeProtocol;
                if (uint128(pFees0) > 0) $.protocolFees.token0 += uint128(pFees0);
                $.feeGrowthGlobal0X128 += FullMath.mulDiv(paid0 - pFees0, FixedPoint128.Q128, _liquidity);
            }
```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L707-L716

### Recommended Mitigation
Reduce the fee0 and fee1 from ``paid0`` and ``paid1``.

```solidity

uint256 paid0 = balance0After - balance0Before-fee0;
uint256 paid1 = balance1After - balance1Before-fee1;

```



##

## [L-4] Authorization Bypass via User-Provided ``Owner`` Address in ``getPeriodReward()`` function

The ``getPeriodReward()`` function has a vulnerability because the ``owner`` address is provided as a user input. This allows an attacker to bypass the authorization check:

```solidity

require(msg.sender == owner, "Not authorized");

```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L394

Anyone can provide his own address as owner and can bypass that authorization check. This check is very weak check and not standard way to check the owner. This will allow claiming others rewards without any problem if the attacker just need to know the period and token address.  

### Recommended Mitigation
Retrieve the owner address internally from the NFT position manager, ensuring that the check is based on the actual owner

```solidity
address realOwner = nfpManager.ownerOf(tokenId);
require(msg.sender == realOwner, "Not authorized");

``` 

##

## [L-5] Risk of Zero Fee Setting Leading to Revenue Loss

The ``setTreasuryFees`` function allows the treasury fee to be set to 0 without any restriction, as there is no explicit check to prevent it. If the fee is set to 0, the protocol may stop collecting fees, which could lead to:

- Loss of revenue for the treasury.
- Unintended behavior where no fees are accumulated or distributed. 

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/gauge
/FeeCollector.sol

/// @inheritdoc IFeeCollector
    function setTreasuryFees(
        uint256 _treasuryFees
    ) external override onlyTreasury {
        require(_treasuryFees <= BASIS, FTL());
        emit TreasuryFeesChanged(treasuryFees, _treasuryFees);

        treasuryFees = _treasuryFees;
    }

```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/FeeCollector.sol#L56-L64

## Recommended Mitigation
Add a Minimum Fee Check

```solidity
require(_treasuryFees > 0, "Fee cannot be zero");  // Prevent 0 fee
``` 

##

## [L-6] Flash Loan (``flash()``) Reverts When Requested Amount Exceeds Available Token Balances

In the flash loan function, the transaction will revert if the requested amount of token0 or token1 exceeds the available balance held by the contract. 

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/core/RamsesV3Pool.sol

if (amount0 > 0) TransferHelper.safeTransfer(token0, recipient, amount0);
if (amount1 > 0) TransferHelper.safeTransfer(token1, recipient, amount1);

```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L695-L696

- ``safeTransfer`` ensures that the requested tokens are successfully transferred to the borrower.
- If the contract’s balance of ``token0`` or ``token1`` is insufficient, the transfer fails and triggers a revert.

### Recommended Mitigation
Add Preemptive Checks can prevent reverts during the transfer by checking the contract's balance before initiating the transfer.

```solidity

if (amount0 > balance0()) revert InsufficientToken0();
if (amount1 > balance1()) revert InsufficientToken1();

```

##

## [L-7] ``createPool()`` Callable by Anyone but Reverts if Caller is Not the ``RamsesV3Factory``

The ``createPool()`` function appears to be publicly callable, meaning that any user can attempt to call it. However, the actual pool deployment will revert internally if the caller is not the authorized deployer (i.e., the RamsesV3Factory).

The issue lies in the misalignment between the function’s visibility and its actual behavior.

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/core
/RamsesV3Factory.sol

 /// @inheritdoc IRamsesV3Factory
    function createPool(
        address tokenA,
        address tokenB,
        int24 tickSpacing,
        uint160 sqrtPriceX96
    ) external override returns (address pool) {
        require(tokenA != tokenB, IT());
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        require(token0 != address(0), A0());
        uint24 fee = tickSpacingInitialFee[tickSpacing];
        require(fee != 0, F0());
        require(getPool[token0][token1][tickSpacing] == address(0), PE());

        parameters = Parameters({
            factory: address(this),
            token0: token0,
            token1: token1,
            fee: fee,
            tickSpacing: tickSpacing
        });
        pool = IRamsesV3PoolDeployer(ramsesV3PoolDeployer).deploy(token0, token1, tickSpacing); //@audit RamsesV3Factory only call this function. but createPool() function callable by anyone 
        delete parameters;

        getPool[token0][token1][tickSpacing] = pool;
        /// @dev populate mapping in the reverse direction, deliberate choice to avoid the cost of comparing addresses
        getPool[token1][token0][tickSpacing] = pool;
        emit PoolCreated(token0, token1, fee, tickSpacing, pool);

        /// @dev if there is a sqrtPrice, initialize it to the pool
        if (sqrtPriceX96 > 0) {
            IRamsesV3Pool(pool).initialize(sqrtPriceX96);
        }
    }


```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Factory.sol#L70C4-L104C1

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/core
/RamsesV3PoolDeployer.sol

/// @dev Deploys a pool with the given parameters by transiently setting the parameters storage slot and then
    /// clearing it after deploying the pool.
    /// @param token0 The first token of the pool by address sort order
    /// @param token1 The second token of the pool by address sort order
    /// @param tickSpacing The tickSpacing of the pool
    function deploy(address token0, address token1, int24 tickSpacing) external returns (address pool) {
        require(msg.sender == RamsesV3Factory);
        pool = address(new RamsesV3Pool{salt: keccak256(abi.encodePacked(token0, token1, tickSpacing))}());
    }

```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3PoolDeployer.sol#L22

### Recommended Mitigation
Clearly document that ``createPool()`` function only callable by ``RamsesV3Factory`` address

##

## [L-8] Inconsistent Period Transitions and Temporal Regression in Time-Based Logic

In the provided code from ``RamsesV3Pool.sol``, the function checks whether the current time has moved to a new week. The original condition was:

```solidity

 if (period != _lastPeriod) {

```

The ``_advancePeriod()`` function manages time-based logic by grouping transactions into weekly periods (defined as 1 weeks). This means that each week is assigned a specific period number, derived from:

```solidity
uint256 period = _blockTimestamp() / 1 weeks;
```

- ``_blockTimestamp()`` returns the current block’s timestamp (in seconds).
- Dividing by 1 week groups all timestamps within the same week under the same period.
The goal of the logic is to check if we’ve moved to a new period (i.e., from one week to another). If the period has changed, the code updates the storage to reflect the new period and resets the related data.

In the original code:

```solidity

if (period != _lastPeriod) { ... }

```

https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L429-L435

This check triggers whenever the new period is different from the previous period (``_lastPeriod``). However, this approach has a potential flaw:

It will still trigger if the current transaction somehow comes from a "previous" period (a period earlier than ``_lastPeriod``).
- It may not properly handle unexpected time shifts or anomalies, like:
Reorgs (blockchain reordering).
- Malicious actors attempting to manipulate the time-based logic by backdating or replaying transactions.

Even though transactions should only move forward in time, if something goes wrong (like block timestamp manipulation), the original logic could behave incorrectly by allowing mismatched periods to slip through.

### Recommended Mitigation
Change the check to 

```solidity

if (period > _lastPeriod) { ... }

```

##

## [L-9] Incorrect Non-Negative Check on Unsigned Integer ``_liquidity``

The condition if (``_liquidity <= 0``) is incorrect because _liquidity is ``uint128`` being less than 0 is impossible. The only meaningful edge case is when _liquidity == 0, which should be explicitly checked. The incorrect condition cause confusion. 

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/core/RamsesV3Pool.sol

if (_liquidity <= 0) revert L();

```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L688

### Recommended Mitigation
```solidity

if (_liquidity == 0) revert L();

```

##

## [L-10] Compromised or malicious Treasury Can Disable Fee Distribution and Seize All Protocol Fees

### Impact
The attacker can collect all accumulated protocol fees by exploiting the ``isTokenLive`` flag.

The gauge-based fee distribution system becomes non-functional, affecting ``users`` and ``LPs`` who are supposed to receive a ``share of the fees``.

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/gauge/FeeCollector.sol

 /// @notice for tokenless deployments, disable fee distribution until ve system exists
    function setIsLive(bool status) external onlyTreasury {
        isTokenLive = status;
    }

 /// @dev if there's no gauge, there's no fee distributor, send everything to the treasury directly
        if (gauge == address(0) || !isAlive || !isTokenLive) {
            (uint128 _amount0, uint128 _amount1) = pool.collectProtocol(
                treasury,
                type(uint128).max,
                type(uint128).max
            );

```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/FeeCollector.sol#L89-L95

The ``setIsLive`` function allows the treasury to control the ``isTokenLive`` status, which determines whether fees are distributed via the fee distributor or sent directly to the treasury. If the treasury address is compromised or controlled by a malicious actor, they could:

- Set isTokenLive to false by calling setIsLive.

- This triggers the following logic

```solidity

if (gauge == address(0) || !isAlive || !isTokenLive) {
    pool.collectProtocol(treasury, type(uint128).max, type(uint128).max);
}

```

Instead of distributing fees properly, all protocol fees are collected directly by the compromised treasury.

### Recommended Mitigation
Introduce an Admin Role. Implement the onlyOwner modifier to control important changes 

##

## [L-11] Failure to Block Zero Amount Rewards in ``notifyRewardAmountNextPeriod()`` and ``notifyRewardAmountForPeriod()`` functions

Current implementation of [notifyRewardAmountNextPeriod()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L165-L179) and [notifyRewardAmountForPeriod()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L182-L197) functions allows the even the reward amount is 0

### Recommended Mitigation
Adding a ``require(amount > 0)`` check ensures that the function only proceeds with valid, non-zero transfers.

##

## [L-12] Unbounded Gas Usage in ``staticcall`` Can Lead to Out-of-Gas Errors and DoS Attacks 

Using a fixed gas limit with ``staticcall`` ensures that the call behaves predictably and prevents denial of service (DoS) attacks via gas griefing. Without a fixed gas limit, a malicious contract could perform gas-heavy operations (like infinite loops), exhausting the caller’s gas. Even though staticcall retains some gas, only ``1/64th`` of the remaining gas can be forwarded, per ``EIP-150`` rules. This makes setting a reasonable gas cap essential to avoid unexpected out-of-gas errors and improve reliability when calling external functions as per [docs](https://www.rareskills.io/post/solidity-staticcall)

```solidity
FILE: 2024-10-ramses-exchange/contracts/CL/gauge/GaugeV3.sol

 (bool success, bytes memory data) = address(this).staticcall(
            abi.encodeCall(
                this.cachePeriodEarned,
                (period, token, owner, index, tickLower, tickUpper, false)
            )
        );

```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L262

### Recommended Mitigation
Pass the gas limit value along with ``staticcall``

##

## [L-13] ``getPeriodReward()`` Fails to Enforce Validation for Future Period Claims 

The ``getPeriodReward()`` function allows claiming rewards for a given period. However, the current implementation lacks a strict validation to prevent claims for future periods.

```solidity

if (period < _blockTimestamp() / WEEK) {
    lastClaimByToken[tokens[i]][_positionHash] = period;
}

```

This check ensures claims can be processed only if the period is less than the current week. However, there is no require statement explicitly preventing future period claims, meaning this condition is bypassed or manipulated elsewhere, rewards could be claimed prematurely for upcoming periods, disrupting the reward system and causing financial inconsistencies.

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/gauge
/GaugeV3.sol

/// @inheritdoc IGaugeV3
    function getPeriodReward(
        uint256 period,
        address[] calldata tokens,
        uint256 tokenId,
        address receiver
    ) external override lock {
        INonfungiblePositionManager _nfpManager = nfpManager;
        address owner = _nfpManager.ownerOf(tokenId);
        address operator = _nfpManager.getApproved(tokenId);

        /// @dev check if owner, operator, or approved for all
        require(
            msg.sender == owner ||
                msg.sender == operator ||
                _nfpManager.isApprovedForAll(owner, msg.sender),
            "Not authorized"
        );

        (, , , int24 tickLower, int24 tickUpper, , , , , ) = _nfpManager
            .positions(tokenId);

        bytes32 _positionHash = positionHash(
            address(_nfpManager),
            tokenId,
            tickLower,
            tickUpper
        );

        for (uint256 i = 0; i < tokens.length; ++i) {
            if (period < _blockTimestamp() / WEEK) {
                lastClaimByToken[tokens[i]][_positionHash] = period;
            }

            _getReward(
                period,
                tokens[i],
                address(_nfpManager),
                tokenId,
                tickLower,
                tickUpper,
                _positionHash,
                receiver
            );
        }
    }



```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L337-L382

### Recommended Mitigation
Add a strict require statement to prevent future period claims

```solidity
require(period <= _blockTimestamp() / WEEK, "Cannot claim future rewards");

```

##

## [L-14] Unsafe ``unchecked`` Block and Potential Overflow Risk 

The use of the ``unchecked`` block in the code introduces a risk of overflow if the ``tokensOwed0`` and ``tokensOwed1`` values grow too large, especially when fees accumulate over a long period without being collected by the position owner. Although ``the comment suggests that the fees should be collected to avoid overflow``, there is ``no enforced deadline`` for the owner to collect them.

If an overflow occurs in the ``tokensOwed0`` or ``tokensOwed1`` calculations, the fees reset to 0, causing the owner to lose accumulated fees. 

Since arithmetic overflow checks are disabled inside the unchecked block, Solidity's default protections do not apply. This makes overflow possible if the position accumulates fees over an extended period without being claimed.

Without a mechanism or deadline to force fee collection, users may forget to claim fees, increasing the risk of overflow and loss of value.

```solidity
FILE: 2024-10-ramses-exchange/contracts/CL/periphery
/NonfungiblePositionManager.sol

  unchecked {
            position.tokensOwed0 += uint128(
                FullMath.mulDiv(
                    feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
                    position.liquidity,
                    FixedPoint128.Q128
                )
            );
            position.tokensOwed1 += uint128(
                FullMath.mulDiv(
                    feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
                    position.liquidity,
                    FixedPoint128.Q128
                )
            );
        }

```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/periphery/NonfungiblePositionManager.sol#L236C7-L251C10

### Recommended Mitigation
Consider removing ``unchecked`` from ``position.tokensOwed0`` and ``position.tokensOwed1`` blocks .

##

## [L-15] ``getPeriodReward()`` Function Allows Transfers to Zero Address Without ``receiver`` address Validation 

The ``getPeriodReward()`` function does not enforce a check to ensure the receiver is not the zero address (address(0)).

The ``safeTransfer`` reverts if the recipient is the zero address (address(0)), which adds a layer of protection. However, it’s still good practice to explicitly check for address(0) in your function to prevent unnecessary calls and ensure clarity and consistency across your contract.

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/gauge/GaugeV3.sol

 /// @inheritdoc IGaugeV3
    function getPeriodReward(
        uint256 period,
        address[] calldata tokens,
        uint256 tokenId,
        address receiver
    ) external override lock {
        INonfungiblePositionManager _nfpManager = nfpManager;
        address owner = _nfpManager.ownerOf(tokenId);
        address operator = _nfpManager.getApproved(tokenId);

        /// @dev check if owner, operator, or approved for all
        require(
            msg.sender == owner ||
                msg.sender == operator ||
                _nfpManager.isApprovedForAll(owner, msg.sender),
            "Not authorized"
        );

        (, , , int24 tickLower, int24 tickUpper, , , , , ) = _nfpManager
            .positions(tokenId);

        bytes32 _positionHash = positionHash(
            address(_nfpManager),
            tokenId,
            tickLower,
            tickUpper
        );

        for (uint256 i = 0; i < tokens.length; ++i) {
            if (period < _blockTimestamp() / WEEK) {
                lastClaimByToken[tokens[i]][_positionHash] = period;
            }

            _getReward(
                period,
                tokens[i],
                address(_nfpManager),
                tokenId,
                tickLower,
                tickUpper,
                _positionHash,
                receiver
            );
        }
    }


```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L338-L382

### Recommended Mitigation
Updated Code with ``address(0)`` Check

```solidity

require(receiver != address(0), "Invalid receiver address");

```

##

## [L-16] ``getPeriodReward()` always Reverts When receiver is ``address(0)``

Instead of reverting when the receiver address is address(0), the function can be modified to automatically redirect the rewards to the owner of the position. This ensures that rewards are not lost and avoids unnecessary reverts while maintaining flexibility. 

[getPeriodReward()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L338-L382)

### Recommended Mitigation
Add this line to ``getPeriodReward()`` function like [collect()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/periphery/NonfungiblePositionManager.sol#L332) function did

```solidity

// If receiver is address(0), default to the owner of the token
receiver = receiver == address(0) ? owner : receiver;

```

##

## [L-17] Setters Lack Equality and Zero Address Checks, Allowing Redundant Updates and Risk of Invalid Address Assignment

Setting the same address as the existing ``feeCollector`` results in unnecessary events being emitted and wasted gas costs.

If the fee collector is mistakenly set to ``address(0)``, any collected fees would be burned and unrecoverable, disrupting the protocol's financial operations.

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/core
/RamsesV3Factory.sol

 /// @inheritdoc IRamsesV3Factory
    function setFeeCollector(address _feeCollector) external override restricted {
        emit FeeCollectorChanged(feeCollector, _feeCollector);
        feeCollector = _feeCollector;
    }


```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Factory.sol#L168-L172

### Recommended Mitigation
Add this checks 

```solidity
require(_feeCollector != address(0), "Invalid address: zero address");
require(_feeCollector != feeCollector, "Address already set as fee collector");

```








