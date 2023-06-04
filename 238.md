IllIllI

high

# Liquidation and ADL orders swap PnL to collateral with unlimited slippage

## Summary

Liquidation and ADL orders swap PnL to collateral with unlimited slippage


## Vulnerability Detail

The code what changed from using `DecreasePositionSwapType.NoSwap` to `DecreasePositionSwapType.SwapPnlTokenToCollateralToken` without adjusting the slippage parameters.


## Impact

Liquidations may end up losing all collateral to the pool impact amount, especially when there are cascading liquidations

There may be a large price impact associated with the ADL order, causing the user's large gain to become a loss, if the price impact factor is large enough, which may be the case if the price suddenly spikes up a lot, and there are many ADL operations after a lot of users exit their positions.


## Code Snippet

Liquidation orders allow swaps with unlimited slippage:
```solidity
// File: gmx-synthetics/contracts/liquidation/LiquidationUtils.sol : LiquidationUtils.createLiquidationOrder()   #1

44            Order.Numbers memory numbers = Order.Numbers(
45                Order.OrderType.Liquidation, // orderType
46 @>             Order.DecreasePositionSwapType.SwapPnlTokenToCollateralToken, // decreasePositionSwapType
47                position.sizeInUsd(), // sizeDeltaUsd
48                0, // initialCollateralDeltaAmount
49                0, // triggerPrice
50                position.isLong() ? 0 : type(uint256).max, // acceptablePrice
51                0, // executionFee
52                0, // callbackGasLimit
53 @>             0, // minOutputAmount
54                Chain.currentBlockNumber() // updatedAtBlock
55:           );
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/liquidation/LiquidationUtils.sol#L44-L53

Same with ADL orders:
```solidity
// File: gmx-synthetics/contracts/adl/AdlUtils.sol : AdlUtils.createAdlOrder()   #2

156            Order.Numbers memory numbers = Order.Numbers(
157                Order.OrderType.MarketDecrease, // orderType
158                Order.DecreasePositionSwapType.SwapPnlTokenToCollateralToken, // decreasePositionSwapType
159                params.sizeDeltaUsd, // sizeDeltaUsd
160                0, // initialCollateralDeltaAmount
161                0, // triggerPrice
162                position.isLong() ? 0 : type(uint256).max, // acceptablePrice
163                0, // executionFee
164                0, // callbackGasLimit
165 @>             0, // minOutputAmount
166                params.updatedAtBlock // updatedAtBlock
167:           );
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/adl/AdlUtils.sol#L156-L167


## Tool used

Manual Review


## Recommendation

Introduce `MAX_SWAP_IMPACT_FACTOR_FOR_ADL` and `MAX_SWAP_IMPACT_FACTOR_FOR_LIQUIDATIONS`
