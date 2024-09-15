

### High



### Med

### `liquidatorReward` calculated incorrectly resulting in liquidator receiving dust fees

## Links to affected code
https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Liquidate.sol#L90-L100

https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Liquidate.sol#L119

## Bug Description

For profitable liquidations, liquidations are incentivised by receiving a `liquidatorReward` on top of the `debtInCollateralToken`.

As shown below, the `liquidatorReward` is the minimum of:
    - The excess collateral assigned to the loan 
    - `state.feeConfig.liquidationRewardPercent` of the future value of the loan (5%)

The bug occurs because the `liquidationRewardPercent` is multiplied by the `debtPosition.futureValue` which is denominated in 'borrow tokens' which have the value and decimals of USDC:

```javascript
    if (assignedCollateral > debtInCollateralToken) {
        uint256 liquidatorReward = Math.min(
            assignedCollateral - debtInCollateralToken, // nearly impossible for this to be the min
@>          Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT) // debtPosition.futureValue is in borrowToken decimals and value
        );
```
The resulting `liquidatorReward` is then added to `debtInCollateralToken` which is denominated in collateral tokens which have the value and decimals of WETH:

```javascript
@>      liquidatorProfitCollateralToken = debtInCollateralToken + liquidatorReward;
```

Due to USDC having lower value and decimals than WETH, the liquidator recieves less than they should when the collateral tokens get transfered:

```javascript
    state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, liquidatorProfitCollateralToken);
```

## Impact
- Liquidation keeper bots will receive dust as fees for liquidating, the `feeRecipient` will receive slightly more, but not enough to offset the reduction the bots receive.
- In the case where bots are down, liquidators are not incentivised to liquidate because the liquidation fee they receive will not cover their gas fees. 
- Liquidations will revert when reasonable values of `minimumCollateralProfit` are passed due to the check performed in `validateMinimumCollateralProfit()`.
- This issue also impacts calls to `liquidateWithReplacement()` due to this function calling  `executeLiquidate()`. This has notably less impact, because `liquidateWithReplacement()` is only callable by keepers which are the same entity as `feeRecipient`.
- Due to decimals, it is extremely difficult for `assignedCollateral - debtInCollateralToken` to be lower than `Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)` meaning a liquidator will not receive the excess assigned collateral in circumstances where they should (they instead always receive the dust value).

## Proof of Concept

As a simple example, lets consider a standard profitble liquidation with the following parameters:
- Price of WETH is 3000 USDC
- Future value of a loan is 100 USDC
- assignedCollateral = 0.04e18 WETH (120 USDC value)
- debtInCollateralToken = 0.03333e18 WETH (100 USDC value)

- With the bug:
    - Liquidator gets 0.03333e18 WETH (100 USDC) + dust
    - feeRecipient gets 6.669999995e14 WETH (2 USDC)

- With recommended mitigation:
    - Liquidator gets 0.03333e18 WETH (100 USDC) + 1.6665e15 WETH (5 USDC)
    - feeRecipient gets 5.0035e14 WETH (1.5 USDC)

This example is shown by the following coded POC, paste this test in `Liquidate.t.sol`:

```javascript
    function test_POC_Liquidate_fee_calculation_bug() public {
        _setPrice(5000e18);

        _deposit(bob, usdc, 150e6); // lender
        _deposit(alice, weth, 0.04e18); // borrower
        _deposit(liquidator, usdc, 10_000e6); // liquidator

        // Bob places a limit order to buy credit
        _buyCreditLimit(bob, block.timestamp + 6 days, YieldCurveHelper.pointCurve(6 days, 0.03e18));
        
        // Alice matches Bob's order by borrowing from Bob.
        uint256 debtPositionId = _sellCreditMarket(alice, bob, RESERVED_ID, 100e6, 6 days, true);
        uint256 futureValue = size.getDebtPosition(debtPositionId).futureValue;
        assertEq(futureValue, 100e6);

        assertTrue(!size.isUserUnderwater(alice), "borrower should not be underwater");

        vm.warp(block.timestamp + 1 days);
        _setPrice(3000e18); // Price of collateral decreases

        assertTrue(size.isUserUnderwater(alice), "borrower should be underwater"); // Borrower is underwater
        assertTrue(size.isDebtPositionLiquidatable(debtPositionId), "loan should be liquidatable"); // Debt position is liquidatable

        uint256 liquidatorActualProfit = _liquidate(liquidator, debtPositionId);
        assertEq(liquidatorActualProfit, 3.3333333338333334e16); // Liquidator receives an amount collateral tokens worth to 100 USDC + dust
        // 3.3333333338333334e16 * 3000 / 1e18 = 100.000000015000002 USDC
    }
```

## Recommended Mitigation Steps

Replace the `debtPosition.futureValue` with `debtInCollateralToken`.

```diff
    // profitable liquidation
    if (assignedCollateral > debtInCollateralToken) {
        uint256 liquidatorReward = Math.min(
            assignedCollateral - debtInCollateralToken,
-           Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)
+           Math.mulDivUp(debtInCollateralToken, state.feeConfig.liquidationRewardPercent, PERCENT)
        );
        liquidatorProfitCollateralToken = debtInCollateralToken + liquidatorReward;
``` 




### Some liquidations are not properly incentivized due to gas prices exceeding liquidation rewards, especially on Ethereum mainnet

## Links to affected code

https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Liquidate.sol#L90-L125

## Bug Description

This issue assumes the bug in the calculation of `liquidatorReward` is fixed and liquidation incentives are as follows:
  - A liquidator receives the debt value in collateral tokens plus the lesser of the remaining excess collateral or 5% of the future value (in collateral tokens).
  - The protocol receives 10% of the remaining excess collateral, assuming there is any remaining 

The `liquidatorReward` of 5% of future value may not be enough to cover the gas of liquidators calling `liquidate()`, especially for small debt positions on Ethereum mainnet when borrower's have a low collateral ratio.

With `minimumCreditBorrowAToken` parameter set to 50e6, imagine a liquidatable debt position:

- Price of WETH is 3000 USDC
- futureValue: 51 USDC (0.017e18 WETH)
- assignedCollateral: 60 USD worth of WETH (0.02e18 WETH)
- A user can liquidate this postion for a reward of 51 USDC worth of WETH + 8.5e14 WETH (worth ~ 2.55 USDC). 
- The protocol will then receive 10% of the remaining assigned collateral ~ 1.915e15 WETH (worth ~ 5.7 USDC).
- With a simple liquidation requiring ~ 1.6e5 gas (shown below in coded POC), and average gas price of 8 gwei, the total USD value of gas spent on the liquidation is ~$4.4 USD. This exceeds the liquidation reward worth ~2.55 USD.
- In this example, no external user would call `liquidate()`, however a keeper which is the same entity as the `feeRecipient` may due to the additional protocol fee received.
- If we modify the above example where the assigned collateral has a value of ~53 USDC, there is no party, including keepers / `feeRecipient` that can liquidate the user without incurring a direct loss.



## Impact
- There will be some debt positions where no external user can liquidate profitibly. I estimate this is true for all debt positions on Ethereum mainnet with 50 USDC <= `futureValue` <= 90 USDC.
- For debt positions in the above range, where the borrower also has a low collateral ratio such that the `liquidatorReward` would exceed the excess collateral, no external user and no keeper or entity of the protocol will be properly incentivized to liquidate the position.
- Consequently, lenders are forced to liquidate at a loss, or wait for the price of the collateral to change with two possible outcomes:
  - Collateral price increases such that it becomes profitable for other users to liquidate the position
  - Collateral price decreases such that the lender can call `selfLiquidate()` (also at a loss).
- Note that lenders with a passive `buyCreditLimit()` are not able to ensure that the borrower's they're matched with open positions with sufficiently large `futureValue`, so they may be in this position through no fault of their own.


## Proof of Concept

Below is a coded POC to calculate gas usage for a simple liquidation, paste the following into `Liquidate.t.sol`:

```javascript
    function test_POC_Liquidate_gas_usage() public {
        _setPrice(5000e18);

        _deposit(bob, usdc, 150e6); // lender
        _deposit(alice, weth, 0.04e18); // borrower
        _deposit(liquidator, usdc, 10_000e6); // liquidator

        // Bob places a limit order to buy credit
        _buyCreditLimit(bob, block.timestamp + 6 days, YieldCurveHelper.pointCurve(6 days, 0.03e18));
        
        // Alice matches Bob's order by borrowing from Bob.
        uint256 debtPositionId = _sellCreditMarket(alice, bob, RESERVED_ID, 100e6, 6 days, true);
        uint256 futureValue = size.getDebtPosition(debtPositionId).futureValue;

        vm.warp(block.timestamp + 1 days);
        _setPrice(3000e18); // Price of collateral decreases

        uint256 gasSnapshot1 = gasleft();
        
        vm.prank(liquidator);
        size.liquidate(
            LiquidateParams({debtPositionId: debtPositionId, minimumCollateralProfit: 0})
        );

        uint256 gasSnapshot2 = gasleft();
        console2.log('gas used %e', gasSnapshot1 - gasSnapshot2); // 1.60495e5
    }
```

Output:

```
[PASS] test_POC_Liquidate_gas_usage() (gas: 1414364)
Logs:
  gas used 1.60495e5
```

## Tools Used

Manual review

## Recommended Mitigation Steps

- Consider increasing the `minimumCreditBorrowAToken` on the Ethereum mainnet deployment




### Borrowers who call `compensate()` with `RESERVED_ID` will never pay a fragmentation fee, despite being responsible for the fragmentation of credit

## Links to affected code

https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Compensate.sol#L106-L156


## Bug Description

According to the [docs](https://docs.size.credit/technical-docs/contracts/2.3-fees#id-2.3.2-fragmentation-fee), fragmentation fees are charged to cover costs associated with claiming additional credit positons created as a result of credit fragmentation:

>Eventually, the new CreditPosition will become liquidity at the due date. However, this new CreditPosition will also have to be claimed (see the claim chapter for more details), which means one additional transaction to be sent on-chain, for each credit fractionalization.
>The Size team intends to run keeper bots to streamline the claim process and aggregate liquidity. However, this operation has some fixed costs in terms of gas, which is why a fixed fee is charged to the user causing the credit split."

However, borrowers calling `compensate()` with `RESERVED_ID` are able to partially repay an existing loan by creating a new credit/debt position. As a result of this operation, the number of claimable credit positions has increased by 1 (ie. credit was fragmented), but no fragmentation fee is payable.

The reason no fragmentation fee is payable when `RESERVED_ID` is passed:
- The `amountToCompensate` variable sets the `futureValue` / `credit` values on the newly created debit and credit positions. 
- The `exiterCreditRemaining` is the difference between the credit on the newly created credit position and `amountToCompensate`. 
- These values can never be different because both components of the subtraction stem from the same value.


```javascript
    function executeCompensate(State storage state, CompensateParams calldata params) external {
        ... SKIP!...

@>      uint256 amountToCompensate = Math.min(params.amount, creditPositionWithDebtToRepay.credit);

        CreditPosition memory creditPositionToCompensate;
        if (params.creditPositionToCompensateId == RESERVED_ID) {
            creditPositionToCompensate = state.createDebtAndCreditPositions({
                lender: msg.sender,
                borrower: msg.sender,
@>              futureValue: amountToCompensate,
                dueDate: debtPositionToRepay.dueDate
            });
        } 

        ... SKIP!...

@>      uint256 exiterCreditRemaining = creditPositionToCompensate.credit - amountToCompensate;

        ... SKIP!...

@>      if (exiterCreditRemaining > 0) {
            // charge the fragmentation fee in collateral tokens, capped by the user balance
            uint256 fragmentationFeeInCollateral = Math.min(
                state.debtTokenAmountToCollateralTokenAmount(state.feeConfig.fragmentationFee),
                state.data.collateralToken.balanceOf(msg.sender)
            );
            state.data.collateralToken.transferFrom(
                msg.sender, state.feeConfig.feeRecipient, fragmentationFeeInCollateral
            );
        }
    }
```


## Impact

- The `msg.sender` responsible for fragmenting liquidity is not charged a fragmentation fee.
- Claimer bots will need make an extra claim without any extra fee to cover their gas.


## Proof of Concept

See the below coded POC where Bob partially repays a debt position by creating a new debt/credit position. Bob does not pay a fragmentation fee despite increasing the number of claimable credit positions (at their respective due dates) by 1.

Paste the below test in `Compensate.t.sol`:

```javascript
    function test_POC_NoFragmentationFeePayable() public {
        _setPrice(1e18);
        _updateConfig("swapFeeAPR", 0);

        // Fragmentation fee is 5 USDC
        assertEq(size.feeConfig().fragmentationFee, 5e6);

        _deposit(alice, usdc, 200e6); // Alice lender
        _deposit(bob, weth, 400e18); // Bob borrower

        _buyCreditLimit(alice, block.timestamp + 365 days, YieldCurveHelper.pointCurve(365 days, 0.5e18)); // Alice places limit order to lend
        uint256 debtPositionId = _sellCreditMarket(bob, alice, RESERVED_ID, 120e6, 365 days, false); // Bob matches Alice's offer
        uint256 futureValue = size.getDebtPosition(debtPositionId).futureValue;
        uint256 creditPositionId = size.getCreditPositionIdsByDebtPositionId(debtPositionId)[0];

        // Snapshot before
        Vars memory _before = _state();
        (uint256 loansBefore,) = size.getPositionsCount();

        vm.prank(bob); 
        size.compensate(
            CompensateParams({
                creditPositionWithDebtToRepayId: creditPositionId, 
                creditPositionToCompensateId: RESERVED_ID, 
                amount: futureValue / 2 // Half of the original debt is repaid by the compensate calls. This fragments the credit.
            })
        );

        // Snapshot after
        Vars memory _after = _state();
        (uint256 loansAfter,) = size.getPositionsCount();

        assertEq(loansAfter - loansBefore, 1); // The compensate() call has increased the number of loans by 1.
        assertEq(_before.bob.collateralTokenBalance, _after.bob.collateralTokenBalance); // Bob did not pay a fragmentation fee
    }
```


## Tools Used

Manual review

## Recommended Mitigation Steps

The diff below shows a fix which would charge a fragmentation fee anytime the amount being compensated is less than the credit of the original debtPosition being repaid. 


```diff
    function executeCompensate(State storage state, CompensateParams calldata params) external {
        ... SKIP!...

-       uint256 exiterCreditRemaining = creditPositionToCompensate.credit - amountToCompensate;
+       uint256 exiterCreditRemaining = creditPositionWithDebtToRepay.credit - amountToCompensate;

        ... SKIP!...

    }
```






### Low / QA issues

### Recipient of `liquidatorProfitCollateralToken` is inconsistant between natspec and code

In the `LiquidateWithReplacement` library, the [natspec](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/LiquidateWithReplacement.sol#L119) indicates that the recipient of the `liquidatorProfitBorrowToken` is the 'liquidator'.

However the value of `borrowAToken` is transfered to the `feeRecipient` [here](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/LiquidateWithReplacement.sol#L162).

`LiquidateWithReplacement()` is only callable by keepers, so the 'liquidator' and the `feeRecipient` will always be the same entitity, just different wallets.

