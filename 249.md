IllIllI

medium

# `initialCollateralDeltaAmount` is incorrectly interpreted as a USD value when calculating estimated remaining collateral

## Summary

Decrease orders have checks to ensure that if collateral is withdrawn, that there is enough left that the liquidation checks will still pass. The code that calculates the remaining collateral incorrectly adds a token amount to a USD value.


## Vulnerability Detail

`initialCollateralDeltaAmount` is incorrectly interpreted as a USD value when calculating estimated remaining collateral which means, depending on the token's decimals, the collateral will either be accepted or not accepted, when it shouldn't be.


## Impact

If the remaining collateral is over-estimated, the `MIN_COLLATERAL_USD` checks later in the function will pass, and the user will be able decrease their collateral, but then will immediately be liquidatable by liquidation keepers, since liquidation orders don't attempt to change the collateral amount. 

If the remaining collateral is under-estimated, the user will be incorrectly locked into their position.


## Code Snippet

`initialCollateralDeltaAmount()` isn't converted to a USD amount before being added to `estimatedRemainingCollateralUsd`, which is a USD amount:
```solidity
// File: gmx-synthetics/contracts/position/DecreasePositionUtils.sol : DecreasePositionUtils.decreasePosition()   #1

139                    // the estimatedRemainingCollateralUsd subtracts the initialCollateralDeltaAmount
140                    // since the initialCollateralDeltaAmount will be set to zero, the initialCollateralDeltaAmount
141                    // should be added back to the estimatedRemainingCollateralUsd
142 @>                 estimatedRemainingCollateralUsd += params.order.initialCollateralDeltaAmount().toInt256();
143                    params.order.setInitialCollateralDeltaAmount(0);
144                }
145    
146                // if the remaining collateral including position pnl will be below
147                // the min collateral usd value, then close the position
148                //
149                // if the position has sufficient remaining collateral including pnl
150                // then allow the position to be partially closed and the updated
151                // position to remain open
152:@>             if ((estimatedRemainingCollateralUsd + cache.estimatedRemainingPnlUsd) < params.contracts.dataStore.getUint(Keys.MIN_COLLATERAL_USD).toInt256()) {
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/position/DecreasePositionUtils.sol#L132-L152

## Tool used

Manual Review


## Recommendation

Convert to a USD amount before doing the addition

