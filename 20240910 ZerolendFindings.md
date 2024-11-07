
## Summary
A summary with the following structure: {root cause} will cause [a/an] {impact} for {affected party} as {actor} will {attack path}

## Root Cause
In case it’s a mistake in the code: In {link to code} the {root cause}

## Internal pre-conditions
A numbered list of conditions to allow the attack path or vulnerability path to happen:

## External pre-conditions
Similar to internal pre-conditions but it describes changes in the external protocols

## Attack Path
A numbered list of steps, talking through the attack path:

## Impact
In case it's an attack path: The {affected party} suffers an approximate loss of {value}. [The attacker gains {gain} or loses {loss}].

## PoC
A coded PoC. Required in some cases but optional for other cases.

## Mitigation
Mitigation of the issue (optional)






# High

# [H-1] `LiquidationLogic::_calculateDebt()` treats debt shares as assets which results in bad debt, invalid health checks, and loss of protocol liquidation fees

## Summary
The pool's liquidation flow incorrectly treats debt values denominated in shares as values denominated in assets. This creates issues such as liquidators being unable to repay the full debt amount due to the `debtIndex` being greater than 1. The reserve asset of the debt is also incorrectly removed from the postiion which breaks future health checks.


## Root Cause
First the root cause is the lack of conversion of debt shares to debt amount in [`LiquidationLogic::_calculateDebt()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L259-L273)

```javascript
  function _calculateDebt(
    DataTypes.ExecuteLiquidationCallParams memory params,
    uint256 healthFactor,
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage balances
  ) internal view returns (uint256, uint256) {
@>  uint256 userDebt = balances[params.debtAsset][params.position].debtShares; // @audit should convert this to debt amount

    uint256 closeFactor = healthFactor > CLOSE_FACTOR_HF_THRESHOLD ? DEFAULT_LIQUIDATION_CLOSE_FACTOR : MAX_LIQUIDATION_CLOSE_FACTOR;

    uint256 maxLiquidatableDebt = userDebt.percentMul(closeFactor);

@>  uint256 actualDebtToLiquidate = params.debtToCover > maxLiquidatableDebt ? maxLiquidatableDebt : params.debtToCover; // @audit compares an amount to shares

@>  return (userDebt, actualDebtToLiquidate); // @audit return units: (shares, amount) causing problems in the liquidation flow
  }
```

The returned `actualDebtToLiquidate` from above will be the amount of debt the liquidator [transfers back to the pool](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L192), so the debt isn't fully repaid (impact 1). 

When the return values in `_calculateDebt()` are equal, the debt is incorrectly [removed from the position](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L150-L152), despite the position having outstanding debt shares. This results in the pending debt no longer being considered in health checks and the position looking more 'healthy' than it actually is (impact 2).

And finally, the error of `actualDebtToLiquidate` being in shares propagates to the `liquidationProtocolFeeAmount` through the `LiquidationLogic::_calculateAvailableCollateralToLiquidate()` function. The liquidation flow incorrectly assumes this is an amount, and [attempts to convert it to shares by dividing by the index](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L181). This results in the protocol earning less fees than expected (impact 4).


## Internal pre-conditions
1. Interest needs to have accrued on a debt position. This is trivial because `ReserveLogic::updateState()` is called at the begining of the liquidation flow.

## External pre-conditions
1. Collateral price fluction makes a position liquidatable


## Attack Path
1. Liquidator calls `Pool::liquidate()` or `Pool::liquidateSimple()` on a liquidatable posistion


## Impact
1. All debt will not be repaid even if when liqudator requests to repay all debt, due to the 'number of shares' being repaid as 'amount' of assets. 
2. When `vars.userDebt == vars.actualDebtToLiquidate`, the borrowed token is set to false in the user configuration object. This means despite the position having outstanding debtShares, this debt will be ignored in future health checks.
3. `liquidationProtocolFeeAmount` is scaled down again as the flow assume it's an 'amount' not 'number of shares', resulting in less fees earned for the protocol.


## PoC

To run the test create a new file in `/test/forge/core/pool` and paste the below contents. The below test shows the first two impacts above in the final assert statements

```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {PoolLiquidationTest} from './PoolLiquidationTests.t.sol';

import {ReserveConfiguration} from './../../../../contracts/core/pool/configuration/ReserveConfiguration.sol';
import {UserConfiguration} from './../../../../contracts/core/pool/configuration/UserConfiguration.sol';
import {DataTypes} from './../../../../contracts/core/pool/configuration/DataTypes.sol';

import {console2} from 'forge-std/src/Test.sol';

contract AuditLiquidationFlowCalcsDebtInShares is PoolLiquidationTest {
  using UserConfiguration for DataTypes.UserConfigurationMap;
  using ReserveConfiguration for DataTypes.ReserveConfigurationMap;

function test_POC_LiquidationFlowCalcsDebtInShares() public {
    // Setup
    oracleA.updateAnswer(1e8);
    oracleB.updateAnswer(1e8);

    // Alice takes loan, Bob is supplier/liquidator
    uint256 aliceMintAmount = 1000e18;
    uint256 bobMintAmount = 1000e18;
    uint256 supplyAmount = 150e18;
    uint256 borrowAmount = 70e18;

    _mintAndApprove(alice, tokenA, aliceMintAmount, address(pool));         // alice collateral
    _mintAndApprove(bob, tokenB, bobMintAmount, address(pool));             // bob supply

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmount, 0); 
    vm.stopPrank();

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, supplyAmount, 0);  // alice collateral
    pool.borrowSimple(address(tokenB), alice, borrowAmount, 0);    
    vm.stopPrank();

    // Price of collateral decreases
    vm.warp(block.timestamp + 90 days);
    oracleA.updateAnswer(5e7);
    oracleA.updateRoundTimestamp();
    oracleB.updateRoundTimestamp();
    pool.forceUpdateReserves(); 




    vm.startPrank(bob);
    bytes32 _pos = keccak256(abi.encodePacked(alice, 'index', uint256(0)));
    pool.liquidateSimple(address(tokenA), address(tokenB), _pos, type(uint256).max); // Bob attempts to liquidate Alice (full liquidation)
    vm.stopPrank();

    (, uint256 totalDebtBase, , , , uint256 healthFactor) = pool.getUserAccountData(alice, 0);

    // Impact 1 - Despite attempting to liquidate all of Alice's debt, Bob only liquidates a portion of it. As a result Alice has remaining collateral value, no remaining debt value, but they have debt shares.
    assert(pool.getBalanceRawByPositionId(address(tokenB), _pos).debtShares > 0); // Alice has remaining debtShares

    // Impact 2 - Invalid future health checks on Alice's position
    assert(totalDebtBase == 0); // Alice's debt value is apparently 0 as it was incorrectly removed consideration as debt
    assert(healthFactor == type(uint256).max); // Alice has infinite health factor despite having outstanding debtShares

  }
}
```

## Mitigation
Convert the debt shares to debt amount in `_calculateDebt()`. Note the input variable `userCollateralBalance` to `LiquidationLogic::_calculateAvailableCollateralToLiquidate()` will also need to be converted to amounts (instead of shares).





# [H-2] `PositionBalanceConfiguration::getSupplyBalance()` adds interest in assets to a shares balance resulting in users being unable to withdraw their entire balance

## Summary
Due to not converting the 'base' amount to a value denominated in assets `PositionBalanceConfiguration::getSupplyBalance()` returns a lower-than expected asset balance. This will block most users from withdrwaing all assets (when they minted shares with `liquidityIndex` > 1e27). Over time the impact increases because the delta between the expected value balance and the actual return value will increase due to the `liquidityIndex` increasing.

## Root Cause

[`PositionBalanceConfiguration::getSupplyBalance()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L126-L129) adds an `increase` denominated in 'asset amount' to `self.supplyShares` denominated in shares:

```javascript
  function getSupplyBalance(DataTypes.PositionBalance storage self, uint256 index) public view returns (uint256 supply) {
    uint256 increase = self.supplyShares.rayMul(index) - self.supplyShares.rayMul(self.lastSupplyLiquidtyIndex);
>@  return self.supplyShares + increase;
  }
```
This results in the function returning lower than expected supply balances.

This function is utilized in the pool withdraw flow in [`ValidationLogic::validateWithdraw()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L123) which checks the amount a user is attempting to withdraw does not exceed the return value for [`getSupplyBalance()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L118). Therefore users will be DOSed from withdrawing their entire balance if the `liquidityIndex` when they deposited was > 1e27.
For example, if the liquidityIndex is 1.2e27 and a user deposits 12e18 weth, they will only be able to withdraw ~10e18 weth.


The function is also called in [`CuratedVaultSetters::_accrueFee()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L178) via [`CuratedVaultGetters::_accruedFeeShares()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultGetters.sol#L186) and ['CuratedVault::totalAssets()'](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVault.sol#L370) and ['Pool::getBalanceByPosition()](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/PoolGetters.sol#L48). So the assets which a vault owns in underlying pools will be also be under-represented in the curated vault accounting.



## Internal pre-conditions
1. Interest needs to be earned on supplied assets, with liquidityIndex updated

## External pre-conditions
N/A

## Attack Path
1. A pool's liquidityIndex is > 1e27
2. Any user supplies assets to a pool 
4. User is DOSed from accessing all of their deposited funds (including or excluding interest earned), even if the pool has liquidity to service the withdrawal.

## Impact
1. Users unable to withdraw all of their deposited funds (shown in the POC).
2. `CuratedVault::totalAssets()` does not return the index-adjusted asset value, leading to a range of issues in the curated vault due to under-representing the assets controlled by a curated vault. For example:
  - Total assets are [calculated incorrectly in `accureFeeShares`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultGetters.sol#L186)
  - Shares to mint upon a deposit are [calculated incorrectly](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVault.sol#L323-L329). Note that as the underlying pool's `liquidityIndex` increases (as it will over time), the assets will be more under-represented favouring early vault depositors at the expense of future vault depositors.
3. `NFTPositionManagerSetters::_supply()` calculates a position's asset amount incorrectly for the puposes of reward calculations
4. `NFTPositionManagerSetters::_withdraw()` calculates assets amount incorrectly for the puposes of reward calculations


## PoC
The following coded POC shows a user losing some of their deposit due to the bug. 

To run the test create a new file in `/test/forge/core/pool` and paste the below contents

```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {console2} from 'forge-std/src/Test.sol';
import {PoolWithdrawTests} from './PoolWithdrawTests.t.sol';
import {WadRayMath} from '../../../../contracts/core/pool/utils/WadRayMath.sol';
import {PoolErrorsLib} from '../../../../contracts/interfaces/errors/PoolErrorsLib.sol';

contract PoolUserUnableToWithdrawBalance is PoolWithdrawTests {
    using WadRayMath for uint256;

  function test_POC_UserUnableToWithdrawEntireBalance() external {
    uint256 supplyAmount = 50 ether;
    uint256 mintAmount = 100 ether;
    uint256 borrowAmount = 30 ether;
    uint256 index = 1;
    address alice = makeAddr('alice');
    address bob = makeAddr('bob');

    // Alice deposits and borrows from the pool to generate some interest
    _mintAndApprove(alice, tokenA, mintAmount, address(pool));
    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, supplyAmount, index);
    pool.borrowSimple(address(tokenA), alice, borrowAmount, index);
    vm.stopPrank();

    // Some time passes, interest is accrued
    for(uint256 i = 0; i < 365; i++){
      vm.warp(block.timestamp + 1 days);
      pool.forceUpdateReserves();
      oracleA.updateRoundTimestamp();
    }
    assert(pool.getReserveData(address(tokenA)).liquidityIndex > 1e27); // Ensure interest has been accrued

    // New user deposits 1.08 e18 tokens
    _mintAndApprove(bob, tokenA, mintAmount, address(pool));
    vm.startPrank(bob);
    uint256 bobSupplyAmount = 1.08e18;
    pool.supplySimple(address(tokenA), bob, bobSupplyAmount, index);

    // Bob immediately attempts to withdraw their supplied balance and reverts
    vm.expectRevert("NOT_ENOUGH_AVAILABLE_USER_BALANCE");
    pool.withdrawSimple(address(tokenA), bob, bobSupplyAmount, index);
  }
}
```

## Mitigation

```diff
  function getSupplyBalance(DataTypes.PositionBalance storage self, uint256 index) public view returns (uint256 supply) {
-   uint256 increase = self.supplyShares.rayMul(index) - self.supplyShares.rayMul(self.lastSupplyLiquidtyIndex);
-   return self.supplyShares + increase;
+   return self.supplyShares.rayMul(index);
  }
```






# [H-3] `PositionBalanceConfiguration::getDebtBalance()` adds interest in assets to a shares balance which prevents full repay of debt and completely DOSes `nftPositionManager::repay()`

## Summary
Users are unable to fully repay their debt through the `pool::repay()` flow and the `nftPositionManager::repay()` is completely DOSed due to `PositionBalanceConfiguration::getDebtBalance()` returning a lower than expected debt balance. `getDebtBalance()` should convert the 'base' debt to a value denominated in assets before returning.

## Root Cause

[`PositionBalanceConfiguration::getDebtBalance()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L137-L140) adds an `increase` denominated in 'asset amount' to `self.debtShares` denominated in shares:

```javascript
  function getDebtBalance(DataTypes.PositionBalance storage self, uint256 index) internal view returns (uint256 debt) {
    uint256 increase = self.debtShares.rayMul(index) - self.debtShares.rayMul(self.lastDebtLiquidtyIndex);
@>  return self.debtShares + increase;
  }
```

This results in the function returning lower than expected debt balances. 

I will focus on explaining how this issue impacts the repayment flows in the protocol. 

1. The `NFTPositionManager::repay()` call [invokes `NFTPositionManagerSetters::_repay()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTPositionManager.sol#L119) which [calls `getDebt()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L119-L121) twice. Each call then [invokes `getDebtBalance()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/PoolGetters.sol#L96). This logic supposedly tracks the debt balance pre and post repayment, and [their difference is compared to `repaid.assets`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L123-L125) to ensure the actual debt repaid matches the expected debt repaid.
The issue here is that `repaid.assets` is in 'assets' and the return value of `getDebt()` is primarily in shares, thus these values won't match for any repaid amount which completely bricks `NFTPositionManager::repay()`. 

2. A user also cannot repay through the `Pool::repay()` flow due to the same issue in `getDebtBalance()`. `Pool::repay()` [invokes `PoolSetters::_repay()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/Pool.sol#L121) which [calls `BorrowLogic::executeRepay()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/PoolSetters.sol#L129-L143). This function sets `payback.assets` to the [return value of `getDebtBalance()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L126) which contains the bug. Finally, this [check can only scale down the value of `payback.assets`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L137). The value of `payback.assets` is then used to [repay the debt](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L158).    


## Internal pre-conditions
1. Interest needs to have accrued on a borrowed asset, with borrowIndex updated

## External pre-conditions
N/A

## Attack Path
1. User mints an NFT position, supplying collateral, and taking debt
2. Interest accrues on debt
3. User is unable to repay the debt through the `NFTPositionManager::repay()` function

Or
1. User has debt in a pool
2. User attempts to repay the full debt
3. The repayment is capped at the return value of `getDebtBalance()`, so all debt is not repaid.

## Impact
1. The `nftPositionManager::repay()` flow is DOSed for all repay values due to the described bug and a strict equality in `NFTPositionManagerSetters::_repay()`
2. The user is unable to repay their full debt through the `Pool::repay()` flow
3. `NFTPositionManagerSetters::_borrow()` calculates debt amount incorrectly for the puposes of reward calculations. Debt is understated and rewards will therefore be earned at a lower rate due to lower 'total balance' 
 
## PoC

Create a new file in `/test/forge/core/positions/` and run `test_POC_NFTPositionManagerRepayDos()`. It shows a simple supply/borrow/repay flow through an NFT position where a repayment of any amount will revert. 

```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {NFTPostionManagerTest} from './NFTPositionManagerTest.t.sol';
import {DataTypes} from 'contracts/core/pool/configuration/DataTypes.sol';
import {INFTPositionManager} from 'contracts/interfaces/INFTPositionManager.sol';
import {NFTErrorsLib} from 'contracts/interfaces/errors/NFTErrorsLib.sol';
import {console2} from 'forge-std/src/Test.sol';

contract NFTPostionManagerRepayDOS is NFTPostionManagerTest {

  function test_POC_NFTPositionManagerRepayDos() external {
    testShouldSupplyAlice();
    uint256 repayAmount = 10 ether;
    uint256 borrowAmount = 20 ether;
    DataTypes.ExtraData memory data = DataTypes.ExtraData(bytes(''), bytes(''));
    INFTPositionManager.AssetOperationParams memory params =
      INFTPositionManager.AssetOperationParams(address(tokenA), alice, borrowAmount, 1, data);

    vm.startPrank(alice);
    nftPositionManager.borrow(params);
    assertEq(tokenA.balanceOf(address(pool)), 30 ether, 'Pool Revert');
    assertEq(tokenA.balanceOf(alice), 70 ether, 'Alice Revert');

    vm.warp(block.timestamp + 1 days); // fast forward some time to accrue interest
    pool.forceUpdateReserves();
    assert(pool.getReserveData(address(tokenA)).borrowIndex > 1e27); // Prove some interest has accrued on borrows, ie borrowAssets : borrowShares is not 1:1

    params.amount = repayAmount;

    vm.expectRevert(NFTErrorsLib.BalanceMisMatch.selector); // will revert due to the bug in getDebt
    nftPositionManager.repay(params);

    vm.stopPrank();
  }
}
```

## Mitigation

```diff
  function getDebtBalance(DataTypes.PositionBalance storage self, uint256 index) internal view returns (uint256 debt) {
-   uint256 increase = self.debtShares.rayMul(index) - self.debtShares.rayMul(self.lastDebtLiquidtyIndex);
-   return self.debtShares + increase;
+   return self.debtShares.rayMul(index);
  }
```




# [H-4] Treasury shares are 'burnt' upon but never 'minted' resulting in suppliers not being able to withdraw all of their assets

## Summary
Treasury shares are a way of keeping track of treasury earnings which is a function of the interest paid on debt taken or the premium earned on flash loans. When they're 'minted' they're tracked in their own variable (separate from regular supplyShares), however when they're burnt they're subtracted from the regular reserve supplyShares which will create an underflow in the withdrawal call.


## Root Cause
Each time a pool's reserve state is updated, [`ReserveLogic::_accrueToTreasury()` is called](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L92).
This function incriments `_reserve.accruedToTreasuryShares` by the shares equiavalent of the assets taken as a fee:

```javascript
  function _accrueToTreasury(uint256 reserveFactor, DataTypes.ReserveData storage _reserve, DataTypes.ReserveCache memory _cache) internal {
    ... SKIP!...

    vars.amountToMint = vars.totalDebtAccrued.percentMul(reserveFactor);

    if (vars.amountToMint != 0) _reserve.accruedToTreasuryShares += vars.amountToMint.rayDiv(_cache.nextLiquidityIndex).toUint128();
  }
```

However, this function does not increment `supplyShares` in the ReserveSupply struct for this reserve asset. 

Note that `_reserve.accruedToTreasuryShares` can also [increment](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/FlashLoanLogic.sol#L116) in the `FlashLoanLogic::_handleFlashLoanRepayment()` function. Again, this accruel of treasury shares does not increment the `supplyShares` in the ReserveSupply struct for this reserve asset.

When any pool withdrawal is executed, [`PoolLogic::executeMintToTreasury()` is called](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/PoolSetters.sol#L84). This [function decrements `totalSupply.supplyShares` by `reserve.accruedToTreasuryShares`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/PoolLogic.sol#L83-L103) but as shown above they were never added. 

```javascript
  function executeMintToTreasury(
    DataTypes.ReserveSupplies storage totalSupply,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    address treasury,
    address asset
  ) external {
    DataTypes.ReserveData storage reserve = reservesData[asset];

    uint256 accruedToTreasuryShares = reserve.accruedToTreasuryShares;

    if (accruedToTreasuryShares != 0) {
      reserve.accruedToTreasuryShares = 0;
      uint256 normalizedIncome = reserve.getNormalizedIncome();
      uint256 amountToMint = accruedToTreasuryShares.rayMul(normalizedIncome);

      IERC20(asset).safeTransfer(treasury, amountToMint);
@>    totalSupply.supplyShares -= accruedToTreasuryShares;

      emit PoolEventsLib.MintedToTreasury(asset, amountToMint);
    }
  }
```

As a result there will be a supplier who will not be able to withdraw all of their supplyShares because the withdrawal flow `PositionBalanceConfiguration::withdrawCollateral()` is called which [decrements `supply.supplyShares`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L95) resulting in the underflow:

```javascript
  function withdrawCollateral(
    DataTypes.PositionBalance storage self,
    DataTypes.ReserveSupplies storage supply,
    uint256 amount,
    uint128 index
  ) internal returns (uint256 sharesBurnt) {
    sharesBurnt = amount.rayDiv(index);
    require(sharesBurnt != 0, PoolErrorsLib.INVALID_BURN_AMOUNT);
    self.lastSupplyLiquidtyIndex = index;
    self.supplyShares -= sharesBurnt;
@>  supply.supplyShares -= sharesBurnt;
  }
```


## Internal pre-conditions
1. A Pool exists with a non-zero ReserveFactor
2. Interest accrues on the supplied assets due to borrowers
3. A withdrawal of any amount occurs

## External pre-conditions
N/A

## Attack Path
1. A supplier deposits into a pool with a non-zero ReserveFactor
2. A borrower borrows all available liquidity
3. Some time passes, interest on the debt accrues
4. The borrower repays in full
5. The supplier attempts to withdraw their entire deposit + interest earned, but their withdrawal reverts

## Impact
- After any withdrawal the total number of supply shares on the `ReserveSupplies` struct for a given reserve asset will be less than the sum of all supplier's supplyShares. As a result, not all suppliers will be able to withdraw their balance and their funds are stuck in the protocol.


## PoC
The below coded POC implements the 'Attack Path' described above.

First, add this line to the `CorePoolTests::_setUpCorePool()` function to create a scenario with a non-zero reserve factor:

```diff
  function _setUpCorePool() internal {
    poolImplementation = new Pool();

    poolFactory = new PoolFactory(address(poolImplementation));
+   poolFactory.setReserveFactor(1e3); // 10%
    configurator = new PoolConfigurator(address(poolFactory));

    ... SKIP!...
  }
```

Then create a new file in /test/forge/core/pool and paste the below contents. Take careful note of the emitted events and the revert reasons as they show the revert is due to the described issue.

```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {console2} from 'forge-std/src/Test.sol';
import {PoolLiquidationTest} from './PoolLiquidationTests.t.sol';
import {PoolEventsLib, PoolSetup} from './PoolSetup.sol';

contract AuditTreasurySharesBurntButNotMinted is PoolLiquidationTest {
    event Transfer(address indexed from, address indexed to, uint256 value);

  function test_POC_TreasurySharesBurntButNotMinted() public {
    uint256 aliceMintAmount = 10_000e18;
    uint256 bobMintAmount = 10_000e18;
    uint256 supplyAmount = 1000e18;
    uint256 borrowAmount = 1000e18;

    _mintAndApprove(alice, tokenA, aliceMintAmount, address(pool));         // alice collateral
    _mintAndApprove(bob, tokenB, bobMintAmount, address(pool));             // bob supply
    _mintAndApprove(alice, tokenB, aliceMintAmount, address(pool));         // alice needs some funds to pay interest

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmount, 0); 
    vm.stopPrank();

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, aliceMintAmount, 0);  // alice collateral
    pool.borrowSimple(address(tokenB), alice, borrowAmount, 0);     // 100% utilization
    vm.stopPrank();

    vm.warp(block.timestamp + 60 days); // Time passes, interest accrues, treasury shares accrue
    pool.forceUpdateReserves();

    // Alice full repay
    vm.startPrank(alice);
    tokenB.approve(address(pool), type(uint256).max);
    pool.repaySimple(address(tokenB), type(uint256).max, 0);
    
    (uint256 supplyBalance, uint256 supplyShares, , uint256 debtShares) = pool.marketBalances(address(tokenB));
    assert(debtShares == 0); // All debt has been repaid

    bytes32 bobPos = keccak256(abi.encodePacked(bob, 'index', uint256(0)));
    assert(pool.supplyShares(address(tokenB), bobPos) + pool.getReserveData(address(tokenB)).accruedToTreasuryShares > supplyShares); // Bob's supplyShares + accruedToTreasuryShares exceeds the total shares
    
    // Bob withdraw
    vm.startPrank(bob);
    uint256 treasuryAssetsToClaim = pool.getReserveData(address(tokenB)).accruedToTreasuryShares * pool.getReserveData(address(tokenB)).liquidityIndex / 1e27;

    vm.expectEmit(true, true, true, true);
    emit PoolEventsLib.Withdraw(address(tokenB), bobPos, bob, supplyBalance); // Withdrawal is successful

    vm.expectEmit(true, true, true, false);
    emit Transfer(address(pool), address(pool.factory().treasury()), treasuryAssetsToClaim); // Transfer of treasury assets is successful

    vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", 0x11));         // Revert is due to underflow when deducting shares
    
    pool.withdrawSimple(address(tokenB), bob, supplyBalance, 0);
  }

}
```


## Mitigation

Remove the decrement operation in [`executeMintToTreasury()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/PoolLogic.sol#L83-L103). This should not affect the liquidityIndex as the `reserveFactor` was already accounted for in the calculation of the `liquidityRate` (however this is out of scope so I am not a position to say for sure how this will be implemented in production).

```diff
  function executeMintToTreasury(
    DataTypes.ReserveSupplies storage totalSupply,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    address treasury,
    address asset
  ) external {
    DataTypes.ReserveData storage reserve = reservesData[asset];


    uint256 accruedToTreasuryShares = reserve.accruedToTreasuryShares;


    if (accruedToTreasuryShares != 0) {
      reserve.accruedToTreasuryShares = 0;
      uint256 normalizedIncome = reserve.getNormalizedIncome();
      uint256 amountToMint = accruedToTreasuryShares.rayMul(normalizedIncome);


      IERC20(asset).safeTransfer(treasury, amountToMint);
-     totalSupply.supplyShares -= accruedToTreasuryShares;


      emit PoolEventsLib.MintedToTreasury(asset, amountToMint);
    }
  }
```


# [H-5] [todo] Bad debt is not handled by the protocol, resulting in withdrawal race conditions for suppliers

## Summary

Once bad debt is in the protocol there is no way to socialize it evenly across all suppliers, meaning the first suppliers to withdraw will retain all funds + interest earned, at the expense of suppliers who are unable to withdraw in time.

## Root Cause

The root cause of bad debt is price fluctions in the collateral/debt making the debt worth more than the collateral. In these circumstances, the liquidation will seize all collateral, but only partially repay the debt (as explained below). The result is a position with no collateral and outstanding debt, for which no price movements can recover the position from being bad debt. 

If the value of bad debt exceeds the difference between the value of all supplyShares and all debtShares then rational suppliers should rush to withdraw their liquidity, as the last users will be left holding `liquidityShares` with no claimable assets. 

Please note, for the below description assume that `debtToCover` and `userCollateralBalance` are in token amounts, not share amounts. I believe this is a separate issue.
In the liquidation flow, `LiquidationLogic::_calculateAvailableCollateralToLiquidate()` is called. This function first calculates `vars.baseCollateral` which is the amount of collateral equal in value to the amount of `debtToCover` (1). This value is then scaled up by the `liquidationBonus` and assigned to `vars.maxCollateralToLiquidate` (2). Then if this value would exceed the position's collateral balance, all collateral is liquidated and the `vars.debtAmountNeeded` is calculated which scales down `debtAssetPrice` based on the amount of collateral being seized (3). Note in step 3, the `liquidationBonus` is still applied so liquidators can always except to receive an amount of collateral worth more than the value of debt they repay. 

```javascript
  function _calculateAvailableCollateralToLiquidate(
    DataTypes.ReserveData storage collateralReserve,
    DataTypes.ReserveCache memory debtReserveCache,
    uint256 debtToCover,
    uint256 userCollateralBalance,
    uint256 liquidationBonus,
    uint256 collateralPrice,
    uint256 debtAssetPrice,
    uint256 liquidationProtocolFeePercentage
  ) internal view returns (uint256, uint256, uint256) {
    ... SKIP!...
@>  vars.baseCollateral = ((vars.debtAssetPrice * debtToCover * vars.collateralAssetUnit)) / (vars.collateralPrice * vars.debtAssetUnit); // 1

    vars.maxCollateralToLiquidate = vars.baseCollateral.percentMul(liquidationBonus); // 2

    if (vars.maxCollateralToLiquidate > userCollateralBalance) {
      vars.collateralAmount = userCollateralBalance;
@>    vars.debtAmountNeeded = ( // 3
        (vars.collateralPrice * vars.collateralAmount * vars.debtAssetUnit) / (vars.debtAssetPrice * vars.collateralAssetUnit)
      ).percentDiv(liquidationBonus);
    } else {
      vars.collateralAmount = vars.maxCollateralToLiquidate;
      vars.debtAmountNeeded = debtToCover;
    }
  ... SKIP!...
  }
```


## Internal pre-conditions
1. A pool has suppliers and borrowers
2. A liquidator partially liquidates a position with bad debt taking all collateral and repaying less than all of the debt

## External pre-conditions
1. Due to price changes, the health of a position declines from 'healthy' (>100% health factor) to 'bad debt' (< avgLTV). Note the liklihood of this occuring varies with the `liquidationThreshold` setting.

## Attack Path
1. A Pool supports tokenA as collateral with an `LTV` of 75%, `liquidationThreshold` of 80%, and `liquidationBonus` of 105%.
2. The same pool has enabled tokenB for borrowing
3. When both tokenA and tokenB have a normalized price of 1e8, a borrower supplies 1e18 tokenA and borrows 0.7e18 tokenB. The resulting healthFactor of the position is .8 * 1e18 * 1e8 / (.7e18 * 1e8) ~= 1.14.
4. In a single price update, the normalized price of tokenA drops to 0.8e8 and the price tokenB normalized increases to 1.2e8. The healthFactor of the position is now .8 * 1e18 * 0.8e8 / (.7e18 * 1.2e8) ~= 0.76. The position is therefore fully liquidatable.
  - However, the debt on the position will not be completely repaid because the liquidator always gets the `liquidationBonus` on the assets they seize.
5. The liquidator attempts to liquidate the max, [all collateral is seized, but the debt amount to repay is scaled down](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L357-L360).
   - Base collat = .7e18 * 1.2e18 / .8e8 = 1.05e18 * 1.05 = 1.103e18
   - But this is more collateral than the position has so it's scaled down to 1e18 worth 0.8 units.
   - The actual debt repaid is therefore (1e18 * 0.8e8 / 1.2e8) / 1.05 = 0.6359e18 tokens worth ~ 0.763 units.
6. The position is now left with no collateral and 0.36e18 debt tokens worth of bad debt.


## Impact
1. Pool suppliers are incentivized to exit their positions from the pool as fast as possible. The last supplier(s) to do so will be left with unclaimable supplyShares. There is likely to be very limited withdrawal liquidity in this time, and savy suppliers may backrun `repay` transactions with `withdraw` transactions to minimize their losses.
2. Bad debt continues to accrue interest which will never be repaid
3. NFTPositions with bad debt will continue to accrue rewards through `NFTRewardDistributor`
4. Bad debt continues to count towards 'useage ratios' of the pool which impact interest rate calculations
5. `supplyIndex` will be out of sync with what the `supplyShares` are actually worth, leading to incorrect valuation of curated vault shares and vault interest calculations

## PoC
N/A

## Mitigation

1. Implement an admin-only function which takes a position and checks for bad debt, and that the position has been liquidated as much as possible (ie. no remaining collateral). If so, remove the debt and rebase the `liquidityIndex` to account for the value of the debt removed. This socializes the bad debt, and removes the race condition for suppliers to withdraw their liquidity as fast as possible.
2. Do not continue accruing interest on bad debt, and a corresponding reduction in supply interest should also be applied.





# Medium


# [M-1] `LiquidationLogic::_repayDebtTokens()` incorrectly assigns burnt debt tokens to `nextDebtShares`, resulting in incorrect interest rate updates

## Summary
In the liquidation flow `_repayDebtTokens()` treats the burnt debt shares as the total remaining debt shares. This error is carried into the interest rate update calculations which depend on accurate total debt amount in order to calculate usage ratios (sometimes called utilization).

## Root Cause
`LiquidationLogic::_repayDebtTokens()` is called in the [liquidation flow](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L161) and assigns the amount of debtShares burnt to the `vars.debtReserveCache.nextDebtShares` variable:

```javascript
  function _repayDebtTokens(
    DataTypes.ExecuteLiquidationCallParams memory params,
    LiquidationCallLocalVars memory vars,
    mapping(bytes32 => DataTypes.PositionBalance) storage balances,
    DataTypes.ReserveSupplies storage totalSupplies
  ) internal {
    uint256 burnt = balances[params.position].repayDebt(totalSupplies, vars.actualDebtToLiquidate, vars.debtReserveCache.nextBorrowIndex);
@>  vars.debtReserveCache.nextDebtShares = burnt;
  }
```

This is problematic because the liquidation flow then [passes](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L165) the `vars.debtReserveCache` object to the `ReserveLogic::updateInterestRates()` function.

The `updateInterestRate()` function then assumes `_cache.nextDebtShares` is the entire remaining debtShare balance as it is [used to calculate `vars.totalDebt`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L158).

The `vars.totalDebt` is then [passed](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L167) to `calculateInterestRates()` and treated as the remaining total debt for the purposes of utilization calculations. This can lead to wild fluctuations in interest rates for all suppliers and borrowers for this reserve on the pool.

`calculateInterestRates()` is a function from an out of scope library for this audit, but the bug has already occured in a function from an in-scope contract (`_repayDebtTokens()`).


## Internal pre-conditions
1. Any position gets liquidated
2. The amount of debt shares burnt in the liquidation flow cannot equal the amount of remaining debt shares for this pool and reserve asset.

## External pre-conditions
1. Collateral price fluction makes a position liquidatable

## Attack Path
1. Alice and Bob supply to and borrow from a pool
2. The price of their collateral declines, and Alice becomes liquidatable
3. Bob liquidates Alice, burning 1e18 debt tokens in the process
4. The pool's borrow rate is incorrectly calculated (shown in POC)

## Impact

- Any interest rate strategy that depends on reserve 'utilization ratios' will be calculated incorrectly as the `vars.totalDebt` passed does not represent the total remaining debt. Note both the borrowRate and supplyRates will be affected by this issue.
- The magnitude of the impact varies based on the relative error between the debt shares burnt in a liquidation and the total debt shares remaining for that pool and reserve. 


## PoC

The below coded POC shows the following scenario:
1. Alice and Bob supply to and borrow from a pool
2. The price of their collateral declines, and Alice becomes liquidatable
3. Bob liquidates Alice, burning 1e18 debt tokens in the process
4. The pool's borrow rate is incorrectly calculated at 1.655% (noting the test interest rate settings here https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/test/forge/core/pool/CorePoolTests.sol#L65).



```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {PoolLiquidationTest} from './PoolLiquidationTests.t.sol';
import {console2} from 'forge-std/src/Test.sol';

contract POC_LiquidationBreaksInterestRates is PoolLiquidationTest {

    function test_POC_LiquidationBreaksInterestRate() public {
        // 1. Alice and Bob supply to and borrow from a pool
        uint256 _mintAmountA = 10e18;
        uint256 _mintAmountB = 20e18;   // Bob has extra tokenB to liquidate Alice
        uint256 _supplyAmountA = 5e18;
        uint256 _supplyAmountB = 10e18; // Bob supplies more collateral than Alice
        uint256 _borrowAmountA = 1e18; 
        uint256 _borrowAmountB = 2e18;

        _mintAndApprove(alice, tokenA, _mintAmountA, address(pool)); 
        _mintAndApprove(bob, tokenB, _mintAmountB, address(pool)); 

        vm.startPrank(bob);
        pool.supplySimple(address(tokenB), bob, _supplyAmountB, 0);
        pool.borrowSimple(address(tokenB), bob, _borrowAmountB, 0);
        vm.stopPrank();

        vm.startPrank(alice);
        pool.supplySimple(address(tokenA), alice, _supplyAmountA, 0);
        pool.borrowSimple(address(tokenB), alice, _borrowAmountA, 0);
        vm.stopPrank();

        // 2. The price of their collateral declines, and Alice becomes liquidatable
        oracleA.updateAnswer(4.5e7);
        pool.forceUpdateReserves(); 

        // 3. Bob liquidates Alice, burning 1e18 debt tokens in the process
        vm.startPrank(bob);
        uint256 preLiquidationDebtShares = pool.getTotalSupplyRaw(address(tokenB)).debtShares;
        pool.liquidateSimple(address(tokenA), address(tokenB), pos, type(uint256).max); 
        uint256 postLiquidationDebtShares = pool.getTotalSupplyRaw(address(tokenB)).debtShares;
        vm.stopPrank();
        uint256 liquidatedDebtShares = preLiquidationDebtShares - postLiquidationDebtShares;
        assert(liquidatedDebtShares == 1e18);   // Alice's entire position was liquidated

        // 4. The pool's borrow rate is incorrectly calculated at 1.655% (noting the test interest rate settings here https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/test/forge/core/pool/CorePoolTests.sol#L65)
        uint256 poolRemainingTokenBLiquidity = tokenB.balanceOf(address(pool));

        uint256 expectedTotalRemainingDebt = postLiquidationDebtShares * pool.getReserveData(address(tokenB)).borrowIndex / 1e27;
        uint256 expectedBorrowUsageRatio = 1e27 * expectedTotalRemainingDebt / (expectedTotalRemainingDebt + poolRemainingTokenBLiquidity);

        uint256 actualTotalRemainingDebt = liquidatedDebtShares * pool.getReserveData(address(tokenB)).borrowIndex / 1e27;
        uint256 actualBorrowUsageRatio = 1e27 * actualTotalRemainingDebt / (actualTotalRemainingDebt + poolRemainingTokenBLiquidity);

        uint256 tokenBBorrowRate = pool.getReserveData(address(tokenB)).borrowRate;
        vm.assertApproxEqRel(tokenBBorrowRate, actualBorrowUsageRatio * 7 / 47, 0.01e18); // As a result of the bug, the tokenBBorrowRate is 11.111% * 7% / 47% ~ 1.655%
        vm.assertApproxEqRel(0.02979e27, expectedBorrowUsageRatio * 7 / 47, 0.01e18); // But the actual useage ratio is 20% so the tokenBBorrowRate should be 20% * 7% / 47% ~ 2.979%
    }

}
```

## Mitigation
- Assign the the remaining debt shares to `vars.debtReserveCache.nextDebtShares`.




# [M-2] Pool supply interest compounds randomly based on the frequency of `ReserveLogic::updateState()` calls, resulting in inconsistant and unexpected returns for suppliers

## Summary
`_updateIndexes()` considers the previously earned interest on supplies as principle on the next update, resulting in a random compounding effect on interest earned. Borrowers may be incentivized to call `forceUpdateReserve()` frequently if gas costs are low relative to the upside of compounding their interest. 

## Root Cause
Any time an action calls `ReserveLogic::updateState()`, [`ReserveLogic::_updateIndexes()` is called](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L91). Note this occurs before important state-changing pool actions as well as through public/external functions such as [`Pool::forceUpdateReserve()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/Pool.sol#L161-L165) and [`Pool::forceUpdateReserves()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/Pool.sol#L168-L172).

`_updateIndexes()` calculates lienar interest since the last update and uses this to [scale up the `_cache.currLiquidityIndex` and assigns to `_cache.nextLiquidityIndex`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L225-L227):

```javascript
  function _updateIndexes(DataTypes.ReserveData storage _reserve, DataTypes.ReserveCache memory _cache) internal {
    ... SKIP!...

    if (_cache.currLiquidityRate != 0) {
@>    uint256 cumulatedLiquidityInterest = MathUtils.calculateLinearInterest(_cache.currLiquidityRate, _cache.reserveLastUpdateTimestamp);
      _cache.nextLiquidityIndex = cumulatedLiquidityInterest.rayMul(_cache.currLiquidityIndex).toUint128();
      _reserve.liquidityIndex = _cache.nextLiquidityIndex;
    }

    ... SKIP!...
  }
```

This creates a compounding-effect on an intended simple interest calculated, based on the frequency it's called. This is due to the interest accumulated on the last refresh being considered principle on the next refresh.


## Internal pre-conditions
1. Pool functions normally with supply/borrow/repay calls

## External pre-conditions
N/A

## Attack Path
1. Supplier supplies assets
2. Borrower takes debt against supplier's assets and pays interest
3. Indexes are updated at some frequency
4. Supplier earns compound-like interest on their supplied assets 

## Impact
- Suppliers receive compound-interest on their supplied assets instead of the intended fixed/linear interest.

## PoC

Create a new file in /test/forge/core/pool and paste the below contents.

Run command `forge test --mt test_POC_InterestRateIndexCalc -vv` to see the following logs which show the more frequently the update, the more compounding-like the `liquidityIndex` becomes.

```javascript
Ran 3 tests for test/forge/core/pool/RandomCompoundingInterestPOC.t.sol:PoolSupplyRandomCompoundingEffect
[PASS] test_POC_InterestRateIndexCalc_1_singleUpdate() (gas: 740335)
Logs:
  Updates once after a year: liquidityIndex 1.333e27
  Updates once after a year: borrowIndex 1.446891914398940457716504e27

[PASS] test_POC_InterestRateIndexCalc_2_monthlyUpdates() (gas: 1094439)
Logs:
  Updates monthly for a year: liquidityIndex 1.388987876426245531179679454e27
  Updates monthly for a year: borrowIndex 1.447734004896004725108257228e27

[PASS] test_POC_InterestRateIndexCalc_3_dailyUpdates() (gas: 65326938)
Logs:
  Updates every four hours for a year: liquidityIndex 1.395111981380752339971733874e27
  Updates every four hours for a year: borrowIndex 1.447734611520782405003308237e27
```


```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {console2} from 'forge-std/src/Test.sol';
import {PoolLiquidationTest} from './PoolLiquidationTests.t.sol';

contract PoolSupplyRandomCompoundingEffect is PoolLiquidationTest {

  function test_POC_InterestRateIndexCalc_1_singleUpdate() public {
    _mintAndApprove(alice, tokenA, 10_000e18, address(pool)); // alice 10k tokenA
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool)); // bob 2000 tokenB

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 10_000e18, 0); // 10k tokenA alice supply
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0); // 750 tokenB bob supply
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, supplyAmountB, 0); // 750 tokenB borrow for 100% utilization
    vm.stopPrank();

    vm.warp(block.timestamp + 365 days);
    
    pool.forceUpdateReserves(); // updates indicies

    console2.log('Updates once after a year: liquidityIndex %e', pool.getReserveData(address(tokenB)).liquidityIndex);
    console2.log('Updates once after a year: borrowIndex %e', pool.getReserveData(address(tokenB)).borrowIndex);
  }

  function test_POC_InterestRateIndexCalc_2_monthlyUpdates() public {
    _mintAndApprove(alice, tokenA, 10_000e18, address(pool)); // alice 10k tokenA
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool)); // bob 2000 tokenB

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 10_000e18, 0); // 10k tokenA alice supply
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0); // 750 tokenB bob supply
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, supplyAmountB, 0); // 750 tokenB borrow for 100% utilization
    vm.stopPrank();

    for(uint256 i = 0; i < 12; i++) {
        vm.warp(block.timestamp + (30 * 24 * 60 * 60));
        pool.forceUpdateReserves(); // updates indicies
    }
    vm.warp(block.timestamp + (5 * 24 * 60 * 60)); // Final 5 days
    pool.forceUpdateReserves(); // updates indicies

    console2.log('Updates monthly for a year: liquidityIndex %e', pool.getReserveData(address(tokenB)).liquidityIndex);
    console2.log('Updates monthly for a year: borrowIndex %e', pool.getReserveData(address(tokenB)).borrowIndex);

  }

  function test_POC_InterestRateIndexCalc_3_dailyUpdates() public {
    _mintAndApprove(alice, tokenA, 10_000e18, address(pool)); // alice 10k tokenA
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool)); // bob 2000 tokenB

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 10_000e18, 0); // 10k tokenA alice supply
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0); // 750 tokenB bob supply
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, supplyAmountB, 0); // 750 tokenB borrow for 100% utilization
    vm.stopPrank();

    for(uint256 i = 0; i < (6 * 365); i++) {
        vm.warp(block.timestamp + (4 * 60 * 60));
        pool.forceUpdateReserves(); // updates indicies
    }

    console2.log('Updates every four hours for a year: liquidityIndex %e', pool.getReserveData(address(tokenB)).liquidityIndex);
    console2.log('Updates every four hours for a year: borrowIndex %e', pool.getReserveData(address(tokenB)).borrowIndex);
  }
}
```

## Mitigation
- Track the principle and the interest separately. Accrue simple interest on the principle only.





# [M-3] Unclaimable reserve assets will accrue in a pool due to the difference between interest paid on borrows and interest earned on supplies 

## Summary
The interest paid on borrows is calculated in a compounding fashion, but the interest earned on supplying assets is calculated in a fixed way. As a result more interest will be repaid by borrowers than is claimable by suppliers. This buildup of balance never gets rebased into the `liquidityIndex`, nor is it claimable with some sort of 'skim' function.

## Root Cause
Any time an action calls `ReserveLogic::updateState()`, [`ReserveLogic::_updateIndexes()` is called](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L91).

In `_updateIndexes()`, the `_cache.nextLiquidityIndex` is a [scaled up](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L226) version of `_cache.currLiquidityIndex` based on the 'linear interest' [calculated in `MathUtils::calculateLinearInterest()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L225).

[`calculateLinearInterest`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/utils/MathUtils.sol#L34-L42) scales a fixed interest annual interest rate by the amount of time elapsed since the last call:

```javascript
  function calculateLinearInterest(uint256 rate, uint40 lastUpdateTimestamp) internal view returns (uint256) {
    //solium-disable-next-line
    uint256 result = rate * (block.timestamp - uint256(lastUpdateTimestamp));
    unchecked {
      result = result / SECONDS_PER_YEAR;
    }

    return WadRayMath.RAY + result;
  }
```


Similarly, the `_cache.nextBorrowIndex` is a [scaled up](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L236) version of `_cache.currBorrowIndex` based on the 'compound interest' [calculated in `MathUtils::calculateCompoundedInterest()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L235).

[`calculateCompoundedInterest`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/utils/MathUtils.sol#L58C12-L89) compounds a rate based on the time elapsed since it was last called:

```javascript
  function calculateCompoundedInterest(uint256 rate, uint40 lastUpdateTimestamp, uint256 currentTimestamp) internal pure returns (uint256) {
    //solium-disable-next-line
    uint256 exp = currentTimestamp - uint256(lastUpdateTimestamp);

    if (exp == 0) {
      return WadRayMath.RAY;
    }

    uint256 expMinusOne;
    uint256 expMinusTwo;
    uint256 basePowerTwo;
    uint256 basePowerThree;
    unchecked {
      expMinusOne = exp - 1;
      expMinusTwo = exp > 2 ? exp - 2 : 0;

      basePowerTwo = rate.rayMul(rate) / (SECONDS_PER_YEAR * SECONDS_PER_YEAR);
      basePowerThree = basePowerTwo.rayMul(rate) / SECONDS_PER_YEAR;
    }

    uint256 secondTerm = exp * expMinusOne * basePowerTwo;
    unchecked {
      secondTerm /= 2;
    }
    uint256 thirdTerm = exp * expMinusOne * expMinusTwo * basePowerThree;
    unchecked {
      thirdTerm /= 6;
    }

    return WadRayMath.RAY + (rate * exp) / SECONDS_PER_YEAR + secondTerm + thirdTerm;
  }
```

As a result, more interest is payable on debt than is earned on supplied liquidity. This is a design choice by the protocol, however without a function to 'skim' this extra interest, these tokens will buildup and are locked in the protocol. 



## Internal pre-conditions
1. Pool operates normally with supplies/borrows/repays
2. `updateState()` must NOT be called every second, as this would create a compounding-effect on the 'linear rate' such that the difference in interest paid on debts is equal to the interest earned on supplies.


## External pre-conditions
N/A

## Attack Path
1. Several users supply tokens to a pool as normal
2. Users borrow against the liquidity
3. Time passes, all borrows are repaid
4. All suppliers withdraw their funds (as part of this operation the treasury also withdraws their fee assets)
5. A pool remains with 0 supplyShares and 0 debtShares, but still has a token balance which is unclaimable by anyone


## Impact
1. Token buildup in contract is unclaimable by anyone
   - The built up token balance can be borrowed and flash loaned, leading to compounding build up of unclaimable liquidity


## PoC

Create a new file in /test/forge/core/pool and paste the below contents. The test shows a simple supply/borrow/warp/repay flow. After the actions are complete, the pool has more `tokenB` than is claimable by the supplier and the treasury. These tokens are now locked in the contract 

```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {console2} from 'forge-std/src/Test.sol';
import {PoolLiquidationTest} from './PoolLiquidationTests.t.sol';

contract AuditUnclaimableBalanceBuildupOnPool is PoolLiquidationTest {

  function test_POC_UnclaimableBalanceBuildupOnPool () public {
    uint256 aliceMintAmount = 10_000e18;
    uint256 bobMintAmount = 10_000e18;
    uint256 supplyAmount = 1000e18;
    uint256 borrowAmount = 1000e18;

    _mintAndApprove(alice, tokenA, aliceMintAmount, address(pool));         // alice collateral
    _mintAndApprove(bob, tokenB, bobMintAmount, address(pool));             // bob supply
    _mintAndApprove(alice, tokenB, aliceMintAmount, address(pool));         // alice needs some funds to pay interest

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmount, 0); 
    vm.stopPrank();

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, aliceMintAmount, 0);  // alice collateral
    pool.borrowSimple(address(tokenB), alice, borrowAmount, 0);     // 100% utilization
    vm.stopPrank();

    vm.warp(block.timestamp + 365 days); // Time passes, interest accrues, treasury shares accrue
    pool.forceUpdateReserves();

    // Alice full repay
    vm.startPrank(alice);
    tokenB.approve(address(pool), type(uint256).max);
    pool.repaySimple(address(tokenB), type(uint256).max, 0);

    (,,, uint256 debtShares) = pool.marketBalances(address(tokenB));
    // All debt has been repaid
    assert(debtShares == 0); 

    // Bob's claim on pool's tokenB
    bytes32 bobPos = keccak256(abi.encodePacked(bob, 'index', uint256(0)));
    uint256 BobsMaxWithdrawAssets = pool.supplyShares(address(tokenB), bobPos) * pool.getReserveData(address(tokenB)).liquidityIndex / 1e27;

    // Treasury claim on pool's tokenB
    uint256 accruedTreasuryAssets = pool.getReserveData(address(tokenB)).accruedToTreasuryShares * pool.getReserveData(address(tokenB)).liquidityIndex / 1e27;

    // Total balance of pool's tokenB
    uint256 poolTokenBBalance = tokenB.balanceOf(address(pool));

    assert(poolTokenBBalance > BobsMaxWithdrawAssets + accruedTreasuryAssets); // There are more tokenB on the pool than all suppliers + treasury claim. 
  }

}
```

## Mitigation
One option is to create a function which claims the latent funds to the treasury, callable by an owner
- Calls `forceUpdateReserves()`
- Calls `executeMintToTreasury()`
- Calculates the latent funds on a pool's reserve (something like `tokenA.balanceOf(pool) - ( totalSupplyShares * liquidityIndex )`)
- Sends these funds to the treasury

Another option would be to occasionally rebase `liquidityIndex` to increase the value of supplyShares so supplies have a claim on these extra funds.

In both cases it may be sensible to leave some dust as a buffer. 


# [M-4] Supply interest is earned on `accruedToTreasuryShares` resulting in higher than expected treasury fees and under rare circumstances DOSed pool withdrawals

## Summary
Fees on debt interest are calculated in 'assets', but not claimed immediately and stored as shares. When they're claimed, they're treated as regular supplyShares and converted back to assets based on the `liquidityIndex` at the time they're claimed. This results in the realized fees being higher than expected, and under an extreme scenario may not leave enough liquidity for regular pool suppliers to withdraw their funds.

## Root Cause

Each time a pool's reserve state is updated, [`ReserveLogic::_accrueToTreasury()` is called](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L92).
This function incriments `_reserve.accruedToTreasuryShares` by the shares equiavalent of the assets taken as a fee. Note `vars.amountToMint` is in assets, and `_reserve.accruedToTreasuryShares` is stored in shares as the 'fee assets' are not always immediately sent to the treasury.

```javascript
  function _accrueToTreasury(uint256 reserveFactor, DataTypes.ReserveData storage _reserve, DataTypes.ReserveCache memory _cache) internal {
    ... SKIP!...

    vars.amountToMint = vars.totalDebtAccrued.percentMul(reserveFactor); // Assets

@>  if (vars.amountToMint != 0) _reserve.accruedToTreasuryShares += vars.amountToMint.rayDiv(_cache.nextLiquidityIndex).toUint128(); // Shares
  }
```

When any pool withdrawal is executed, [`PoolLogic::executeMintToTreasury()` is called](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/PoolSetters.sol#L84). The stored shares are converted back to assets based on the current `liquidityIndex`:

```javascript
  function executeMintToTreasury(
    DataTypes.ReserveSupplies storage totalSupply,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    address treasury,
    address asset
  ) external {
    DataTypes.ReserveData storage reserve = reservesData[asset];

    uint256 accruedToTreasuryShares = reserve.accruedToTreasuryShares;

    if (accruedToTreasuryShares != 0) {
      reserve.accruedToTreasuryShares = 0;
      uint256 normalizedIncome = reserve.getNormalizedIncome();
      uint256 amountToMint = accruedToTreasuryShares.rayMul(normalizedIncome); // Assets, scaled up by current liquidityIndex

      IERC20(asset).safeTransfer(treasury, amountToMint);
      totalSupply.supplyShares -= accruedToTreasuryShares;

      emit PoolEventsLib.MintedToTreasury(asset, amountToMint);
    }
  }
```

As a result, the actual fee taken by the treasury exceeds the original x% intended by `reserveFactor` at the time the debt was repaid.

In addition, the accumulation of interest on these `accruedToTreasuryShares` can lead to pool withdrawals being DOSed under circumstances where the `_updateIndexes()` is called close to every second to create a compounding effect. This compounding effect on brings the `liquidityIndex` closer to the `borrowIndex`. This results in the interest earned on `accruedToTreasuryShares` causing the pool to run out of liquidity when suppliers withdraw.


## Internal pre-conditions
Impact 1 
1. A pool has a non-zero `reserveFactor`
2. Pool operates normally with supplies/borrows/repays

Impact 2
1. A pool has a non-zero `reserveFactor`
2. Pool operates normally with supplies/borrows/repays
3. `_updateIndexes()` is called each second (or close to)

## External pre-conditions
N/A

## Attack Path
Impact 2
1. User1 supplies to a pool
2. User2 borrows from the same pool
3. As time elapses, `_updateIndexes()` is called close to every second, bringing `liquidityIndex` closer to `borrowIndex`. Note this is callable from an external function `Pool::forceUpdateReserves()`
4. User2 repays their borrow including interest
5. Repeat step 3 just for a few seconds
6. User1 attempts to withdraw their balance but due to the accrued interest on `accruedToTreasuryShares`, the pool runs out of liquidity DOSing the withdrawal.

## Impact
1. Generally speaking, in all pools the treasury will end up taking a larger fee than what was set in `reserveFactor`. That is, if `reserveFactor` is 1e3 (10%) and 1e18 interest is earned, the protocol will eventually claim more than 10% * 1e18 assets.
2. Under a specific scenario where `_updateIndexes()` is called every second, there will not be enough liquidity for suppliers to withdraw because the treasury earning supply interest on their `accruedToTreasuryShares` is not accounted for.


## PoC
The below coded POC implements the 'Attack Path' described above.

First, add this line to the `CorePoolTests::_setUpCorePool()` function to create a scenario with a non-zero reserve factor:

```diff
  function _setUpCorePool() internal {
    poolImplementation = new Pool();

    poolFactory = new PoolFactory(address(poolImplementation));
+   poolFactory.setReserveFactor(1e3); // 10%
    configurator = new PoolConfigurator(address(poolFactory));

    ... SKIP!...
  }
```

Then, create a new file on /test/forge/core/pool and paste the below contents.

```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {console2} from 'forge-std/src/Test.sol';
import {PoolLiquidationTest} from './PoolLiquidationTests.t.sol';

contract AuditHighSupplyRateDosWithdrawals is PoolLiquidationTest {

function test_POC_DosedWithdrawalsDueToTreasurySharesAccruing() public {
    uint256 aliceMintAmount = 10_000e18;
    uint256 bobMintAmount = 10_000e18;
    uint256 supplyAmount = 1000e18;
    uint256 borrowAmount = 1000e18;

    _mintAndApprove(alice, tokenA, aliceMintAmount, address(pool));         // alice collateral
    _mintAndApprove(bob, tokenB, bobMintAmount, address(pool));             // bob supply
    _mintAndApprove(alice, tokenB, aliceMintAmount, address(pool));         // alice needs some funds to pay interest

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmount, 0); 
    vm.stopPrank();

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, aliceMintAmount, 0);  // alice collateral
    pool.borrowSimple(address(tokenB), alice, borrowAmount, 0);     // 100% utilization
    vm.stopPrank();

    for(uint256 i = 0; i < (12 * 60 * 60); i++) { // Each second `_updateIndexes()` is called via external function `forceUpdateReserves()`
        vm.warp(block.timestamp + 1);
        pool.forceUpdateReserves();
    }

    // Alice full repay
    vm.startPrank(alice);
    tokenB.approve(address(pool), type(uint256).max);
    pool.repaySimple(address(tokenB), type(uint256).max, 0);
    uint256 post_aliceTokenBBalance = tokenB.balanceOf(alice);
    uint256 interestRepaidByAlice = aliceMintAmount - post_aliceTokenBBalance;

    for(uint256 i = 0; i < (60); i++) { // warp after for treasury to accrue interest on their 'fee shares' 
        vm.warp(block.timestamp + 1);
        pool.forceUpdateReserves();
    }

    // Check debt has been repaid
    (, , , uint256 debtShares) = pool.marketBalances(address(tokenB));
    assert(debtShares == 0); // All debt has been repaid

    // Treasury assets to claim
    uint256 treasuryAssetsToClaim = pool.getReserveData(address(tokenB)).accruedToTreasuryShares * pool.getReserveData(address(tokenB)).liquidityIndex / 1e27;

    // Bob's assets to claim
    bytes32 bobPos = keccak256(abi.encodePacked(bob, 'index', uint256(0)));
    uint256 bobsAssets = pool.supplyShares(address(tokenB), bobPos) * pool.getReserveData(address(tokenB)).liquidityIndex / 1e27;

    // Impact 1: the interest claimable by the treasury is greater than 10% of the interest repaid
    assert(treasuryAssetsToClaim > pool.factory().reserveFactor() * interestRepaidByAlice / 1e4);

    // Impact 2: Bob & the treasury's claim on the assets is greater than available assets, despite no outstanding debt. 
    // This assert demonstrates that bob's withdrawal would be DOSed as withdrawal calls include a transfer of treasury assets.
    // The withdrawal DOS cannot be shown due to the call reverting due to the 'share underflow' issue described in another report
    uint256 poolLiquidity = tokenB.balanceOf(address(pool));
    assert(bobsAssets + treasuryAssetsToClaim > poolLiquidity); 
  }
}
```

## Mitigation
Three possible solutions:
1. Immediately send the 'fee assets' to treasury rather than accruing them over time
2. Store the 'fee assets' in assets instead of shares. This will correctly capture the amount of fee that is intended by `reserveFactor`. For example if a fee is 10%, the protocol will take exactly 10% of the interest earned on debt.
3. Account for the creation of new `supplyShares` by diluting the `liquidityIndex` upon creating these shares. This solution will allow the conversion back to assets in `executeMintToTreasury()` to remain unchanged.
   - Note I acknowledge that the [calculation of `liquidityRate`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/periphery/ir/DefaultReserveInterestRateStrategy.sol#L126-L128) does account for `reserveFactor`, however given this is out of scope I did not focus on it. Regardless, it does not scale the rate down enough to account for the interest the treasury will earn on these shares. 



# [M-5] [Dont submit, impact is wrong] The liquidation flow inputs `vars.userCollateralBalance` to `LiquidationLogic::_calculateAvailableCollateralToLiquidate()` as supplyShares not 'assets', resulting in an incorrect amount of collateral liquidated

## Summary

Incorrectly inputting `vars.userCollateralBalance` as a value in shares instead of a value in assets will result in less than the intended amount of debt being liquidated, for a range of `debtToCover` values.

Note: this issue was submited separately to another issue: "`LiquidationLogic::_calculateDebt()` treats debt shares as assets which results in bad debt, loss of funds for liquidators, invalid health checks, and loss of protocol liquidation fees". Despite both issues being conceptually similar, the root cause and fix are both distinct. This issue also has lower impact than the other issue.

## Root Cause

During the liquidation flow `LiquidationLogic::executeLiquidationCall()` is called. This function calls `_calculateAvailableCollateralToLiquidate()` in order to determine the amount of collateral to liquidate. The amount of collateral that can be liquidated is calculated as a [function of the `debtToCover` and the price of the debt and collateral assets](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L352). As shown below this value is then scaled up by the `liquidationBonus` and assigned to `vars.maxCollateralToLiquidate`: 


```javascript
  function _calculateAvailableCollateralToLiquidate(
    DataTypes.ReserveData storage collateralReserve,
    DataTypes.ReserveCache memory debtReserveCache,
    uint256 debtToCover,
    uint256 userCollateralBalance,
    uint256 liquidationBonus,
    uint256 collateralPrice,
    uint256 debtAssetPrice,
    uint256 liquidationProtocolFeePercentage
  ) internal view returns (uint256, uint256, uint256) {

    ... SKIP!...

    // This is the base collateral to liquidate based on the given debt to cover
    vars.baseCollateral = ((vars.debtAssetPrice * debtToCover * vars.collateralAssetUnit)) / (vars.collateralPrice * vars.debtAssetUnit);
    // @note finds the amount of baseCollateral required such that: baseCollateral * normalizedCollateralPrice =  debtToCover * normalizedDebtPrice 

    vars.maxCollateralToLiquidate = vars.baseCollateral.percentMul(liquidationBonus);

@>  if (vars.maxCollateralToLiquidate > userCollateralBalance) {
      vars.collateralAmount = userCollateralBalance;
      vars.debtAmountNeeded = (
        (vars.collateralPrice * vars.collateralAmount * vars.debtAssetUnit) / (vars.debtAssetPrice * vars.collateralAssetUnit)
      ).percentDiv(liquidationBonus);
    } else {
      vars.collateralAmount = vars.maxCollateralToLiquidate;
      vars.debtAmountNeeded = debtToCover;
    }

    ... SKIP!...
  }
```

There will be a range of input values of `debtToCover` for which, due to the bug, the `(vars.maxCollateralToLiquidate > userCollateralBalance)` will incorrectly evaluate as true and therefore the `debtToCover` will be scaled down when assigned to `vars.debtAmountNeeded`. As a result liquidators will liquidate less debt than they intended.

## Internal pre-conditions
1. Interest needs to have accrued on a debt position. This is trivial because `ReserveLogic::updateState()` is called at the begining of the liquidation flow.

## External pre-conditions
1. Collateral price fluction makes a position liquidatable

## Attack Path
1. Liquidator calls `Pool::liquidate()` or `Pool::liquidateSimple()` on a liquidatable posistion

## Impact
The impact varies depending on how the input variables `debtToCover` and `userCollateralBalance` are denominated:

**Current state** - `debtToCover` and `userCollateralBalance` denominated in shares:
- Debt will be priced incorrectly and collateral that gets seized will be lower value than debt that gets seized. As described in the '`_calculateDebt` issue', `borrowIndex` compounds so it will nearly always be larger `liquidityIndex` as this scales based on fixed interest. As a result the value of `vars.baseCollateral` will be reduced and can be lower than the value of `debtToCover` in the case where the difference between the indexes outweighs the `liquidationBonus`, ie. liquidators lose value by liquidating. Note this is the same impact as the other issue but different root cause.

**With proposed fix in the `_calculateDebt` issue** - `debtToCover` in assets and `userCollateralBalance` in shares.
- The [inequalty check](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L356) compares a value denominated in assets to a value denominated in shares. There will be a range of `debtToCover` input values which will result in an unwanted and unnecessary reduction in collateral assets being seized and debt being liquidated.

## PoC
A coded PoC. Required in some cases but optional for other cases.

## Mitigation
Convert the [`vars.userCollateralBalance`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L136) to assets instead of shares. 


# [M-6] `CuratedVault::maxDeposit()` breaks ERC4626 complaiance as it can return a value that would revert if used for a `deposit()` call

## Summary
The readme specifies that the Curated Vault contract should follow the ERC4626 standard. However there are two scenarios where `maxDeposit()` can return a value greater than what would be accepted as a deposit. 

## Root Cause

**Root cause 1**
The Curated Vault's `supplyQueue` is an ordered array of pools which dictate the order in which vault deposits are deposited into the underlying pools. `supplyQueue` can be set by an 'allocator' through the `CuratedVault::setSupplyQueue()` function, which has no checks that the new supply queue does not contain duplicate pools. This fact was acknowledged by the devs in a [relavent natspec comment](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/interfaces/vaults/ICuratedVaultBase.sol#L165-L167). 

The issue is in order for the vault to be ERC4626 compliant, the `maxDeposit()` function cannot return a value that would revert if used in a `deposit()` call in the same transaction (https://eips.ethereum.org/EIPS/eip-4626#maxdeposit). However `maxDeposit()` makes a call to `_maxDeposit()` which iterates through `supplyQueue` cumulatively sums the assets suppliable and returns this value as 'max deposit'. Note if `supplyQueue` contained a duplicate pool, their 'suppliable' amount would be counted twice.


```javascript
  function _maxDeposit() internal view returns (uint256 totalSuppliable) {
@>  for (uint256 i; i < supplyQueue.length; ++i) {
      IPool pool = supplyQueue[i];

      uint256 supplyCap = config[pool].cap;
      if (supplyCap == 0) continue;

      uint256 supplyShares = pool.supplyShares(asset(), positionId);
      (uint256 totalSupplyAssets, uint256 totalSupplyShares,,) = pool.marketBalances(asset());

      // `supplyAssets` needs to be rounded up for `totalSuppliable` to be rounded down.
      uint256 supplyAssets = supplyShares.toAssetsUp(totalSupplyAssets, totalSupplyShares);
@>    totalSuppliable += supplyCap.zeroFloorSub(supplyAssets);
    }
  }
```

**Root cause 2**
`_maxDeposit()` considers the `supplyCap` for this pool set by the curated vault but does not consider the underlying pool's supply cap. As a result, there are circumstances where the underlying pools are nearing their supply cap but the maximum deposit value will be based on the `supplyCap` mapping on the curated vault contract only. This is another way `maxDeposit()` may return more assets than would actually be accepted in a deposit.

**Root cause 3**
Finally, `_maxDeposit()` does not check whether the reserve asset that would be supplied to a pool in the `supplyQueue` is frozen. If the curated vault attempts to supply to a vault that has the supply asset frozen, the deposit will [revert](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ValidationLogic.sol#L78-L79).


## Internal pre-conditions
Scenario 1
- Duplicate pools in `supplyQueue`
- The duplicated pool cannot be at the cap

Scenario 2
- The the sum of the difference between the vault's `supplyCap` and vault's assets in the underlying pools exceeds the difference between the underlying pool cap and total liquidity in the pools.

Scenario 3
- One or more of the pools in `supplyQueue` have the reserve asset frozen.

## External pre-conditions
N/A

## Attack Path
1. A user wishes to deposit the maximum funds into a curated vault which is performing well. 
2. They call make a call to `deposit` and set `assets` to the value returned by a prior call to `maxDeposit()`.
3. Due to one of the reasons described above, their deposit reverts


## Impact
1. Internal integrations (such as Pool hooks) and external integrations will be broken.
2. A failure of compliance with ERC4626 which contradicts the contest readme.

Please note the impact of this issue also exists in the `CuratedVault::maxMint()` function which utilizes that same `_maxDeposit()` function.

## PoC
N/A

## Mitigation
- Do not allow duplicate pools to be added to the `supplyQueue`, there doesn't seem to be a valid reason for an allocator to duplicate pools
- In `CuratedVaultGetters::_maxDeposit()` add the minimum of the available supply Cap in the underlying pool and the available `supplyCap` on the curated vault contract to the `totalSuppliable` variable.



# [M-7] `CuratedVaultSetters::_supplyPool()` does not consider the pool cap of the underlying pool, which may cause `deposit()` to revert or lead to an unintended reordering of `supplyQueue`

## Summary
The curated vault's `_supplyPool()` function deposits assets into the underlying pools in `supplyQueue`. Whilst it considers the curated vault's cap for a given pool, it does not consider the underlying pool's internal cap. As a result, some `CuratedVault::deposit()` transactions will revert due to running out of pools to deposit to, or the liquidity will be allocated to the `supplyQueue` in a different order. 

## Root Cause

`CuratedVaultSetters::_supplyPool()` is [called](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L38) in the deposit flow of the curated vault. As shown below, it attempts to deposit the minimum of `assets` and `supplyCap - supplyAssets` (which is the 'available curated pool cap' for this pool). 

```javascript
  function _supplyPool(uint256 assets) internal {
    for (uint256 i; i < supplyQueue.length; ++i) {
      IPool pool = supplyQueue[i];

      uint256 supplyCap = config[pool].cap;
      if (supplyCap == 0) continue;

      pool.forceUpdateReserve(asset());

      uint256 supplyShares = pool.supplyShares(asset(), positionId);

      // `supplyAssets` needs to be rounded up for `toSupply` to be rounded down.
      (uint256 totalSupplyAssets, uint256 totalSupplyShares,,) = pool.marketBalances(asset());
@>    uint256 supplyAssets = supplyShares.toAssetsUp(totalSupplyAssets, totalSupplyShares);

@>    uint256 toSupply = UtilsLib.min(supplyCap.zeroFloorSub(supplyAssets), assets);

      if (toSupply > 0) {
        // Using try/catch to skip markets that revert.
        try pool.supplySimple(asset(), address(this), toSupply, 0) {
          assets -= toSupply;
        } catch {}
      }

      if (assets == 0) return;
    }

    if (assets != 0) revert CuratedErrorsLib.AllCapsReached();
  }
```

However, the function does not consider an underlying pool's `supplyCap`. Underlying pools have their own `supplyCap` which will cause `supply()` calls to revert if they would put the pool over it's `supplyCap`:

```javascript
  function validateSupply(
    DataTypes.ReserveCache memory cache,
    DataTypes.ReserveData storage reserve,
    DataTypes.ExecuteSupplyParams memory params,
    DataTypes.ReserveSupplies storage totalSupplies
  ) internal view {
    ... SKIP!...

    uint256 supplyCap = cache.reserveConfiguration.getSupplyCap();

    require(
      supplyCap == 0
@>      || ((totalSupplies.supplyShares + uint256(reserve.accruedToTreasuryShares)).rayMul(cache.nextLiquidityIndex) + params.amount)
          <= supplyCap * (10 ** cache.reserveConfiguration.getDecimals()),
      PoolErrorsLib.SUPPLY_CAP_EXCEEDED
    );
  }
```

Also note that the `pool::supplySimple()` call in `_supplyPool()` is wrapped in a [try/catch block](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L134-L136), so if a `pool::supplySimple()` call were to revert, it will just continue to the next pool.

## Internal pre-conditions
- A `CuratedVault::deposit()` call needs to be within the limits of the curated vault's cap (`config[pool].cap`) but exceed the limits of the underlying pool's `supplyCap`.

## External pre-conditions
N/A

## Attack Path
1. A curated pool has two pools in the `supplyQueue`.
2. The first underlying pool has an internal `supplyCap` of 100e18 and is currently at 99e18.
3. The first underlying pool has an internal `supplyCap` of 100e18 and is currently at 99e18.
4. A user calls `deposit()` on the curated vault with a value of 2e18.
5. The value does not exceed the curated vault's `config[pool].cap` for either pool.
6. The underlying call to `Pool::supplySimple()` will silently revert on both pools, and the entire transaction will revert due to [running out of available pools to supply the assets to](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L142)
7. As a result, no assets are deposits, despite the underlying pools having capacity to accept the 2e18 deposit between them.

## Impact
**Deposit Reverts**
- If a deposit would be able to be deposited across two or more underlying pools in the `supplyQueue`, but is too large to be added to any one of these underlying pools, the deposit will completely revert, despite the underlying pools having capacity to accept the deposit.

**Inefficient reorder of `supplyQueue`**
- If a deposit amount is within the limits of the curated pool's `config[pool].cap`, but would exceed the limits an underlying pool in `supplyQueue`. Then the silent revert would skip this pool and attempt to deposit it's liquidity to the next pool in the queue. This is an undesired/inefficient reordering of the `supplyQueue` as a simple check on the cap of the underlying pool would reveal some amount that would be accepted by the underlying pool.

## PoC
N/A

## Mitigation

Create an external getter for a pool's supply cap, similar to `ReserveConfiguration::getSupplyCap()` the next function should also scale the supply cap by the reserve token's decimals. 

Then, add an extra check in `CuratedVaultSetters::_supplyPool()` as shown below.


```diff
  function _supplyPool(uint256 assets) internal {
    for (uint256 i; i < supplyQueue.length; ++i) {
      IPool pool = supplyQueue[i];

      uint256 supplyCap = config[pool].cap;
      if (supplyCap == 0) continue;

      pool.forceUpdateReserve(asset());

      uint256 supplyShares = pool.supplyShares(asset(), positionId);

      // `supplyAssets` needs to be rounded up for `toSupply` to be rounded down.
      (uint256 totalSupplyAssets, uint256 totalSupplyShares,,) = pool.marketBalances(asset());
      uint256 supplyAssets = supplyShares.toAssetsUp(totalSupplyAssets, totalSupplyShares);

      uint256 toSupply = UtilsLib.min(supplyCap.zeroFloorSub(supplyAssets), assets);

+     toSupply = UtilsLib.min(toSupply, pool.getSupplyCap(pool.getConfiguration(asset())) - totalSupplyAssets );

      if (toSupply > 0) {
        // Using try/catch to skip markets that revert.
        try pool.supplySimple(asset(), address(this), toSupply, 0) {
          assets -= toSupply;
        } catch {}
      }

      if (assets == 0) return;
    }

    if (assets != 0) revert CuratedErrorsLib.AllCapsReached();
  }
```


# [M-8] Curated Vault allocators cannot `reallocate()` a pool to zero due to attempting to withdraw 0 tokens from the underlying pool

## Summary
The `reallocate()` function is the primary way a curated vault can remove liquidity from an underlying pool, so being unable to fully remove the liquidity is problematic. However, due to a logic issue in the implementation, any attempt to `reallocate()` liquidity in a pool to zero will revert. 

## Root Cause

The function `CuratedVault::reallocate()` is callable by an allocator and reallocates funds between underlying pools. As shown below, `toWithdraw` is the difference between `supplyAssets` (total assets in the underlying pool controlled by the curated vault) and the `allocation.assets` which is the target allocation for this pool. Therefore, if `toWithdraw` is greater than zero, a withdrawal is required from that pool:

```javascript
  function reallocate(MarketAllocation[] calldata allocations) external onlyAllocator {
    uint256 totalSupplied;
    uint256 totalWithdrawn;

    for (uint256 i; i < allocations.length; ++i) {
      MarketAllocation memory allocation = allocations[i];
      IPool pool = allocation.market;

      (uint256 supplyAssets, uint256 supplyShares) = _accruedSupplyBalance(pool);
@>    uint256 toWithdraw = supplyAssets.zeroFloorSub(allocation.assets);

      if (toWithdraw > 0) {
        if (!config[pool].enabled) revert CuratedErrorsLib.MarketNotEnabled(pool);

        // Guarantees that unknown frontrunning donations can be withdrawn, in order to disable a market.
        uint256 shares;
        if (allocation.assets == 0) {
          shares = supplyShares;
@>        toWithdraw = 0;
        }

@>      DataTypes.SharesType memory burnt = pool.withdrawSimple(asset(), address(this), toWithdraw, 0);
        emit CuratedEventsLib.ReallocateWithdraw(_msgSender(), pool, burnt.assets, burnt.shares);
        totalWithdrawn += burnt.assets;
      } else {
        
        ... SKIP!...
      }
    }
```

The issue arrises when for any given pool, `allocation.assets` is set to 0, meaning the allocator wishes to empty that pool and allocate the liquidity to another pool. Under this scenario, `toWithdraw` is set to 0, and passed into `Pool::withdrawSimple()`. This is a logic mistake to attempt to withdraw 0 instead of the `supplyAssets` when attempting to withdraw all liquidity to a pool. The call will revert due to a [check](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ValidationLogic.sol#L97) in the pool's withdraw flow that ensures the amount being withdrawn is greater than 0.

It seems this bug is a result of forking the Metamorpho codebase which [implements the same logic](https://github.com/morpho-org/metamorpho/blob/cc6b01610c9b0000d965faef705e1670859d9c0f/src/MetaMorpho.sol#L382-L389). However, Metamorpho's underlying withdraw() function [can take either an asset amount or share amount](https://github.com/morpho-org/morpho-blue/blob/0448402af51b8293ed36653de43cbee8d4d2bfda/src/Morpho.sol#L216-L217), but ZeroLend's `Pool::withdrawSimple()` only [accepts an amount of assets](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/Pool.sol#L94). 


## Internal pre-conditions
1. A curated vault having more than 1 underlying pool in the `supplyQueue`

## External pre-conditions
N/A

## Attack Path
1. A curated vault is setup with more than 1 underlying pool in the `supplyQueue`
2. An allocator wishes to reallocate all the supplied liquidity from one pool to another
3. The allocator calls constructs the array of `MarketAllocation` withdrawing all liquidity from the first pool and depositing the same amount to the second pool
4. The action fails due to the described bug


## Impact
- Allocators cannot remove all liquidity in a pool throuhg the `reallocate()` function. The natspec comments indicate that emptying a pool to zero through the `reallocate()` function is the [first step of the intended way to remove a pool](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/interfaces/vaults/ICuratedVaultBase.sol#L141-L142).  

## PoC

Paste and run the below test in /test/forge/core/vaults/. It shows a simple 2-pool vault with a single depositors. An allocator attempts to reallocate the liquidity in the first pool to the second pool, but reverts due to the described issue.

```javascript
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity ^0.8.0;

import {Math} from '@openzeppelin/contracts/utils/math/Math.sol';
import './helpers/IntegrationVaultTest.sol';
import {console2} from 'forge-std/src/Test.sol';
import {MarketAllocation} from '../../../../contracts/interfaces/vaults/ICuratedVaultBase.sol';

contract POC_AllocatorCannotReallocateToZero is IntegrationVaultTest {
  function setUp() public {
    _setUpVault();
    _setCap(allMarkets[0], CAP);
    _sortSupplyQueueIdleLast();

    oracleB.updateRoundTimestamp();
    oracle.updateRoundTimestamp();
  }

  function test_POC_AllocatorCannotReallocateToZero() public {

    // 1. supplier supplies to underlying pool through curated vault
    uint256 assets = 10e18;
    loanToken.mint(supplier, assets);

    vm.startPrank(supplier);
    uint256 shares = vault.deposit(assets, onBehalf);
    vm.stopPrank();

    // 2. Allocator attempts to reallocate all of these funds to another pool
    MarketAllocation[] memory allocations = new MarketAllocation[](2);
    
    allocations[0] = MarketAllocation({
        market: vault.supplyQueue(0),
        assets: 0
    });

    // Fill in the second MarketAllocation struct
    allocations[1] = MarketAllocation({
        market: vault.supplyQueue(1),
        assets: assets
    });

    vm.startPrank(allocator);
    vm.expectRevert("NOT_ENOUGH_AVAILABLE_USER_BALANCE");
    vault.reallocate(allocations); // Reverts due to attempting a withdrawal amount of 0
  }
}
```

## Mitigation

The comment "Guarantees that unknown frontrunning donations can be withdrawn, in order to disable a market" does not seem to apply to Zerolend pools as a donation to the pool would not disable the market, nor would it affect the amount of assets the curated vault can withdraw from the underlying pool. With this in mind, Metamorpho's safety check can be removed:

```diff
  function reallocate(MarketAllocation[] calldata allocations) external onlyAllocator {
    uint256 totalSupplied;
    uint256 totalWithdrawn;


    for (uint256 i; i < allocations.length; ++i) {
      MarketAllocation memory allocation = allocations[i];
      IPool pool = allocation.market;

      (uint256 supplyAssets, uint256 supplyShares) = _accruedSupplyBalance(pool);
      uint256 toWithdraw = supplyAssets.zeroFloorSub(allocation.assets);

      if (toWithdraw > 0) {
        if (!config[pool].enabled) revert CuratedErrorsLib.MarketNotEnabled(pool);


-       // Guarantees that unknown frontrunning donations can be withdrawn, in order to disable a market.
-       uint256 shares;
-       if (allocation.assets == 0) {
-         shares = supplyShares;
-         toWithdraw = 0;
-       }

        DataTypes.SharesType memory burnt = pool.withdrawSimple(asset(), address(this), toWithdraw, 0);
        emit CuratedEventsLib.ReallocateWithdraw(_msgSender(), pool, burnt.assets, burnt.shares);
        totalWithdrawn += burnt.assets;
      } else {
        uint256 suppliedAssets =
          allocation.assets == type(uint256).max ? totalWithdrawn.zeroFloorSub(totalSupplied) : allocation.assets.zeroFloorSub(supplyAssets);

        if (suppliedAssets == 0) continue;

        uint256 supplyCap = config[pool].cap;
        if (supplyCap == 0) revert CuratedErrorsLib.UnauthorizedMarket(pool);

        if (supplyAssets + suppliedAssets > supplyCap) revert CuratedErrorsLib.SupplyCapExceeded(pool);

        // The market's loan asset is guaranteed to be the vault's asset because it has a non-zero supply cap.
        IERC20(asset()).forceApprove(address(pool), type(uint256).max);
        DataTypes.SharesType memory minted = pool.supplySimple(asset(), address(this), suppliedAssets, 0);
        emit CuratedEventsLib.ReallocateSupply(_msgSender(), pool, minted.assets, minted.shares);
        totalSupplied += suppliedAssets;
      }
    }

    if (totalWithdrawn != totalSupplied) revert CuratedErrorsLib.InconsistentReallocation();
  }
```


# [M-9] Claiming dust buildup in `NFTRewardsDistributor` will remove unclaimed rewards for all users

## Summary
Dust builds up on the `NFTRewardsDistributor` contract. The impact is lower for tokens with higher precision, however as meantioned in the readme, USDC is in scope which has 6 decimals of precision. This leads to a non-trivial buildup of dust across the possible `assetHash` that may be rewarded. This dust buildup is supposed to be claimable through the `sweep()` function, however this function transfers all assets off the contract including unclaimed rewards, causing user `getReward()` calls to revert.

## Root Cause

Precision loss occurs on the two lines indicated of [`notifyRewardAmount`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L125-L137) below. Some amount of `reward` is divided by the `rewardDuration` and the result is rounded down, leaving the dust amount to accrue on the contract. This value will become significant across all possible `_assetHash` as these are unique for a given pool/asset/isDebt combination, with each requiring their own reward amount.

```javascript
  function notifyRewardAmount(uint256 reward, address pool, address asset, bool isDebt) external onlyRole(REWARDS_ALLOCATOR_ROLE) {
    rewardsToken.transferFrom(msg.sender, address(this), reward);

    bytes32 _assetHash = assetHash(pool, asset, isDebt);
    _updateReward(0, _assetHash);

    if (block.timestamp >= periodFinish[_assetHash]) {
@>    rewardRate[_assetHash] = reward.div(rewardsDuration);
    } else {
      uint256 remaining = periodFinish[_assetHash].sub(block.timestamp);
      uint256 leftover = remaining.mul(rewardRate[_assetHash]);
@>    rewardRate[_assetHash] = reward.add(leftover).div(rewardsDuration);
    }
```

Over time, the admin would like to sweep this dust off the contract, so they call [`NFTPositionManager::sweep()`](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTPositionManager.sol#L131-L139), which transfers all assets off the contract including **all* unclaimed rewards across all pools/assets/users.

```javascript
  function sweep(address token) external onlyRole(DEFAULT_ADMIN_ROLE) {
    if (token == address(0)) {
      uint256 bal = address(this).balance;
      payable(msg.sender).transfer(bal);
    } else {
      IERC20Upgradeable erc20 = IERC20Upgradeable(token);
      erc20.safeTransfer(msg.sender, erc20.balanceOf(address(this)));
    }
  }
```


## Internal pre-conditions
- Admin sweeps dust buildup of any `rewardToken`

## External pre-conditions
N/A

## Attack Path
- After some time, significant dust has built up on the `NFTRewardsDistributor` contract, and the admin would like to sweep it.
- Admin calls `sweep()`
- All pending rewards including ongoing reward distributions and previously completed but unclaimed reward distributions are removed, along with the dust.
- Any users who had pending rewards will not be able to claim them, as `getReward()` will revert.


## Impact
- The impact is variable as it depends on when how many unclaimed rewards existed when `sweep()` was called. 
- There may never be a fair time to call `sweep()`:
  - To have no impact on stakers, it must be called when no ongoing rewards are being distributed across all pools/assets/isDebt combinations, and all pending rewards have been claimed by all users. 
  - It seems like this condition may never be met and therefore the admin is left with a choice to forfeit the dust or remove unclaimed rewards from users.



## PoC

Paste the following test in /test/forge/core/positions/. It shows an admin attempting to sweep dust off the contract, but inadvertently removing Alice's rewards

```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {NFTPostionManagerTest} from './NFTPositionManagerTest.t.sol';
import {DataTypes} from 'contracts/core/pool/configuration/DataTypes.sol';
import {INFTPositionManager} from 'contracts/interfaces/INFTPositionManager.sol';
import {NFTErrorsLib} from 'contracts/interfaces/errors/NFTErrorsLib.sol';
import {console2} from 'forge-std/src/Test.sol';

contract Audit_ClaimingDustRemovesPendingRewards is NFTPostionManagerTest {

  function test_POC_ClaimingDustRemovesPendingRewards() external {
    testShouldSupplyAlice();
    uint256 repayAmount = 10 ether;
    uint256 borrowAmount = 20 ether;
    DataTypes.ExtraData memory data = DataTypes.ExtraData(bytes(''), bytes(''));
    INFTPositionManager.AssetOperationParams memory params =
      INFTPositionManager.AssetOperationParams(address(tokenA), alice, borrowAmount, 1, data);

    vm.startPrank(alice);
    nftPositionManager.borrow(params);
    assertEq(tokenA.balanceOf(address(pool)), 30 ether, 'Pool Revert');
    assertEq(tokenA.balanceOf(alice), 70 ether, 'Alice Revert');
    vm.stopPrank();

    // Setup reward distribution
    address rewarder = makeAddr('Rewarder');
    vm.startPrank(owner);
    nftPositionManager.grantRole(bytes32(keccak256('REWARDS_ALLOCATOR_ROLE')), rewarder);
    vm.stopPrank();

    // Rewarder allocates
    uint256 rewardAmount = 10e18;
    _mintAndApprove(rewarder, tokenA, rewardAmount, address(nftPositionManager)); 
    vm.startPrank(rewarder);
    nftPositionManager.notifyRewardAmount(rewardAmount, address(pool), address(tokenA), true);
    vm.stopPrank();

    // Time passes
    vm.warp(block.timestamp + 7 days);

    // Halfway through Alice's reward distribution, admin sweeps
    vm.startPrank(owner);
    nftPositionManager.sweep(address(tokenA));
    vm.stopPrank();

    vm.warp(block.timestamp + 7 days);

    // Alice claims
    vm.startPrank(alice);
    bytes32 assetHash = keccak256(abi.encode(address(pool), address(tokenA), true));
    vm.expectRevert('ERC20: transfer amount exceeds balance'); // Alice can't claim as their rewards were swept
    nftPositionManager.getReward(1, assetHash);
    vm.stopPrank();

  }
}
```

## Mitigation
- Keep internal accounting of the accumulated dust as a separate balance from the `rewardToken.balanceOf()`
- Adjust the `sweep()` function to only send the accumulated dust, not all reward tokens.



# [M-10] Lack of guardrails in the system significantly increases the likelihood of bad debt occuring

## Summary

Whilst `PoolLogic::setReserveConfiguration()` does implement some sanity checks on `LTV`, `liquidationThreshold`, and `LiquidationBonus`, there is little protection against the occurance of bad debt within the system. 

## Root Cause

Upon pool creation, the  `Pool::initialize()` is [invoked](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/PoolFactory.sol#L86) from the Pool Factory contract. This function [calls](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/Pool.sol#L42-L52) `PoolLogic::executeInitReserve()`, which [calls](https://github.com/sherlock-audit/2024-06-new-scope-Nihavent/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/PoolLogic.sol#L65) `PoolLogic::setReserveConfiguration()`. 


In this function shown below:
1. There is a check that `liquidationThreshold` is greater than or equal to `LTV`. The issue is the closer these values are the closer an opened position can be to immediately liquidation.
2. There is a check that `liquidationBonus` is greater than 100%, however this should be much higher to ensure liquidating small positions remains profitable (especially on mainnet).


```javascript
  function setReserveConfiguration(
    mapping(address => DataTypes.ReserveData) storage _reserves,
    address asset,
    address rateStrategyAddress,
    address source,
    DataTypes.ReserveConfigurationMap memory config
  ) public {
    ... SKIP!...

@>  require(config.getLtv() <= config.getLiquidationThreshold(), PoolErrorsLib.INVALID_RESERVE_PARAMS); // 1

    if (config.getLiquidationThreshold() != 0) {
      // liquidation bonus must be bigger than 100.00%, otherwise the liquidator would receive less
      // collateral than needed to cover the debt
@>    require(config.getLiquidationBonus() > PercentageMath.PERCENTAGE_FACTOR, PoolErrorsLib.INVALID_RESERVE_PARAMS); // 2

      // if threshold * bonus is less than PERCENTAGE_FACTOR, it's guaranteed that at the moment
      // a loan is taken there is enough collateral available to cover the liquidation bonus
      require(
        config.getLiquidationThreshold().percentMul(config.getLiquidationBonus()) <= PercentageMath.PERCENTAGE_FACTOR,
        PoolErrorsLib.INVALID_RESERVE_PARAMS
      );

    ... SKIP!...
    }
  }
```

## Internal pre-conditions
N/A

## External pre-conditions
N/A

## Attack Path
N/A


## Impact
1. Bad debt will be fragmented all over the protocol
2. A borrow can leave a user on the brink of liquidation, and importantly all of the debt cannot be liquidated profitably by the liquidators (ie. all collateral will be seized, but debt will be outstanding)
3. Small positions won't be liquidated, especially on mainnet

## PoC
A coded PoC. Required in some cases but optional for other cases.

## Mitigation
- Implement a mandatory buffer between `liquidationThreshold` and `LTV` to eliminate the possibility of borrowers being liquidated upon borrowing, resulting in bad debt. The larger the required buffer the less bad debt the protocol will incur. 
- Change the `liquidationBonus` to a fixed + small variable component. This ensures there is incentive to liquidate small positions, especially on mainnet.

