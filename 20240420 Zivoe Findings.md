
3/4 issues accepted 2x high, 1x medium
Should have also submitted the issues I found relating to first month of yield not calculating correctly & dust attack in ZioeRewards as both of these issues were accepted.

# Current issues to write up / submit

# (accepted) [H] ITO participants' vested ZVE rewards are diluted by post-ITO deposits into tranches due to rewards being denominated by the total supply of `zSTT` and `zJTT` at the time when claimAirdrop() is called.

## Summary

Any ITO depositor who calls `ZivoeITO::claimAirdrop()` after any post-ITO deposit is made to either tranche will have a diluted allocation of vested $ZVE. The amount of dilution is dependant on the volume of post-ITO deposits which occur before the user calls `ZivoeITO::claimAirdrop()`. This creates potential unfairness along with necessarily allocating less than the intended amount of vested $ZVE to ITO participants.

## Vulnerability Detail

The ITO is a liquidity-bootstrapping period for the Zivoe protocol. Early users are incentivized to deposit stable coins into the Senior and Junior tranches. The ITO period concludes when `ZivoeITO::migrateDeposits()` is called. At this point:

1. Business-as-usual deposits are open in the Junior and Senior tranches, along with usual minting of `zSTT` and `zJTT` tokens, and;
2. Users can claim their vested $ZVE incentives by calling `ZivoeITO::claimAirdrop()`

Users who are unable to `ZivoeITO::claimAirdrop()` before any deposits in step 1, will be allocated less zested $ZVE than if they called `ZivoeITO::claimAirdrop()` before any deposits in step 1. This is due their allocation being denominated in the total supply of `zSTT` and `zJTT` tokens which will stricly stay the same of increase after the ITO period.

As shown below, the user's allocation of zested $ZVE is made up of three components, `upper`, `middle`, and `lower`. When post-ITO liquidity enters the tranches before `ZivoeITO::claimAirdrop()` is called by any ITO depositor, the value of `lower` increases which dilutes ITO depositors' allocation. In addition, the value of `middle` will never fully be allocated to ITO depositors, even if they all call this function. 

https://github.com/sherlock-audit/2024-03-zivoe-Nihavent/blob/a566fe2fd4529013234520cde9b00a2b2a17da45/zivoe-core-foundry/src/ZivoeITO.sol#L223-L239

```javascript

        // Calculate proportion of $ZVE awarded based on $pZVE credits.
        uint256 upper = seniorCreditsOwned + juniorCreditsOwned;                                      
        uint256 middle = IERC20(IZivoeGlobals_ITO(GBL).ZVE()).totalSupply() / 20;   // Less than this will be allocated
@>      uint256 lower = IERC20(IZivoeGlobals_ITO(GBL).zSTT()).totalSupply() * 3 + ( // Increases with post-ITO tranche deposits
@>          IERC20(IZivoeGlobals_ITO(GBL).zJTT()).totalSupply()
        );

        emit AirdropClaimed(depositor, seniorCreditsOwned / 3, juniorCreditsOwned, upper * middle / lower);

        IERC20(IZivoeGlobals_ITO(GBL).zJTT()).safeTransfer(depositor, juniorCreditsOwned);
        IERC20(IZivoeGlobals_ITO(GBL).zSTT()).safeTransfer(depositor, seniorCreditsOwned / 3);

        if (upper * middle / lower > 0) {
            ITO_IZivoeRewardsVesting(IZivoeGlobals_ITO(GBL).vestZVE()).createVestingSchedule(
@>              depositor, 0, 360, upper * middle / lower, false
            );
        }


```

## Impact

There are a couple of impacts, firstly users are incentivized to call `ZivoeITO::claimAirdrop()` as quickly as possible to minimize the diluation of their airdrop. ITO participants who do not call `ZivoeITO::claimAirdrop()` before post-ITO liqudiity enters the Tranches will permanantly lose a portion of their allocation of vested $ZVE.
In addition, each $ZVE token worth of dilution will be reduced from the airdropped supply resulting in less than the indended 5% of $ZVE tokens allocated to ITO depositors.

## Code Snippet

POC - Paste the following test and helper function in `Test_ZivoeITO.sol` to demonstrate the issue.

The test shows `_ZVE_Vested_SAM` == `vestedZveAllocationIfClaimDelay` and `vestedZveAllocationIfClaimDelay` < `vestedZveAllocationIfClaimImmediatelyAfterITO` as a result of a post-ITO deposit into a tranche.

```javascript
    function test_audit_ZivoeITO_ITODepositorsDilutedByPostITODeposits() public {

        // commence ITO
        zvl.try_commence(address(ITO));

        // warp to after ITO started to enable ITO deposits
        hevm.warp(ITO.end() - 30 days + 1 seconds);

        // Sam deposits into senior tranche during ITO
        uint256 amount_senior = 100 ether;
        depositSenior(DAI, amount_senior);

        // Warp to end of ITO.
        hevm.warp(ITO.end() + 1 seconds);

        // migrate funds to unlock tranche deposits
        ITO.migrateDeposits();

        // Supplies in tranches immediately after ITO
        uint256 seniorSupp_ImmediatelyAfterITO = IERC20(zSTT).totalSupply();
        uint256 juniorSupp_ImmediatelyAfterITO = IERC20(zJTT).totalSupply();

        // If Sam claims airdrop now, his vested ZVE allocation will be the maximum
        uint256 vestedZveAllocationIfClaimImmediatelyAfterITO = helper_calculateAirdropZVE_Rewards(address(sam));

        // Sam does not claim airdrop immediately, and another user deposits into seniorTranche
        address newUser = makeAddr("newUser");
        uint256 newLiquidityAmount = 40 ether;
        mint("DAI", newUser, newLiquidityAmount);
        vm.prank(newUser);
        IERC20(DAI).approve(address(ZVT), newLiquidityAmount);
        vm.prank(newUser);
        ZVT.depositJunior(newLiquidityAmount, address(DAI));
        //ZVT.depositSenior(newLiquidityAmount, address(DAI)); // dilution is greater with senior deposits because they are weighted higher in ZivoeITO::claimAirdrop()::lower
        vm.stopPrank();

        // Supplies in tranches after new user deposits post-ITO
        uint256 seniorSupp_AfterNormalTrancheDeposit = IERC20(zSTT).totalSupply();
        uint256 juniorSupp_AfterNormalTrancheDeposit = IERC20(zJTT).totalSupply();

        // Supplies in tranches have increased since the ITO concluded
        assert(seniorSupp_AfterNormalTrancheDeposit + juniorSupp_AfterNormalTrancheDeposit > 
                seniorSupp_ImmediatelyAfterITO + juniorSupp_ImmediatelyAfterITO);

        // If Sam were to claim now, they would receive less ZVE than if they had claimed immediately after the ITO
        uint256 vestedZveAllocationIfClaimDelay = helper_calculateAirdropZVE_Rewards(address(sam));
        assert(vestedZveAllocationIfClaimDelay < vestedZveAllocationIfClaimImmediatelyAfterITO);

        // Sam now claims airdrop and gets less vested ZVE due to dilution post-ITO 
        (uint256 _zSTT_Claimed_SAM,, uint256 _ZVE_Vested_SAM) = sam.claimAirdrop(address(ITO), address(sam));
        assert(_ZVE_Vested_SAM == vestedZveAllocationIfClaimDelay);

        //console2.log("Percent of maximum ZVE received: ", BIPS * _ZVE_Vested_SAM / vestedZveAllocationIfClaimImmediatelyAfterITO);
    }

    function helper_calculateAirdropZVE_Rewards(address user) public returns (uint256 vestedZVE_Alocation) {

        uint256 seniorCreditsOwned = ITO.seniorCredits(address(user));
        uint256 juniorCreditsOwned = ITO.juniorCredits(address(user));

        // Calculate proportion of $ZVE awarded based on $pZVE credits.
        uint256 upper = seniorCreditsOwned + juniorCreditsOwned;                       
        uint256 middle = ZVE.totalSupply() / 20;                                       
        uint256 lower = zSTT.totalSupply() * 3 + zJTT.totalSupply(); 

        vestedZVE_Alocation = upper * middle / lower;
    }
```

## Tool used

Manual Review

## Recommendation

Snapshot the supply of `zJTT` and `zSTT` when `ZivoeITO::migrateDeposits()` is called, use these supplies to calculate vested $ZVE allocations.





# (rejected) [M] Incorrect computation of `M` in `ZivoeMath::ema()` function results in over-weighting base-value `bV` instead of current-value `cV`

## Summary

According to the docs (https://docs.zivoe.com/user-docs/yield-distribution/mathematics#ema), the `ZivoeMath::ema()` function "Conceptually we are exponentially weighting the current value relative to the prior value, to give greater weighting to the most recent value."

Due to the calculation of `M` in `ZivoeMath::ema()`, greater weight is given to the recent value (`cV`) for `N` <= 2 which equates to the first two times `ZivoeYDL::distributeYield()` is called. For the remainder of the life of the protocol (`N` > 2), greater weight will be given to the prior value (`bV`).

## Vulnerability Detail

In the `ZivoeMath::ema()`, `M` can take a range of values between 0 and WAD. When this value is 0, 100% weighting is given to the prior value and 0% weighting to the current value. When this value is WAD, 0% weighting is given to the prior value and 100% weighting to the current value.

As stated in the documentation, it is intended to give greater weighting to the current value, implying that `M` should be > 0.5*WAD for the majority of the life of the protocol. However, with the current calculation of `M`:

https://github.com/sherlock-audit/2024-03-zivoe-Nihavent/blob/a566fe2fd4529013234520cde9b00a2b2a17da45/zivoe-core-foundry/src/ZivoeMath.sol#L37

```javascript
  uint256 M = (WAD * 2).floorDiv(N + 1);
```
The protocol will give greater weighting to the prior value, as shown in the table below:

| N           | M           | Weight of current value Cv | Weight of prior value Pv |
| ----------- | ----------- | -------------------------- | ------------------------ |
| 1           | 1           | 100%                       | 0
| 2           | 0.667       | 66.7%                      | 33.3%
| 3           | 0.5         | 50%                        | 50%
| 4           | 0.4         | 40%                        | 60%
| 5           | 0.333       | 33.3%                      | 66.6%
| >=6         | 0.286       | 28.6%                      | 71.4%


## Impact

The `ZivoeYDL::emaSTT` and `ZivoeYDL::emaJTT` state variables are calculated incorrectly and as a result, each Tranche's EMA value is extremely slow to respond to changes in the size of their respective tranche.

Consider the example provided in the docs (https://docs.zivoe.com/user-docs/yield-distribution) with an indefinite continuation where the size of the Senior Tranche does not change after 24,000,000:

| Month | Adjust Supply of zSTT | Senior EMA |
|-------|------------------------|------------|
| 1     | 10,000,000             | 9,333,333  |
| 2     | 12,000,000             | 10,666,666 |
| 3     | 14,000,000             | 11,999,999 |
| 4     | 16,000,000             | 13,333,332 |
| 5     | 18,000,000             | 14,666,665 |
| 6     | 20,000,000             | 16,190,474 |
| 7     | 22,000,000             | 17,850,338 |
| 8     | 24,000,000             | 19,607,384 |
| 9     | 24,000,000             | 20,862,417 |
| 10    | 24,000,000             | 21,758,869 |
| 11    | 24,000,000             | 22,399,192 |
| 12    | 24,000,000             | 22,856,566 |
| 13    | 24,000,000             | 23,183,261 |
| 14    | 24,000,000             | 23,416,615 |
| 15    | 24,000,000             | 23,583,297 |
| 16    | 24,000,000             | 23,702,355 |
| 17    | 24,000,000             | 23,787,396 |
| 18    | 24,000,000             | 23,848,140 |
| 19    | 24,000,000             | 23,891,529 |
| 20    | 24,000,000             | 23,922,520 |

Etc..

Assuming adjusted supply of `zSTT` does not change, it will take ~49 months for Senior EMA to catch up to the tranche size!

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe-Nihavent/blob/a566fe2fd4529013234520cde9b00a2b2a17da45/zivoe-core-foundry/src/ZivoeMath.sol#L35-L39

```javascript

    function ema(uint256 bV, uint256 cV, uint256 N) external pure returns (uint256 eV) {
        assert(N != 0);
        uint256 M = (WAD * 2).floorDiv(N + 1);
        eV = ((M * cV) + (WAD - M) * bV).floorDiv(WAD);
    }

```

## Tool used

Manual Review

## Recommendation

Update the calculation of `M`, so in the limit, it stabalizes somewhere in the range 0.5*WAD < `M` < 1. The exact calculation of `M` used will be a commercial decision.



# (accepted) [H] Griefing attack possible in `ZivoeRewards::depositRewards()` by continuously extending `periodFinish`

## Summary

Lack of checks on who can call and how much value


## Vulnerability Detail

Any address can call `ZivoeRewards::depositRewards()` with any amount of `_rewardsToken`. Each time this function is called, `rewardData[_rewardsToken].periodFinish` is set to `block.timestamp` + `rewardData[_rewardsToken].rewardsDuration`, and a new `rewardData[_rewardsToken].rewardRate` is calculated.
If `ZivoeRewards::depositRewards()` is called before the previous `rewardData[_rewardsToken].periodFinish` has passed for some `_rewardsToken`, then the previously allocated rewards will be distributed slower than intended (evenly over a full new `rewardsDuration`).

https://github.com/sherlock-audit/2024-03-zivoe-Nihavent/blob/a566fe2fd4529013234520cde9b00a2b2a17da45/zivoe-core-foundry/src/ZivoeRewards.sol#L225-L243

```javascript
    /// @notice Deposits a reward to this contract for distribution.
    /// @param _rewardsToken The asset that's being distributed.
    /// @param reward The amount of the _rewardsToken to deposit.
    function depositReward(address _rewardsToken, uint256 reward) external updateReward(address(0)) nonReentrant {
        IERC20(_rewardsToken).safeTransferFrom(_msgSender(), address(this), reward);

        // Update vesting accounting for reward (if existing rewards being distributed, increase proportionally).
        if (block.timestamp >= rewardData[_rewardsToken].periodFinish) {
            rewardData[_rewardsToken].rewardRate = reward.div(rewardData[_rewardsToken].rewardsDuration);
        } else {
            uint256 remaining = rewardData[_rewardsToken].periodFinish.sub(block.timestamp);
            uint256 leftover = remaining.mul(rewardData[_rewardsToken].rewardRate);
@>          rewardData[_rewardsToken].rewardRate = reward.add(leftover).div(rewardData[_rewardsToken].rewardsDuration); // new rewardRate calculated
        }

        rewardData[_rewardsToken].lastUpdateTime = block.timestamp;
@>      rewardData[_rewardsToken].periodFinish = block.timestamp.add(rewardData[_rewardsToken].rewardsDuration); // periodFinish extended each time this function is called
        emit RewardDeposited(_rewardsToken, reward, _msgSender());
    }


```

This allows griefing attacks where users can maliciously call `ZivoeRewards::depositRewards()` for any `_rewardsToken` any number of times, on any instance of this contract (`stSTT`, `stJTT`, `stZVE`, or `vestZVE`) to perpetually delay users' staking rewards.

Also note that delayed staking rewards will likely happen accidently with any user able to call `OCE_ZVE::forwardEmissions()` which subsequently makes calls to `ZivoeRewards::depositRewards()` with `ZVE` as the `_rewardsToken`.

## Impact

The table below shows the percentage of staking rewards a user will be able to claim after the full `rewardsDuration` has elapsed with a malicious user repeatedly calling `ZivoeRewards::depositRewards()`. 

Note this was modelled for simplicitly, but malicious calls to `ZivoeRewards::depositRewards()` do not need to be at consistent intervals.

| Malicious Calls to depositRewards()     | Staker Rewards as a % of their expected rewards |
|-----------------------------------------|-------------|
| None                       | 99.99%      |
| extra deposit after 15 days             | 74.99%      |
| 2 extra deposits, each after 10 days    | 70.37%      |
| 5 extra deposits, each after 6 days     | 67.23%      |
| deposit every single day                | 63.83%      |
| deposit every single hour               | 63.23%      |
| deposit every single minute             | 63.21%      |


## Code Snippet

POC - Paste this test into `Test_ZivoeRewards.sol`. The test shows a single extra call to `ZivoeRewards::depositRewards()` resulting in a staker being only able to claim 75% of their expected rewards after the full `rewardsDuration` has elapsed.

```javascript
    function test_Audit_ExtraDeposit_Scenario1() public {
        uint256 stakeAmount = 1000e18;
        uint256 rewardAmount = 100e18;
        uint256 startTs = block.timestamp;

        // 1. Jim stakes their 1k tokens
        assert(jim.try_approveToken(address(ZVE), address(stZVE), stakeAmount));
        jim.try_stake(address(stZVE), stakeAmount);

        // 2. Bob deposits DAI into stZVE, triggering a 30 day unlock period for stakers
        depositReward_DAI(address(stZVE), rewardAmount);

        // 3. Extra deposit(s) occur
        hevm.warp(block.timestamp + 15 * 86400);
        depositReward_DAI(address(stZVE), 1);
        hevm.warp(block.timestamp + 15 * 86400);
        
        // 4. 30 days has passed since initial reward was allocated, Jim calls getRewards()
        assert(block.timestamp == startTs + 30 * 86400);
        uint256 jimPreClaimBalance = IERC20(DAI).balanceOf(address(jim));
        assert(jim.try_getRewards(address(stZVE)));
        uint256 jimPostClaimBalance = IERC20(DAI).balanceOf(address(jim));
        assert(jimPostClaimBalance - jimPreClaimBalance < rewardAmount); // Jim would reasonably expect to claim close to 100e18 being the only staker, but can only claim ~ 75e18 due to the periodFinish being extended. 
        //console2.log("Percent claim: ", BIPS * (jimPostClaimBalance - jimPreClaimBalance) / rewardAmount);
    }
```

## Tool used

Manual Review

## Recommendation

Some mitigigations are:
1. Increase cost the the attack by asserting `ZivoeRewards::depositRewards()` must be called with a minimum `amount` of `rewardData[_rewardsToken].rewardsDuration`. After all when `amount` < `rewardData[_rewardsToken].rewardsDuration`, then `rewardData[_rewardsToken].rewardRate` truncates to 0 anyway, so no rewards are unlocked by stakers.
2. Access controls on who can call `ZivoeRewards::depositRewards()`, limit access to key yield distributing contracts such as `ZivoeYDL` and `OCE_ZVE`.


# (accepted) [M] Current tranche sizes are not considered when calculating tranche target yield in `ZivoeYDL::distributeYield()` resulting in tranche yield being paid based on outdated supply values.

## Summary

In the `ZivoeYDL::distributeYield()` function, EMA state variables `ZivoeYDL::emaSTT` and `ZivoeYDL::emaJTT` are updated after `ZivoeYDL::earningsTrancheuse()` is called, this results in yield targets calculated off 'old' tranche supply values.

Current adjusted supply values `ZivoeYDL::distributeYield::aJTT` and `ZivoeYDL::distributeYield::aSTT` will not be used in the EMA calculation until next month's call of `ZivoeYDL::distributeYield()`. Due to it being possible for these adjusted values to change significantly month-to-month, yield may be distributed to the tranches in unexpected ways.

## Vulnerability Detail

Assuming `ZivoeYDL::distributeYield()` is called every 30-days, the yield targets will be calculated on the previous two of `ZivoeGlobals::adjustedSupplies()`, ie. adjusted supplies from 60-days ago and 30-days ago will be used in the current months's yield distribution.

## Impact

Consider the following scenario where the protocol is in month N of operation.

In month N, `ZivoeGlobals::defaults` is small relative to `zJTT.totalSupply()` and therefore the adjusted supply of the Junior Tranche `ZivoeYDL::distributeYield::aJTT` is significantly greater than 0.

In month N+1, the protocol sustains significant defaults that exceed the size of the Junior Tranche. The value of `ZivoeYDL::distributeYield::aJTT` becomes 0.

In month N+2, the defaults remain, and again `ZivoeYDL::distributeYield::aJTT` is 0. In this month when `ZivoeYDL::distributeYield()` is called, the target yield in `ZivoeYDL::earningsTrancheuse()` utilises an EMA calculation based on months N and N+1, some yield is paid to the junior tranche assuming the total yield payable exceeded the protocol yield target.

In month N+3, defaults are resolved, the Junior Tranhce is now healthy with `ZivoeYDL::distributeYield::aJTT` being greater than 0. When `ZivoeYDL::distributeYield()` is called, the target yield for the Junior Tranche will be 0 because the EMA calculation is based on month N+1 and N+2, in which defaults exceeded the size of the Junior tranche. Even if the protocol earnings exceeded the protocol yield target, no yield will be paid to the healthy Junior Tranche stakers.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe-Nihavent/blob/a566fe2fd4529013234520cde9b00a2b2a17da45/zivoe-core-foundry/src/ZivoeYDL.sol#L229-L239

```javascript
        // Calculate yield distribution (trancheuse = "slicer" in French).
        (
            uint256[] memory _protocol, uint256 _seniorTranche, uint256 _juniorTranche, uint256[] memory _residual
        ) = earningsTrancheuse(protocolEarnings, postFeeYield); 

        emit YieldDistributed(_protocol, _seniorTranche, _juniorTranche, _residual);
        
        // Update ema-based supply values.
        (uint256 aSTT, uint256 aJTT) = IZivoeGlobals_YDL(GBL).adjustedSupplies();
        emaSTT = MATH.ema(emaSTT, aSTT, retrospectiveDistributions.min(distributionCounter)); // Updates after the values were used. These new EMA-calculated values will be used next month
        emaJTT = MATH.ema(emaJTT, aJTT, retrospectiveDistributions.min(distributionCounter)); // Updates after the values were used. These new EMA-calculated values will be used next month
```

## Tool used

Manual Review

## Recommendation

Calculate `emaSTT` and `emaJTT` before calling `ZivoeYDL::earningsTrancheuse`.

```diff

+        // Update ema-based supply values.
+        (uint256 aSTT, uint256 aJTT) = IZivoeGlobals_YDL(GBL).adjustedSupplies();
+        emaSTT = MATH.ema(emaSTT, aSTT, retrospectiveDistributions.min(distributionCounter)); 
+        emaJTT = MATH.ema(emaJTT, aJTT, retrospectiveDistributions.min(distributionCounter));

        // Calculate yield distribution (trancheuse = "slicer" in French).
        (
            uint256[] memory _protocol, uint256 _seniorTranche, uint256 _juniorTranche, uint256[] memory _residual
        ) = earningsTrancheuse(protocolEarnings, postFeeYield); 

        emit YieldDistributed(_protocol, _seniorTranche, _juniorTranche, _residual);
        
-        // Update ema-based supply values.
-        (uint256 aSTT, uint256 aJTT) = IZivoeGlobals_YDL(GBL).adjustedSupplies();
-        emaSTT = MATH.ema(emaSTT, aSTT, retrospectiveDistributions.min(distributionCounter)); 
-        emaJTT = MATH.ema(emaJTT, aJTT, retrospectiveDistributions.min(distributionCounter));
```
