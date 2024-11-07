### Template

## Summary

## Vulnerability Detail

## Impact

## Code Snippet

## Tool used

Manual Review

## Recommendation





### High

### If the last `pool.totalDepositAssets` are rebalanced in a Base Pool, the next depositor will lose a portion of their deposit

## Summary

If positions liquidated through the `liquidateBadDebt()` flow rebalance the last `pool.totalDepositAssets` in a pool, then a problematic pool state is created. The next depositor is guaranteed to instantly lose a portion of their deposit. The bad debt socialized through the `liquidateBadDebt()` is essentially also socialized to the next depositor who was not part of the pool when the bad debt was incurred. 


## Vulnerability Detail

When a 'position' becomes undercollateralized, the protocol admin can liquidate this bad debt via a call to [`PositionManager::liquidateBadDebt()`](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/PositionManager.sol#L446-L464) which in turn calls `Pool::rebalanceBadDebt()`. As shown below, the socialization of liquidated bad debt is executed by reducing the number of depositAssets in a base pool without reducing the number of depositShares in a base pool:

```javascript
    function rebalanceBadDebt(uint256 poolId, address position) external {
        ... SKIP!...

        // compute pool and position debt in shares and assets
        uint256 totalBorrowShares = pool.totalBorrowShares;
        uint256 totalBorrowAssets = pool.totalBorrowAssets;
        uint256 borrowShares = borrowSharesOf[poolId][position];
        // [ROUND] round up against lenders
        uint256 borrowAssets = _convertToAssets(borrowShares, totalBorrowAssets, totalBorrowShares, Math.Rounding.Up);

        // rebalance bad debt across lenders
        pool.totalBorrowShares = totalBorrowShares - borrowShares;
        // handle borrowAssets being rounded up to be greater than totalBorrowAssets
        pool.totalBorrowAssets = (totalBorrowAssets > borrowAssets) ? totalBorrowAssets - borrowAssets : 0;
        uint256 totalDepositAssets = pool.totalDepositAssets;
@>      pool.totalDepositAssets = (totalDepositAssets > borrowAssets) ? totalDepositAssets - borrowAssets : 0;
        borrowSharesOf[poolId][position] = 0;
    }
```


Usually this works as the remaining shares are correctly devalued, however when it's the last liquidity in the pool it creates a problematic pool state. The pool will have 0 `totalDepositAssets` and nonzero `totalDepositShares`. Due to having 0 `totalDepositAssets` the next depositor will mint deposit shares at a 1:1 ratio as shown below:

```javascript
    function deposit(uint256 poolId, uint256 assets, address receiver) public returns (uint256 shares) {
        ... SKIP!...

@>      shares = _convertToShares(assets, pool.totalDepositAssets, pool.totalDepositShares, Math.Rounding.Down);
        
        ... SKIP!...
    }

    function _convertToShares(
        uint256 assets,
        uint256 totalAssets,
        uint256 totalShares,
        Math.Rounding rounding
    ) internal pure returns (uint256 shares) {
@>      if (totalAssets == 0) return assets;
        shares = assets.mulDiv(totalShares, totalAssets, rounding);
    }

```

As soon as these shares are minted, they're worth less than the deposit value due to the latent shares in the pool with no corresponding assets. The user who deposited will not be able to withdraw their full deposit.



The preconditions for this issue are:
- A base pool having 100% utilization
- Decline in collateral value (this need not be a large decline for assets with high LTV)
- All positions on that pool liquidated through the `liquidateBadDebt()` flow

I believe this is high severity because it will cause definite loss of funds to the next depositor, without extensive limitations due to the folowing design principles:
1. Pools can be created permissionlessly so liquidity is fragmented, it wouldn't be uncommon for an entire pool's liquidity to be loaned to a single position.
2. Excess deposits in a pool would likely be removed and re-allocated to ensure the utilization of a pool is high which results in higher interest rates for depositors (under linear and kinked IRM models). 
3. The maximum LTV is 98% which provides little room for a standard liquidation, ie. a single price update where collateral declines ~3% could take a position from a healthy state to an undercollateralized state.
4. Even when multiple positions have loans from a pool:
   - the risk parameters are shared as LTV is set by the pool, and
   - due to a whitelisting of collateral assets, if the market declines, it's likely multiple positions will be undercollateralized at the same time


## Impact

- The next depositor into this pool lose funds proportional to the size of their deposit relative to the amount of remaining deposit shares in the pool before their deposit, ie. as shown in the coded POC below, if a pool has 0 assets and 1e18 shares, the next deposit of 1e18 assets causes the depositor to lose 50% of their funds.
- Bad debt which gets rebalanced is socialised to future depositors, even though they did not have a deposit in the pool when the bad debt was liquidated.
- The above losses can extend to SuperPool depositors if the pool is subsequently added to a SuperPool deposit queue (loss will be among all SuperPool share holders).
- Trying to 'fix' the pool by depositing a small value such as 1 wei could have unintended consequences elsewhere in the system, such as rounding. For example if a pool has 0 assets, and 1e18 shares, and a user deposits 1 wei minting 1 share, the next deposit of 1e18 assets will mint > 1e36 shares. 


## Code Snippet

https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L528-L549
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/PositionManager.sol#L446-L464
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L309-L331
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L275-L283


## POC

Paste the below coded POC into LiquidateTest.t.sol. 

It shows the following scenario:
1. User deposits into pool
2. User2 opens position takes debt from pool equal to available liquidity
3. Position becomes undercollateralized due to decline in collateral price
4. Protocol liquidates bad debt
5. Original user has shares in pool but no assets exist, this is by design because pool depositors are exposed to the risk of bad debt
6. User3 deposits into pool and instantly loses funds

```javascript
    function test_POC_LastPoolLiquidityRebalanced() public {
        // 0. Setup pool -> we're using LinearRatePool as there is no existing deposits in setup.
        vm.startPrank(poolOwner); 
        riskEngine.requestLtvUpdate(linearRatePool, address(asset2), 0.9e18); 
        riskEngine.acceptLtvUpdate(linearRatePool, address(asset2)); // Need to set LTV so asset2 is accepted as collateral
        vm.stopPrank();

        // 1. User deposits into pool
        vm.startPrank(user);
        asset1.mint(user, 1e18);
        asset1.approve(address(pool), 1e18);
        pool.deposit(linearRatePool, 1e18, user);
        vm.stopPrank();

        // 2. Position takes debt from pool equal to available liquidity
        vm.startPrank(user2);
        asset2.mint(user2, 2e18);
        asset2.approve(address(positionManager), 1.5e18);
        
        Action[] memory actions = new Action[](4);
        (position, actions[0]) = newPosition(user2, bytes32(uint256(0x123456789)));
        actions[1] = deposit(address(asset2), 1.5e18);
        actions[2] = addToken(address(asset2));
        actions[3] = borrow(linearRatePool, 1e18);
        positionManager.processBatch(position, actions);
        assertTrue(riskEngine.isPositionHealthy(position));
        vm.stopPrank();

        // 3. Position becomes undercollateralized due to decline in collateral price
        FixedPriceOracle pointOneEthOracle = new FixedPriceOracle(0.5e18);
        vm.prank(protocolOwner);
        riskEngine.setOracle(address(asset2), address(pointOneEthOracle));
        riskEngine.validateBadDebt(position);
        vm.stopPrank();

        // 4. Protocol liquidates bad debt
        vm.startPrank(address(protocolOwner));
        positionManager.liquidateBadDebt(position);
        vm.stopPrank();

        // 5. User1 has shares in pool but no assets exist, this is by design because pool depositors are exposed to the risk of bad debt
        (,,,,,,,,, uint256 totalDepositAssets, uint256 totalDepositShares) = pool.poolDataFor(linearRatePool);
        uint256 user1Shares = pool.balanceOf(user, linearRatePool);
        assertEq(totalDepositAssets, 0);
        assertEq(totalDepositShares, 1e18); 
        assertEq(user1Shares, 1e18);

        // 6. User3 deposits into pool and instantly loses funds
        address victim;
        uint256 victimDeposit = 1e18;
        vm.startPrank(victim);
        asset1.mint(victim, victimDeposit);
        asset1.approve(address(pool), victimDeposit);
        pool.deposit(linearRatePool, victimDeposit, victim);
        vm.stopPrank();

        assertEq(pool.getAssetsOf(linearRatePool, victim), victimDeposit / 2); // Victim deposited 1e18 and instantly lost half the funds
    
    }
```

## Tool used

Manual Review

## Recommendation

The following recommended fix assumes the protocol wishes to retain the design where all assets on a bad debt position are sent to the protocol multisig/goverance wallet.

In the `rebalanceBadDebt()` function, implement to check if the last deposit in the pool was liquidated. If so, transfer the pool ownership to protocol admin and pause the pool.

```diff
    function setPoolOwner(uint256 poolId, address newOwner) external {
-       if (msg.sender != ownerOf[poolId]) revert Pool_OnlyPoolOwner(poolId, msg.sender);
+       if (msg.sender != ownerOf[poolId] && msg.sender != owner()) revert Pool_OnlyPoolOwner(poolId, msg.sender); // Allow the protocol owner to set the owner of a pool
        // address(0) cannot own pools since it is used to denote uninitalized pools
        if (newOwner == address(0)) revert Pool_ZeroAddressOwner();
        ownerOf[poolId] = newOwner;
        emit PoolOwnerSet(poolId, newOwner);
    }


    function rebalanceBadDebt(uint256 poolId, address position) external {
        PoolData storage pool = poolDataFor[poolId];
        accrue(pool, poolId);


        // revert if the caller is not the position manager
        if (msg.sender != positionManager) revert Pool_OnlyPositionManager(poolId, msg.sender);


        // compute pool and position debt in shares and assets
        uint256 totalBorrowShares = pool.totalBorrowShares;
        uint256 totalBorrowAssets = pool.totalBorrowAssets;
        uint256 borrowShares = borrowSharesOf[poolId][position];
        // [ROUND] round up against lenders
        uint256 borrowAssets = _convertToAssets(borrowShares, totalBorrowAssets, totalBorrowShares, Math.Rounding.Up);


        // rebalance bad debt across lenders
        pool.totalBorrowShares = totalBorrowShares - borrowShares;
        // handle borrowAssets being rounded up to be greater than totalBorrowAssets
        pool.totalBorrowAssets = (totalBorrowAssets > borrowAssets) ? totalBorrowAssets - borrowAssets : 0;
        uint256 totalDepositAssets = pool.totalDepositAssets;
-       pool.totalDepositAssets = (totalDepositAssets > borrowAssets) ? totalDepositAssets - borrowAssets : 0;
+       if (totalDepositAssets - borrowAssets > 0) {
+           pool.totalDepositAssets = totalDepositAssets - borrowAssets;
+       }
+       else {
+           pool.totalDepositAssets = 0;
+           setPoolOwner(poolId, msg.sender); // Assumes the owner of PositionManager == owner of Pool == trusted admin
+           togglePause(poolId); // Pause the pool to protect future depositors
+       }
        borrowSharesOf[poolId][position] = 0;
    }
```

- Alternatively an admin can deposit a small amount to the pool, however there may be downstream precision issues if `totalDepositShares` >> `totalDepositAssets`




### Medium



### Liquidators may repay a position's debt to pools that are within their risk tolerance, breaking the concept of isolated risk in base pools

## Summary

The trust model of the protocol is such that depositors to pools must trust the pool owners, however there was no documented trust assumption between base pool owners. Creating a base pool is permissionless so the owner of pool A shouldn't be able to do something that adversely affects pool B.

However, liquidations that affect Pool B can be caused by Pool A's risk settings despite the position being within Pool B's risk tolerance, which means base pools do not have isolated risk and there is a trust assumption between base pool owners.

According to the [Sentiment Docs](https://docs.sentiment.xyz/concepts/core-concepts/isolated-pools#base-pools), one of the core concepts is isolated financial activities and risk:

>"Each Base Pool operates independently, ensuring the isolation of financial activities and risk."

But with the current design, the LTVs set by a base pool impacts the likelihood of liquidations in every other base pool which shares a common position via loans.


## Vulnerability Detail

A position with debt and recognized assets is determined to be healthy if the recognized collateral exceeds the `minReqAssetValue`:

```javascript

    function isPositionHealthy(address position) public view returns (bool) {
        // a position can have four states:
        // 1. (zero debt, zero assets) -> healthy
        // 2. (zero debt, non-zero assets) -> healthy
        // 3. (non-zero debt, zero assets) -> unhealthy
        // 4. (non-zero assets, non-zero debt) -> determined by weighted ltv

        ... SKIP!...

@>      uint256 minReqAssetValue =
            _getMinReqAssetValue(debtPools, debtValueForPool, positionAssets, positionAssetWeight, position);
        return totalAssetValue >= minReqAssetValue; // (non-zero debt, non-zero assets)
    }
```

`_getMinReqAssetValue` is the sum of required asset value across all collateral tokens and debt positions, adjusted for: the weight of each collateral token, magnitude of debt from a given pool, and the ltv setting for that asset set by that pool:

```javascript
    function _getMinReqAssetValue(
        uint256[] memory debtPools,
        uint256[] memory debtValuleForPool,
        address[] memory positionAssets,
        uint256[] memory wt,
        address position
    ) internal view returns (uint256) {
        uint256 minReqAssetValue;

        ... SKIP!...

        uint256 debtPoolsLength = debtPools.length;
        uint256 positionAssetsLength = positionAssets.length;
        for (uint256 i; i < debtPoolsLength; ++i) {
            for (uint256 j; j < positionAssetsLength; ++j) {
                uint256 ltv = riskEngine.ltvFor(debtPools[i], positionAssets[j]);
                ... SKIP!...
@>              minReqAssetValue += debtValuleForPool[i].mulDiv(wt[j], ltv, Math.Rounding.Up);
            }
        }
        ... SKIP!...
        return minReqAssetValue;
    }
```

Note from above, that a position is either healthy or unhealthy across all debtPools and assets held by the position. There is no allocation of collateral to a debt position with respect to it's risk parameters. This means that the risk settings of one pool can directly impact the ability to liquidate a position in another pool.

Also note, in the [liquidation flow](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/PositionManager.sol#L430-L444), liquidators are [free to chose which assets they seize and which debt they repay](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/PositionManager.sol#L432-L433), as long as the position returns to a [healthy state after the liquidation](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/PositionManager.sol#L442). This means that debt from a pool may be repaid even though the position was within the risk parameters of that pool.

Base pool owners are able to set LTV for assets individually through a [request /](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/RiskEngine.sol#L167-L187) [accept](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/RiskEngine.sol#L190-L210) pattern. 
But as shown, LTVs set by base pools do not strictly impact the risk in their pool, but all pools for which a single position has debt in. 

There is a [timelock delay](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/RiskEngine.sol#L182) on proposed changes to LTV, and here is why this doesn't completely mitigates this issue:
1. The issue doesn't not require a change in LTV in any pool for a pool to be exposed to the risk settings of another pool via a liquidated position (that is just the adversarial-pool attack path).
2. There is no limit on how many different positions a pool will loan to at any given time (call this numPools). Each position a pool loans to can have debts in [up to 4 other pools](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Position.sol#L25). So even though a [`TIMELOCK_DURATION`](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/RiskEngine.sol#L20) of 24 hours is implemented, it may not be practical for pools to monitor the proposed LTV changes for up to numPools^4 (numPools is uncapped).
3. An adversarial pool could propose an LTV setting, then all other impacted pools may notice this and respond by adjusting their own LTVs to ensure their `totalBorrowAssets` is minimally impacted, then the adverserial pool may not even accept the proposed setting. Even if the setting is accepted there will be a window between when the first pool is allowed to update the settings and when other pools are able to, in which liquidations can occur.


## Impact

- Pool A's risk settings can cause liquidations in Pool B, despite the debt position being within the risk tolerance of Pool B.
- The liquidation of Pool B would decrease borrow volume and utilization which decreases earnings for all depositors (both through volume and rate in the linear and kinked IRM models).
- This may occur naturally, or through adversarial pools intentionally adjusting the LTV of assets to cause liquidations. In fact they may do this to manipulate the utilization or TVL in other pools, or to liquidate more positions themselves and claim the liquidation incentives.


## Code Snippet

https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/RiskModule.sol#L67-L85
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/RiskModule.sol#L250-L278


## POC

Paste the below coded POC into LiquidateTest.t.sol. 

Simply put, it shows a single debt position on a pool get liquidated when the collateral price did not change and the pool owner did not change LTV settings. The sole cause of the liquidation was another pool changing their LTV settings.

Step by step:
1. user deposits into base fixedRatePool and linearRatePool which both accept asset1. Both pools accept asset2 as collateral.
   - fixedRatePool has an LTV for asset2 of 70% (ie. minReqCollateral = debt / .7)
   - linearRatePool has an LTV for asset2 of 70% (ie. minReqCollateral = debt / .7)
2. user2 opens a position and deposits 3e18 asset2 as collateral
3. user2 borrows from both pools and has a healthy position:
   - user2 borrows 1e18 from fixedRatePool and 1e18 from linearRatePool
   - minReqCollateral = (1e18 * 1e18 / 0.7e18) + (1e18 * 1e18 / 0.7e18) = 1.428571e18 + 1.428571e18 = 2.857142e18
4. fixedRatePool decides to decrease the LTV setting for asset2 to 60%
5. Position is no longer health because minReqCollateral = (1e18 * 1e18 / 0.6e18) + (1e18 * 1e18 / 0.7e18) = 1.666e18 + 1.428571e18 = 3.094571e18
6. A liquidator, which could be controlled by the owner of fixedRatePool then liquidates the position which has become unhealthy by repaying the debt from linearRatePool, thus impacting the utilization and interest rate of linearRatePool, despite the collateral price not changing and the owner of linearRatePool not adjusting it's LTV settings.


```javascript
    function test_AuditBasePoolsShareRisk() public {

        // Pool risk settings
        vm.startPrank(poolOwner);
        riskEngine.requestLtvUpdate(fixedRatePool, address(asset2), 0.7e18); 
        riskEngine.requestLtvUpdate(linearRatePool, address(asset2), 0.7e18); 
        vm.warp(block.timestamp +  24 * 60 * 60); // warp to satisfy timelock
        riskEngine.acceptLtvUpdate(fixedRatePool, address(asset2));
        riskEngine.acceptLtvUpdate(linearRatePool, address(asset2));
        vm.stopPrank();

        // 1. user deposits into base fixedRatePool and linearRatePool which both accept asset1. Both pools accept asset2 as collateral.
        vm.startPrank(user);
        asset1.mint(user, 20e18);
        asset1.approve(address(pool), 20e18);
        pool.deposit(fixedRatePool, 10e18, user);
        pool.deposit(linearRatePool, 10e18, user);
        vm.stopPrank();

        // 2. user2 opens a position and deposits 3e18 asset2 as collateral
        vm.startPrank(user2);
        asset2.mint(user2, 3e18);
        asset2.approve(address(positionManager), 3e18); // 3e18 asset2
        
        Action[] memory actions = new Action[](5);
        (position, actions[0]) = newPosition(user2, bytes32(uint256(0x123456789)));
        actions[1] = deposit(address(asset2), 3e18);
        actions[2] = addToken(address(asset2));

        // 3. user2 borrows from both pools and has a healthy position:
        actions[3] = borrow(fixedRatePool, 1e18);
        actions[4] = borrow(linearRatePool, 1e18);
        positionManager.processBatch(position, actions);
        assertTrue(riskEngine.isPositionHealthy(position));
        vm.stopPrank();


        // 4. fixedRatePool decides to decrease the LTV setting for asset2 to 60%
        vm.startPrank(poolOwner);
        riskEngine.requestLtvUpdate(fixedRatePool, address(asset2), 0.6e18); 
        vm.warp(block.timestamp + 24 * 60 * 60); // warp to satisfy timelock
        riskEngine.acceptLtvUpdate(fixedRatePool, address(asset2));
        vm.stopPrank();

        // 5. Position is no longer health because minReqCollateral = (1e18 * 1e18 / 0.6e18) + (1e18 * 1e18 / 0.7e18) = 1.666e18 + 1.428571e18 = 3.094571e18
        assertTrue(!riskEngine.isPositionHealthy(position));

        // 6. A liquidator, which could be controlled by the owner of fixedRatePool then liquidates the position which has become unhealthy by repaying the debt from linearRatePool, thus impacting the utilization and interest rate of linearRatePool, despite the collateral price not changing and the owner of linearRatePool not adjusting it's LTV settings.
        DebtData[] memory debts = new DebtData[](1);
        DebtData memory debtData = DebtData({ poolId: linearRatePool, amt: type(uint256).max });
        debts[0] = debtData;

        AssetData memory asset1Data = AssetData({ asset: address(asset2), amt: 1.25e18 });
        AssetData[] memory assets = new AssetData[](1);
        assets[0] = asset1Data;

        vm.startPrank(liquidator);
        asset1.mint(liquidator, 2e18);
        asset1.approve(address(positionManager), 2e18);
        positionManager.liquidate(position, debts, assets);
        vm.stopPrank();
    }
```

## Tool used

Manual Review

## Recommendation

- To maintain the concept of isolated financial risk in base pools, a position's health can be considered at the 'position level', but in the liquidation flow, collateral could be weighted to pools based on the level of debt in each pool.
  
- For example, taking the example from the POC above, after fixedRatePool changed the LTV setting from 70% to 60%, the position became unhealthy as the minReqAssetValue of 3.094571e18 exceeded the deposited collateral worth 3e18. 
- The minReqCollateral was calculated in each iteration of the loop in `RiskModule::_getMinReqAssetValue()`, and we saw in the POC that the contribution required from linearRatePool was 1.4285e18 and the contribution required from fixedRatePool was 1.666e18.
- If we apportion the deposited collateral based on the size of debt in each pool we would apportion 1.5e18 value of collateral to each debt (because the value of each debt was equal), this would show:
  - The position is within linearRatePool's risk tolerance because 1.5e18 > 1.4285e18
  - The position is not within fixedRatePool's risk tolerance because 1.5e18 < 1.666e18
- So I recommend we allow the liquidation of debt from fixedRatePool but not linearRatePool. This makes sense as fixedRatePool was the pool who opted for a riskier LTV.
- This solution is consistent with the idea of isolated pool risk settings and a trustless model between the owners of base pools




### The PUSH0 opcode is not supported on all EVM-compatable networks, causing deployment reverts

## Summary

The contest [readme](https://audits.sherlock.xyz/contests/349) mentions that the codebase should be able to be deployed to any EVM network. With the current codebase, some deployments will revert due to Solidity version 0.8.24 generating the PUSH0 opcode.

> "On what chains are the smart contracts going to be deployed? Any EVM-compatbile network"


## Vulnerability Detail

Not all EVM-compatable networks support all opcodes. ZkSync, Linea, and Polygon ZkEvm do not currently support the PUSH0 opcode resulting in reverts upon deployment.

## Impact

Deployment will revert

## Code Snippet

https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L2

## Tool used

Manual Review

## Recommendation

Before depploying the codebase to a network, check opcode compatability. When PUSH0 is not supported, compile the code using an earlier version where PUSH0 opcodes are not generated.





### `PositionManager::liquidateBadDebt()` will be frontrun by pool depositors to avoid losing their deposited assets, it can also be backrun to mint more shares than they originally held

## Summary

The `PositionManager::liquidateBadDebt()` function socializes the bad debt to the current pool depositors of the pool the loan was taken from. Depositors can gain financial advantage by frontrunning this transaction and withdrawing as much liquidity as possible before  `liquidateBadDebt()` is executed.

## Vulnerability Detail

On Ethereum mainnet and in the future when L2s have decentralized sequencers, depositors are incentivized to obseve the mempool and frontrun the `PositionManager::liquidateBadDebt()` function with a `Pool::withdraw()` call. This is because `liquidateBadDebt()` socializes the bad debt to pool depositors by reducing the number of depositAssets in a base pool without reducing the number of depositShares in a base pool:

```javascript
    function rebalanceBadDebt(uint256 poolId, address position) external {
        ... SKIP!...

        // compute pool and position debt in shares and assets
        uint256 totalBorrowShares = pool.totalBorrowShares;
        uint256 totalBorrowAssets = pool.totalBorrowAssets;
        uint256 borrowShares = borrowSharesOf[poolId][position];
        // [ROUND] round up against lenders
        uint256 borrowAssets = _convertToAssets(borrowShares, totalBorrowAssets, totalBorrowShares, Math.Rounding.Up);

        // rebalance bad debt across lenders
        pool.totalBorrowShares = totalBorrowShares - borrowShares;
        // handle borrowAssets being rounded up to be greater than totalBorrowAssets
        pool.totalBorrowAssets = (totalBorrowAssets > borrowAssets) ? totalBorrowAssets - borrowAssets : 0;
        uint256 totalDepositAssets = pool.totalDepositAssets;
@>      pool.totalDepositAssets = (totalDepositAssets > borrowAssets) ? totalDepositAssets - borrowAssets : 0;
        borrowSharesOf[poolId][position] = 0;
    }
```

Example:
- Lender A and lender B each have 2e18 worth of deposited assets in Pool A, they each hold 2e18 shares
- A single borrow position exists with a borrow value of 1e18
- The position becomes bad debt due to collateral price decline and gets liquidated via `liquidateBadDebt`
- Lender B frontRuns the `liquidateBadDebt` transaction with a `Pool::withdraw()` transaction and removes 2e18 worth of liquidity
- `liquidateBadDebt` is executed and lender A bears the cost of the debt
- There is now 1 pool depositor with 1e18 worth of deposited assets and 2e18 deposit shares.
- (optionally) lender B can backrun the transaction with a `Pool::Deposit()` transaction to mint = 2e18 * 2e18 / 1e18 = 4e18 shares.



## Impact

- Users who execute this attack prevent themselves from paying the cost of bad debt incurred by the pool they posited into. By dooing so, they pass these costs onto other other pool depositors.

## Code Snippet

https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L528-L549

## Tool used

Manual Review

## Recommendation

- Change design to swap collateral for the pool asset and repay some of the debt to minimize the impact on pool depositors. If `liquidateBadDebt()` is called quickly after a pool becomes undercollateralized, then losses to pool depositors would be minimal and therefore little incentive to frontrun the call.
- If the design is fixed, then `liquidateBadDebt()` transaction should be sent through [Flashbots RPC](https://docs.flashbots.net/flashbots-protect/overview) or other MEV protection.



### Approve function not supported for USDT on some networks, resulting in inability to deploy a SuperPool with USDT as the asset

## Summary

USDT not supported due to having no return value on mainnet. This will cause EVM reverts due to solidity's return data length check.

## Vulnerability Detail

The contest [readme](https://audits.sherlock.xyz/contests/349) indicates that the intention is to deploy the protocol on any EVM-compatible network:

>"Q: On what chains are the smart contracts going to be deployed? A: Any EVM-compatbile network"

In addition, it is expected that USDT is supported:

>"Protocol governance will ensure that oracles are only set for standard ERC-20 tokens (plus USDC/USDT)"

The issue occurs in [`SuperPoolFactory::deploySuperPool()`](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/SuperPoolFactory.sol#L73), and in [`SuperPool::reallocate()`](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/SuperPool.sol#L450
) which both attempt to call `approve()` on the `asset`.


## Impact

- SuperPool cannot be deployed with USDT as the asset on mainnet due to reverts when trying to approve the initial deposit in `SuperPoolFactory::deploySuperPool()`
- Even if a SuperPool was deployed with USDT as the asset without using the factory, any call to `SuperPool::reallocate()` would also revert

## Code Snippet
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/SuperPoolFactory.sol#L73
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/SuperPool.sol#L450


## Tool used

Manual Review

## Recommendation

Use `forceApprove()` instead to ensure compatability




### WETH incompatability on Blast L2 due to the contract not handling `safeTransferFrom()` when src is `msg.sender`

## Summary

The protocol cannot be deployed on Blast L2 if WETH is to be an approved token.

## Vulnerability Detail

The contest [readme](https://audits.sherlock.xyz/contests/349) indicates that the intention is to deploy the protocol on any EVM-compatible network:

>"Q: On what chains are the smart contracts going to be deployed? A: Any EVM-compatbile network"

Additionally, due to the following factors, it is assumed the protcol would be interested in handling WETH on any deployment:
- Weth is [listed for deployment](https://gist.github.com/ruvaag/58c9fc2e5c139451c83c21fda27b77a2) on the example Arbitrum deployment
- Collateral and debt value is denominated in the value of ETH, so having WETH as an approved asset within the system will be convenient

However the [WETH contract on Blast](https://blastscan.io/token/0x4300000000000000000000000000000000000004?a=0x7dec49d6ccd33486ad46e16207c29f5371492996#code) uses a different implementation than on other chains, and consequently will revert when `safeTransferFrom()` is called with `src` == `msg.sender`.


## Impact

- This will cause a [revert](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/SuperPoolFactory.sol#L72) if a SuperPool is attempted to be deployed with the `ASSET` as WETH due to the initial deposit reverting. Even if a SuperPool were to be deployed without using the factory, the `deposit` method is DOSed due to the same reason.
- This will also [prevent any deposit](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L318) into a base pool with `pool.asset` as WETH.
- Please also be aware that WETH is a rebasing token by default on Blast but this functionality can be disabled be by adjusting `YieldMode` (https://docs.blast.io/building/guides/weth-yield)

## Code Snippet
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/SuperPoolFactory.sol#L72
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L318

## POC 

Copy this test into a new file and run the test on https://rpc.blast.io

```javascript
pragma solidity ^0.8.24;

import { Test } from "forge-std/Test.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract BlastTransferTest is Test {
    address user1 = address(0x7dec49D6CCd33486aD46e16207C29F5371492996); // Find an address with sufficient balance on Blast
    address user2 = makeAddr('user2');
    IERC20 public constant WETH = IERC20(0x4300000000000000000000000000000000000004); // WETH on Blast

    function test_POC_TransferFromRevert() public {
        assert(WETH.balanceOf(user1) > 0.01 ether);
        vm.startPrank(user1);
        vm.expectRevert();
        WETH.transferFrom(user1, user2, 0.01 ether);
        vm.stopPrank();
    }
}
```

## Tool used

Manual Review

## Recommendation
- If WETH is required as an approved asset within the system, avoid deploying on Blast L2





### Unliquidatable tokens can be exploited by positions, at the expense of pool depositors

## Summary

Borrowed tokens cannot be seized in a liquidation, despite the protocol recognizing the asset and being able to correctly price it. This results in positions being able to hedge their losses in the case of liquidations, at the expense of pool depositors.

## Vulnerability Detail

In a previous audit, a vulnerability ("C-02 Free Borrowing When Collateral Is The Same Asset") was reported which showed if a pool sets `LTV` for an asset to 100%, then the protocol incorrectly evaluates the health of a postition because the newly borrowed asset is counted as collateral on a position. The recommendation was to not allow borrow and collateral assets to be the same.

The recommendation was implemented by [not allowing pools to set the LTV for their asset](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/RiskEngine.sol#L177) which results in [positions not being able to borrow if the borrow token was 'added to the position'](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/RiskModule.sol#L264-L267) as the LTV for this asset will not be set and the health check will fair.

However, this fix has a downstream impact that the borrowed token is unliquidatable because if the liquidator attempts to seize an asset that is not added to the position, the [liquidation attempt will revert](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/PositionManager.sol#L471-L473).

In addition, if the position is being liquidated through the `liquidateBadDebt()` flow, it is also not seized as this call iterates through [only the assets added to a position](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/PositionManager.sol#L451-L455) and transfers them to the protocol multisig/governance address.

Essentially after any liquidation, the position will retain the borrowed asset, and have their debt reduced or cleared. This fact can be gamed by positions at the expensive of pool depositors in the following way:

- Position makes a borrow from a pool with high LVR and provides just enough collateral to stay healthy. They are able to utilize the borrowed token in Defi with the `PositionManager::exec()` function.
  - If the price of their collateral declines relative to the borrow token, they can get liquidated and retain the more valuable borrow token which was not liquidatable.
  - If the price of their collateral increases relative to the borrow token, they can repay the borrow token and retain the more valuable collateral tokens.

## Impact

- Positions can hedge against liquidations by holding the borrowed token in their position. As shown, this increases the risk to pool depositors who expected to have a future claim on the token the pool loaned (along with interest they earned).


## Code Snippet

https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/RiskEngine.sol#L177
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/RiskModule.sol#L264-L267
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/PositionManager.sol#L471-L473
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/PositionManager.sol#L451-L455

## POC 

Scenario:
1. linearRatePool is created offering loans with 98% LVR on assset 1, user1 deposits assets.
2. User2 creates a position and deposits 1.021e18 worth of asset2 as collateral and adds this token to their position
3. The position borrows 1e18 worth of asset1 from fixedRatePool and is healthy
    The position:
    - totalAssets: 1.021e18
    - totalDebt: 1e18
    - minReqAssets: 1.020e18    
    - total unrecognized assets: 1e18 (asset1 cannot be added to their position)
4. The value of asset2 declines by ~3% in a single price update and is now worth 0.99e18, User2's position becomes undercollateralized
5. The protocol owner calls `liquidateBadDebt()` on this position:
    - Assets worth ~0.99e18 eth are sent to the protocol multisig
    - Debt worth 1e18 eth is socialized among pool depositors
    - Debt is removed from the position
    - 1e18 eth worth of asset1 remains on the position
6. User2 profited from the liquidation of their position at the expense of the pool depositor..


First update the min/max LTV settings in `BaseTest.t.sol` to reflect those in the contest readme:

```diff
    function setUp() public virtual {
        Deploy.DeployParams memory params = Deploy.DeployParams({
            owner: protocolOwner,
            proxyAdmin: proxyAdmin,
            feeRecipient: address(this),
-           minLtv: 2e17, // 0.1
-           maxLtv: 8e17, // 0.8
+           minLtv: 1e17, // 0.1
+           maxLtv: 9.8e17, // 0.98
            minDebt: 0,
            minBorrow: 0,
            liquidationFee: 0,
            liquidationDiscount: 200_000_000_000_000_000,
            badDebtLiquidationDiscount: 1e16,
            defaultOriginationFee: 0,
            defaultInterestFee: 0
        });
```

Then paste/run the following test in `LiquidationTest.t.sol`:

```javascript
    function test_POC_PositionBenefitsFromBorrowTokenBeingUnliquidatable() public {
        // 1. linearRatePool is created offering loans with 98% LVR on assset 1, user1 deposits assets.
        vm.startPrank(poolOwner); 
        riskEngine.requestLtvUpdate(linearRatePool, address(asset2), 0.98e18); 
        riskEngine.acceptLtvUpdate(linearRatePool, address(asset2));
        vm.stopPrank();

        vm.startPrank(user);
        asset1.mint(user, 2e18);
        asset1.approve(address(pool), 2e18);
        pool.deposit(linearRatePool, 2e18, user);
        vm.stopPrank();

        // 2. User2 creates a position and deposits 1.021e18 worth of asset2 as collateral and adds this token to their position
        vm.startPrank(user2);
        asset2.mint(user2, 2e18);
        asset2.approve(address(positionManager), 2e18);
        
        Action[] memory actions = new Action[](4);
        (position, actions[0]) = newPosition(user2, bytes32(uint256(0x123456789)));
        actions[1] = deposit(address(asset2), 1.021e18);
        actions[2] = addToken(address(asset2));


        // 3. The position borrows 1e18 worth of asset1 from fixedRatePool and is healthy
        //     The position:
        //     - totalAssets: 1.021e18
        //     - totalDebt: 1e18
        //     - minReqAssets: 1.020e18    
        //     - total unrecognized assets: 1e18 (asset1 cannot be added to their position)
        actions[3] = borrow(linearRatePool, 1e18);
        positionManager.processBatch(position, actions);
        assertTrue(riskEngine.isPositionHealthy(position));
        vm.stopPrank();

        // 4. The value of asset2 declines by ~3% in a single price update and is now worth 0.99e18, User2's position becomes undercollateralized
        FixedPriceOracle pointOneEthOracle = new FixedPriceOracle(0.97e18); // 3% decline in collateral value
        vm.prank(protocolOwner);
        riskEngine.setOracle(address(asset2), address(pointOneEthOracle));
        riskEngine.validateBadDebt(position);
        vm.stopPrank();

        // 5. The protocol owner calls `liquidateBadDebt()` on this position:
        //     - Assets worth ~0.99e18 eth are sent to the protocol multisig
        //     - Debt worth 1e18 eth is socialized among pool depositors
        //     - Debt is removed from the position
        //     - 1e18 eth worth of asset1 remains on the position
        uint256 pre_User1PoolAssets = pool.getAssetsOf(linearRatePool, user);
        (uint256 pre_totalAssetValue, ,) = riskEngine.getRiskData(position);

        vm.startPrank(address(protocolOwner));
        positionManager.liquidateBadDebt(position);
        vm.stopPrank();

        uint256 post_User1PoolAssets = pool.getAssetsOf(linearRatePool, user);
        (uint256 post_totalAssetValue, ,) = riskEngine.getRiskData(position);

        // 6. User2 profited from the liquidation of their position at the expense of the pool depositor.
        uint256 positionLostDueToLiquidation = pre_totalAssetValue - post_totalAssetValue;
        uint256 valueOfRetainedUnliquidatableAsset = protocol.riskModule().getAssetValue(position, address(asset1));

        assert(valueOfRetainedUnliquidatableAsset > positionLostDueToLiquidation); // postion profited from liquidation, ie. the entire borrowed asset was retained by the position which was more valuable than the assets lost due to liquidation
        assert(post_User1PoolAssets < pre_User1PoolAssets); // the profit made by the position owner was at the expense of pool depositor - user1
    }
```


## Tool used

Manual Review

## Recommendation

- Now that the valid range of LTVs a can set for an asset does not include 100% (according to the contest readme https://audits.sherlock.xyz/contests/349), borrow tokens could be added to positions to be considered collateral. The position would still need to add external collateral to remain healthy.

- Or, continue to not consider the value borrow token for the purposes of position health checks, but allow liquidations to seize the borrowed token as well as any other known asset to the protocol. This prevents the position from retaining valuable assets upon liquidation.



### If `Pool::defaultInterestFee` is 0, depositors can borrow against themselves to DOS pool withdrawals

## Summary

Users must be able to trust the pool owner, but not other depositors in the same pool. But if the `defaultInterestFee` is set to 0, a borrower can borrow against their own capital up to the `poolCap` to DOS pool withdrawals for other users.

## Vulnerability Detail

According to the [readme](https://audits.sherlock.xyz/contests/349) `defaultInterestFee` has a range of permitted values from 0 to 10%. The `defaultInterestFee` represents the [`interestFee`](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L585) that is set when a pool is initialized. 

The `interestFee` pool parameter is used to calculate how many `feeAssets` the `feeRecipient` should recieve, and in turn how many `feeShares` should be [minted to the `feeRecipient`](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L385-L395). If set to 0, [no fees shares will be minted](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L386).


In this case, 100% of the `interestAccrued` is passed onto depositors because `pool.totalBorrowAssets` and `pool.totalDepositAssets` are incremented by the same value.

```javascript
    function accrue(PoolData storage pool, uint256 id) internal {
        (uint256 interestAccrued, uint256 feeShares) = simulateAccrue(pool);

        if (feeShares != 0) _mint(feeRecipient, id, feeShares);

        // update pool state
        pool.totalDepositShares += feeShares; // feeShares is 0 in this case
@>      pool.totalBorrowAssets += interestAccrued;
@>      pool.totalDepositAssets += interestAccrued;

        ... SKIP!...
    }
```

The issue is that a position controlled by a pool depositor can borrow against their own liquidity for free (no fees and interest paid by the attcker will be equal to interest earned). If they do this at the required volume to bring the pool to it's cap, it will block withdrawals for regular users. The amount available to with is [capped to the available liquidity in the pool](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L359-L362).

To withdraw funds from the pool, regular users will need to wait for repayments to create available liquidty. However in the case of large depositors waiting to withdraw, upon repayments there will be a race conditions to withdraw.

Some other details that make this DOS attack viable:
- There is no deadline for positions to repay debt, consequenctly there is no deadline on how long a large pool deposit could be DOSed
- There is no minimum value that a `poolCap` can be set. Due to the permissionless nature of setting up a pool, there will be many pools where this attack is possible
- The attacker will need to deposit extra collateral, this is small relative to the borrow size given they can swap the borrowed funds into an approved collateral asset, and the LTV can be set to as high as 98%
- The attacker's deposit into the pool is still capital efficient as they can utilize the borrowed funds in defi to earn yield through `exec()`


## Impact

- Pool withdrawals can be DOSed for free if the attacker has sufficient capital to reach the pool's cap.
- Race conditions will be created to withdraw when liquidity becomes available. Possible impacts of this are the pool shares trading at less than their potentially redeemable value.

## Code Snippet

https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L585
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L380-L398
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L401-L414
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/Pool.sol#L359-L362

## Tool used

Manual Review

## Recommendation

- Only allow `defaultInterestFee` paramater to be set > 0 to ensure there is non-trivial costs for this attack.




### `maxWithdraw()` and `maxRedeem()` can overestimate maximum the available liquidity, resulting in a lack of ERC4626 complianace

## Summary

`SuperPool::maxWithdraw()` and `SuperPool::maxRedeem()` make internal calls to `_maxWithdraw()`. This function considers the available liquidity of all base pools in the deposit queue and the available asset balance on the SuperPool contract, but does not consider the amount of assets owned by the SuperPool in each underlying pool. 
As a result, there are states where `maxWithdraw()` and `maxRedeem()` will return a value which would cause a revert if `withdraw()` or `redeem()` were called with the reported max values in the same transaction. This breaks ERC4626 compliance.

## Vulnerability Detail

For full [ERC4626 compliance](https://eips.ethereum.org/EIPS/eip-4626), `maxWithdraw()`:

>"MUST return the maximum amount of assets that could be transferred from owner through withdraw and not cause a revert, which MUST NOT be higher than the actual maximum that would be accepted (it should underestimate if necessary)."

And `maxRedeem()`:

>"MUST return the maximum amount of shares that could be transferred from owner through redeem and not cause a revert, which MUST NOT be higher than the actual maximum that would be accepted (it should underestimate if necessary)."

However, both `maxWithdraw()` and `maxRedeem()` make an internal call to `_maxWithdraw()`. As shown below, this function sums the available liquidity in all pools in the deposit queue and the assets available on the SuperPool contract. However this liquidity can include liquidity this SuperPool does not have access to thus overreporting the maximum withdrawable value:

```javascript
    function _maxWithdraw(address _owner, uint256 _totalAssets, uint256 _totalShares) internal view returns (uint256) {
        uint256 totalLiquidity; // max assets that can be withdrawn based on superpool and underlying pool liquidity
        uint256 depositQueueLength = depositQueue.length;
        for (uint256 i; i < depositQueueLength; ++i) {
@>          totalLiquidity += POOL.getLiquidityOf(depositQueue[i]); // @audit this includes liquidity we cannot withdraw
        }
        totalLiquidity += ASSET.balanceOf(address(this)); // unallocated assets in the superpool

        // return the minimum of totalLiquidity and _owner balance
        uint256 userAssets = _convertToAssets(ERC20.balanceOf(_owner), _totalAssets, _totalShares, Math.Rounding.Down);
        return totalLiquidity > userAssets ? userAssets : totalLiquidity;
    }
```

## Impact

- Lack of ERC4626 complaiance
- Internal/external contract integrations expect `maxWithdraw()` and `maxRedeem()` to return a value that would not revert if used in the corresponding functions.


## POC

1. A SuperPool has Base Pools A and B in the `depositQueue` and a single depositor (user1) holding all shares
2. For this SuperPool, the call `Pool::getAssetsOf()` reveals:
  -  Pool A: 10e18 assets 
  -  Pool B: 1e18 assets
3. Calls to `getLiquidityOf()` for Pool A reveals:
  - Pool A: 1e18 liquidity 
  - Pool B: 10e18 liqudity
4. The SuperPool has no latent `ASSET` balance
5. User1 calls `maxWithdraw()` which returns 11e18.
6. User1 calls `Withdraw()` attempting to withdraw 11e18 which reverts as it attempts to transfer user1 more assets than is available.


## Code Snippet

https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/0b472f4bffdb2c7432a5d21f1636139cc01561a5/protocol-v2/src/SuperPool.sol#L474-L485


## Tool used

Manual Review

## Recommendation

The true maximum withdrawable liquidity from each underlying pool is the minimum of the `assetsInPool` and `poolLiquidity`, similar to the flow in `SuperPool::__withdrawFromPools`:

```diff
    function _maxWithdraw(address _owner, uint256 _totalAssets, uint256 _totalShares) internal view returns (uint256) {
        uint256 totalLiquidity; // max assets that can be withdrawn based on superpool and underlying pool liquidity
        uint256 depositQueueLength = depositQueue.length;
        for (uint256 i; i < depositQueueLength; ++i) {
-           totalLiquidity += POOL.getLiquidityOf(depositQueue[i]);
+           uint256 poolId = depositQueue[i];
+           uint256 poolLiquidity = POOL.getLiquidityOf(poolId);
+           uint256 assetsInPool = POOL.getAssetsOf(poolId, address(this));
+           totalLiquidity += (poolLiquidity > assetsInPool) ? assetsInPool : poolLiquidity // Cumulatively add the minimum of assetsInPool and poolLiquidity
        }
        totalLiquidity += ASSET.balanceOf(address(this)); // unallocated assets in the superpool

        // return the minimum of totalLiquidity and _owner balance
        uint256 userAssets = _convertToAssets(ERC20.balanceOf(_owner), _totalAssets, _totalShares, Math.Rounding.Down);
        return totalLiquidity > userAssets ? userAssets : totalLiquidity;
    }
```






























### Depracated

### [note, not a valid issue given deployers of SuperPool are trusted by depositors] ERC4626 donation attack in SuperPool causing first depositor to lose funds // @todo can't make this work yet, a depositor can lose funds but can't see how an attacker can get them


### [not true, review and remove] Standard liquidations can leave positions on the brink of being undercollateralized, after which a minor price change would result in a bad debt liquidation 

Root cause: no buffer in standard liquidations

After a standard `PositionManager::liquidation()` is executed there is a check to ensure the liquidated position is in a healthy state, however there is no buffer. As a result a liquidator seizes assets and repays as little debt as possible to leave the position on the brink of being undercollateralized. After such time, a minor decrease in the price of the position's collateral, or a delayed oracle update will make the position undercollateralized and allow a liquidation through `PositionManager::liquidateBadDebt()`



### [incorrect, don't submit] Incorrect application of `ltv` parameter in `RiskModule::_getMinReqAssetValue()` means it is not possible for a position to be leveraged

## Summary

The protocol docs state that it intneds to allow [leveraged positions](https://docs.sentiment.xyz/concepts/core-concepts/positions#collateral-isolation).

Currently, the range of accepted 'ltv' values for a given pool and asset are not congruent with way the 'ltv' parameter is used, as a result creating a leveraged positon is not possible.


## Vulnerability Detail

In the `RiskEngine` contract, state variables `minLtv` and `maxLtv` are set in the constructor with hardcoded checks to ensure the `minLtv` is greater than 0 and the `maxLtv` is less than 1e18.

https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/9eb56d82d74589fe87de3fb74701c7228bc4adda/protocol-v2/src/RiskEngine.sol#L102-L111
```javascript
    constructor(address registry_, uint256 minLtv_, uint256 maxLtv_) Ownable() {
@>      if (minLtv_ == 0) revert RiskEngine_MinLtvTooLow();
@>      if (maxLtv_ >= 1e18) revert RiskEngine_MaxLtvTooHigh();

        REGISTRY = Registry(registry_);
        minLtv = minLtv_;
        maxLtv = maxLtv_;

        emit LtvBoundsSet(minLtv_, maxLtv_);
    }
```

These state variables control the range of ltv settings that pool owners can propose via the `RiskEngine::requestLtvUpdate()` function:

https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/9eb56d82d74589fe87de3fb74701c7228bc4adda/protocol-v2/src/RiskEngine.sol#L167C1-L187C6
```javascript
    function requestLtvUpdate(uint256 poolId, address asset, uint256 ltv) external {
        ... SKIP!...

        // ensure new ltv is within global limits. also enforces that an existing ltv cannot be updated to zero
@>      if (ltv < minLtv || ltv > maxLtv) revert RiskEngine_LtvLimitBreached(ltv);

        ... SKIP!...
    }
```

As a result, the `RiskEngine::ltvFor` mapping for a given pool and asset is strictly allowed to be in the range 0 < `ltv` < 1e18.

This means that no leveraged position can ever be created in the system due to:
1. the `ltv` being a denominator in the calculation of `minReqAssetValue` in the `RiskModule::_getMinReqAssetValue()` function:


https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/9eb56d82d74589fe87de3fb74701c7228bc4adda/protocol-v2/src/RiskModule.sol#L250-L278
```javascript
    function _getMinReqAssetValue(
        uint256[] memory debtPools,
        uint256[] memory debtValuleForPool,
        address[] memory positionAssets,
        uint256[] memory wt,
        address position
    ) internal view returns (uint256) {
        ... SKIP!...


                // debt is weighted in proportion to value of position assets. if your position
                // consists of 60% A and 40% B, then 60% of the debt is assigned to be backed by A
                // and 40% by B. this is iteratively computed for each pool the position borrows from
@>              minReqAssetValue += debtValuleForPool[i].mulDiv(wt[j], ltv, Math.Rounding.Up);

       ... SKIP!...

        return minReqAssetValue;
    }
```

2. The 'position health' being checked after every action in the `PositionManager` contract:

https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/9eb56d82d74589fe87de3fb74701c7228bc4adda/protocol-v2/src/PositionManager.sol#L231
https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/9eb56d82d74589fe87de3fb74701c7228bc4adda/protocol-v2/src/PositionManager.sol#L245
```javascript
    function process(address position, Action calldata action) external nonReentrant whenNotPaused {
        ... SKIP!...

@>      if (!riskEngine.isPositionHealthy(position)) revert PositionManager_HealthCheckFailed(position);
    }

    ... SKIP!...

    function processBatch(address position, Action[] calldata actions) external nonReentrant whenNotPaused {
        ... SKIP!...

        // after all the actions are processed, the position should be within risk thresholds
@>      if (!riskEngine.isPositionHealthy(position)) revert PositionManager_HealthCheckFailed(position);
    }
```

Noting that:
- `PositionManager::process()` and `PositionManager::processBatch()` are the only user entry points to manage their position, and
- `RiskEngine::isPositionHealthy()` calls `RiskModule::isPositionHealthy()` which calls `RiskModule::_getMinReqAssetValue()`.


## Impact

1. Positions cannot open leveraged positions
2. Positions can be liquidated as soon as a position becomes leveraged (due to the decline in value of collateral token relative to borrowed token). This is because a position is [liquidatable](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/9eb56d82d74589fe87de3fb74701c7228bc4adda/protocol-v2/src/RiskModule.sol#L84) if `totalAssetValue < minReqAssetValue` across all position assets and pool debts.


## POC

Analysing the [`minReqAssetValue`](https://github.com/sherlock-audit/2024-08-sentiment-v2-Nihavent/blob/9eb56d82d74589fe87de3fb74701c7228bc4adda/protocol-v2/src/RiskModule.sol#L272) with respect to the range of permitted `ltv` settings 

```javascript
    minReqAssetValue += debtValuleForPool[i].mulDiv(wt[j], ltv, Math.Rounding.Up);
```

Taking the simplest scenario where a `postion` has one debt pool and one collateral asset (`wt[j]` == 1e18). Lets see the required `minReqAssetValue` for various `ltv` settings according to the above formula:

| `ltv`   | `debtValuleForPool[i]` | `wt[j]` | `minReqAssetValue` |
|---------|------------------------|---------|--------------------|
| 1       | 1e18                   | 1e18    | 1     e36          |
| 1e17    | 1e18                   | 1e18    | 10    e18          |
| 5e17    | 1e18                   | 1e18    | 2     e18          |
| 9e17    | 1e18                   | 1e18    | 1.11  e18          |
| 9.99e17 | 1e18                   | 1e18    | 1.001 e18          |

As shown above, with a range of 0 < `ltv` < 1e18, no leverage is possible as the minimum assets required for a position to be healthy are greater than the debt value.

Paste the following POC into `RiskModule.t.sol` to show that opening a 2x leveraged position is not possible when `ltv` == 5e17.

```javascript
    function test_POC_CreatingLeveragedPositionNotPossible() public {
        vm.startPrank(user);
        asset2.approve(address(positionManager), 1e18); 

        assertEq(riskEngine.ltvFor(fixedRatePool, address(asset2)), 5e17); // LTV for this pool and asset is 5e17

        Action[] memory actions = new Action[](4);
        (position, actions[0]) = newPosition(user, bytes32(uint256(0x123456789)));  // Create new position
        actions[1] = deposit(address(asset2), 1e18);                                // Deposit 1e18 asset2
        actions[2] = addToken(address(asset2));                                     // Add asset2 to position
        actions[3] = borrow(fixedRatePool, 2e18);                                   // Attempt to borrow 2e18 asset1
        vm.expectRevert(abi.encodeWithSelector(PositionManager.PositionManager_HealthCheckFailed.selector, position));
        positionManager.processBatch(position, actions);
        vm.stopPrank();
    }
```


## Tool used

Manual Review

## Recommendation

Swap the order of `ltv` and `wt[j]` in `_getMinReqAssetValue()`. An LTV setting of 5e17 will now correspond to 2x leverage.

```diff
    function _getMinReqAssetValue(
        uint256[] memory debtPools,
        uint256[] memory debtValuleForPool,
        address[] memory positionAssets,
        uint256[] memory wt,
        address position
    ) internal view returns (uint256) {
        ... SKIP!...


                // debt is weighted in proportion to value of position assets. if your position
                // consists of 60% A and 40% B, then 60% of the debt is assigned to be backed by A
                // and 40% by B. this is iteratively computed for each pool the position borrows from
-               minReqAssetValue += debtValuleForPool[i].mulDiv(wt[j], ltv, Math.Rounding.Up);
+               minReqAssetValue += debtValuleForPool[i] * wt[j] * ltv / (1e18 * 1e18);

       ... SKIP!...

        return minReqAssetValue;
    }
```










### When pool utilization is high, race conditions are created for withdrawals & borrows after a deposit or repayment is made

- What can be done about this whilst still retaining the design principle of fragmented/isolated pool liquidity?



### Chainlink price feeds are not available on all EVM-compatile networks

## Summary

As it stands at the time of the audit, the protocol is not compatible with all EVM-compatible networks due to the absence of Chainlink price feeds on all EVM-compatible networks

## Vulnerability Detail

The contest readme states that the contracts will be deployed on any EVM-compatbile network:

">On what chains are the smart contracts going to be deployed? Any EVM-compatbile network"

However as it stands, this requirement cannot be met due to the absence of Chainlink price feeds, for example Blast L2 does not have price feed addresses available.

## Impact

## Code Snippet

## Tool used

Manual Review

## Recommendation