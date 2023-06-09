IllIllI

high

# Pool amount adjustments for collateral decreases aren't undone if swaps are successful

## Summary

Pool amount adjustments, which are added to storage as a temporary accounting tracking mechanism, aren't undone if swaps are successful


## Vulnerability Detail

`swapProfitToCollateralToken()` uses `Keys.poolAmountAdjustmentKey()` to temporarily store an adjustment to the pool amount, and undoes the adjustment at the end of the function, after the try-catch block. However, the function bypasses the end of the function in the succeess case, because it returns early, and therefore doesn't undo the adjustment.


## Impact

The value is looked up and used in `getNextPoolAmountsParams()` for swap orders that occur afterwards. The adjusted amount will be included until someone does another swap of the profit to a collateral token, at which point it will have the new value (and not be reset unless there is an exception). The adjustment ends up being used in the calculation of swap impact, so all subsequent swaps will be priced incorrectly, giving some users discounts they don't deserve, and others a penalty that they don't deserve, when doing swaps.


## Code Snippet

Adjustment is added to storage, but isn't undone if `swapHandler.swap()` is successful:
```solidity
// File: gmx-synthetics/contracts/position/DecreasePositionCollateralUtils.sol : DecreasePositionCollateralUtils.swapProfitToCollateralToken()   #1

452                // adjust the pool amount by the poolAmountDelta so that the price impact of the swap will be
453                // more accurately calculated
454 @>             params.contracts.dataStore.setInt(Keys.poolAmountAdjustmentKey(params.market.marketToken, pnlToken), poolAmountDelta);
455    
456                try params.contracts.swapHandler.swap(
457                    SwapUtils.SwapParams(
...
470                    )
471                ) returns (address /* tokenOut */, uint256 swapOutputAmount) {
472 @>                 return (true, swapOutputAmount);
473                } catch Error(string memory reason) {
474                    emit SwapUtils.SwapReverted(reason, "");
475                } catch (bytes memory reasonBytes) {
476                    (string memory reason, /* bool hasRevertMessage */) = ErrorUtils.getRevertMessage(reasonBytes);
477                    emit SwapUtils.SwapReverted(reason, reasonBytes);
478                }
479    
480 @>             params.contracts.dataStore.setInt(Keys.poolAmountAdjustmentKey(params.market.marketToken, pnlToken), 0);
481            }
482    
483            return (false, 0);
484:       }
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/position/DecreasePositionCollateralUtils.sol#L452-L484


## Tool used

Manual Review


## Recommendation

Undo the adjustment before returning

