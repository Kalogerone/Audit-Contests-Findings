# [H-1] - Liquidator's reward is significantly lower than it should be

## Impact

According to the size [docs](https://docs.size.credit/technical-docs/contracts/3.5-liquidations) "The liquidator gets up to a fixed 5% reward on the loan's face value". This can also be seen through code in the `Deploy` config:

```javascript
function setupProduction(
        address _owner,
        .
        .
        f = InitializeFeeConfigParams({
            swapFeeAPR: 0.005e18,
            fragmentationFee: 5e6,
@>          liquidationRewardPercent: 0.05e18,
            overdueCollateralProtocolPercent: 0.01e18,
            collateralProtocolPercent: 0.1e18,
            feeRecipient: _feeRecipient
        });
        .
        .
```

In reality this is not the case. This is the line that calculates the liquidator's reward:

```javascript
uint256 liquidatorReward = Math.min(assignedCollateral - debtInCollateralToken,
                Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT));
```

We see that `liquidatorReward` is the minimum value between:

1. `assignedCollateral - debtInCollateralToken`: In case the borrower's `collateral ratio >= 1` but very close to 1, the liquidation will still be profitable but it's possible that the borrower's collateral won't be enough to cover both the liquidation + the full 5% reward. So the liquidator is given all of the borrower's collateral in this case.

2. `Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)`: Here lies our issue. The liquidator is supposed to get 5% of the loans value and get paid in `collateralToken` which is essentially `WETH`. But the code is taking 5% of `futureValue` which has 6 decimals and is a USDC amount and then adding that to a `collateralToken` amount which has 18 decimals and is essentially a WETH amount, which is not nearly close to the correct liquidation reward.

```javascript
liquidatorProfitCollateralToken = debtInCollateralToken + liquidatorReward;
```

## Proof of Concept

Providing an example scenario with code walkthrough and a POC test with the same scenario:

1. At ETH = $1000, Alice deposits 1 ETH into size and borrows 500 USDC at 10% apr.
2. ETH price falls to $700, Alice's collateral ratio is now 1.27 and can be liquidated.
3. James has deposited 0 ETH and 1000 USDC, he decides to liquidate Alice paying 550 USDC.
4. `Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)` = 550e6 * 0.05e18 / 1e18 = 27.5e6
5. `liquidatorReward` is 27.5e6 getting paid in `collateralToken` which has 18 decimals.
6. James gets transferred `collateralTokens` equivalent to 550 USDC + dust amount.

```javascript
    function test_liquidation_rewards() public {
        _setPrice(1000e18);
        _deposit(bob, usdc, 500e6);
        _deposit(alice, weth, 1e18);
        _deposit(james, usdc, 1000e6);
        uint256[] memory tenors = new uint256[](1);
        tenors[0] = 365 days;

        int256[] memory aprs = new int256[](1);
        aprs[0] = 0.1e18;
        uint256[] memory marketRateMultipliers = new uint256[](1);

        _sellCreditLimit(alice, YieldCurve({tenors: tenors, aprs: aprs, marketRateMultipliers: marketRateMultipliers}));
        uint256 debtPositionId = _buyCreditMarket(bob, alice, RESERVED_ID, 500e6, 365 days, true);

        // 500e6 + 10% apr = 550e6
        assertEq(size.getDebtPosition(debtPositionId).futureValue, 550e6);

        uint256 jamesCollateralBefore = size.data().collateralToken.balanceOf(james);
        uint256 jamesBorrowATokensBefore = size.data().borrowAToken.balanceOf(james);

        // ETH price = 700 USD
        _setPrice(700e18);
        console2.log(size.collateralRatio(alice));

        // Make sure Alice's CR is above 1.2 and below 1.3 so the liquidation should be undoubtedly profitable
        assertGt(size.collateralRatio(alice), 1.2e18);
        assertLt(size.collateralRatio(alice), 1.3e18);

        _liquidate(james, debtPositionId);

        uint256 jamesCollateralAfter = size.data().collateralToken.balanceOf(james);
        uint256 jamesBorrowATokensAfter = size.data().borrowAToken.balanceOf(james);
        uint256 borrowATokensPaid = jamesBorrowATokensBefore - jamesBorrowATokensAfter;
        uint256 collateralGained = jamesCollateralAfter - jamesCollateralBefore;
        uint256 collateralToUsd = collateralGained * 700;

        // James has paid 550 borrowATokens and received collateral equivalent to less than 551 USD
        assertEq(borrowATokensPaid, 550e6);
        assertLt(collateralToUsd, 551e18);
    }
```

If we `console.log` the `collateralGained` value we see that it's `785714285741785715` which at ETH price = 700 as the scenario above is equivalent to 550.000000019250000500 USDC.


## Tools Used

Manual review, foundry

## Recommended Mitigation Steps

Convert the `futureValue` of the loan from USD amount to ETH price. There is a function used just above that does just that.

```diff
    function executeLiquidate(State storage state, LiquidateParams calldata params)
        external
        returns (uint256 liquidatorProfitCollateralToken)
    {
        .
        .
        uint256 assignedCollateral = state.getDebtPositionAssignedCollateral(debtPosition); //18 decimals
        uint256 debtInCollateralToken = state.debtTokenAmountToCollateralTokenAmount(debtPosition.futureValue);
        uint256 protocolProfitCollateralToken = 0;

        // profitable liquidation
        if (assignedCollateral > debtInCollateralToken) {
            uint256 liquidatorReward = Math.min(
                assignedCollateral - debtInCollateralToken,
-               Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)
+               Math.mulDivUp(debtInCollateralToken, state.feeConfig.liquidationRewardPercent, PERCENT)
            );

            liquidatorProfitCollateralToken = debtInCollateralToken + liquidatorReward;
            .
            .
    }
```

# [H-2] - Swap fees calculation is inconsistent, producing different fees for essentiallythe same loans

## Impact

The calculation of the swap fees is inconsistent. According to the Size [docs](https://docs.size.credit/technical-docs/contracts/2.3-fees) "The protocol fees are defined as 0.5% per year on the exchange of credit for cash operations, namely, BuyCreditMarket and SellCreditMarket". 

But this is not the case. For the same amount of `credit/futureValue` we will see that the fees are different, depending if using the `buyCreditMarket` or `sellCreditMarket` function for essentially the same loan.


## Proof of Concept

```javascript
    function test_fee_inconsistent() public {
        _deposit(bob, usdc, 10000e6);
        _deposit(bob, weth, 100e18);
        _deposit(alice, usdc, 10000e6);
        _deposit(alice, weth, 100e18);

        uint256[] memory tenors = new uint256[](1);
        tenors[0] = 365 days;

        int256[] memory aprs = new int256[](1);
        aprs[0] = 0.1e18;

        uint256[] memory marketRateMultipliers = new uint256[](1);

        // Track fee recipient balance
        uint256 feeRecipientBalance = size.data().borrowAToken.balanceOf(size.feeConfig().feeRecipient);

        // Create a loan through sellCreditMarket and track the increase in fee recipient balance
        _buyCreditLimit(
            alice,
            block.timestamp + 365 days,
            YieldCurve({tenors: tenors, aprs: aprs, marketRateMultipliers: marketRateMultipliers})
        );
        uint256 debtPositionId1 = _sellCreditMarket(bob, alice, RESERVED_ID, 100e6, 365 days, false);
        uint256 creditPositionId1 = size.getCreditPositionIdsByDebtPositionId(debtPositionId1)[0];
        uint256 credit1 = size.getCreditPosition(creditPositionId1).credit;
        uint256 feeRecipientDiff1 =
            size.data().borrowAToken.balanceOf(size.feeConfig().feeRecipient) - feeRecipientBalance;

        // Create the same loan off the same credit amount and track the increase in fee recipient balance
        _sellCreditLimit(bob, YieldCurve({tenors: tenors, aprs: aprs, marketRateMultipliers: marketRateMultipliers}));
        uint256 debtPositionId2 = _buyCreditMarket(alice, bob, RESERVED_ID, credit1, 365 days, false);
        uint256 creditPositionId2 = size.getCreditPositionIdsByDebtPositionId(debtPositionId2)[0];
        uint256 credit2 = size.getCreditPosition(creditPositionId2).credit;
        uint256 feeRecipientDiff2 =
            size.data().borrowAToken.balanceOf(size.feeConfig().feeRecipient) - feeRecipientDiff1;

        // Confirm that the credit amounts of the 2 loans are the same but the fee increases are different
        assertEq(credit1, credit2);
        assert(feeRecipientDiff1 != feeRecipientDiff2);
    }
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Review the fee calculation functions `getCashAmountIn`, `getCashAmountOut`, `getCreditAmountIn` and `getCreditAmountOut` as they seems to produce different fee results for the same loans.