IllIllI

medium

# Unnecessary loss of precision

## Summary

Unnecessary loss of precision from doing multiplication on the result of a division


## Vulnerability Detail

Multiplication is done on the result of a division, rather than doing the multiplication first, to avoid loss of precision

This is the same issue as was identified in the [previous contest](https://github.com/sherlock-audit/2023-02-gmx-judging/issues/171), but wasn't addressed (it was one of the duplicates of another precision issue).


## Impact

PnL calculations under/over estimate profit/loss


## Code Snippet

Multiplication on the result of a division:

```solidity
// File: gmx-synthetics/contracts/position/PositionUtils.sol : PositionUtils.getPositionPnlUsd()   #1

210            if (position.sizeInUsd() == sizeDeltaUsd) {
211                cache.sizeDeltaInTokens = position.sizeInTokens();
212            } else {
213                if (position.isLong()) {
214 @>                 cache.sizeDeltaInTokens = Calc.roundUpDivision(position.sizeInTokens() * sizeDeltaUsd, position.sizeInUsd());
215                } else {
216 @>                 cache.sizeDeltaInTokens = position.sizeInTokens() * sizeDeltaUsd / position.sizeInUsd();
217                }
218            }
219    
220 @>         cache.positionPnlUsd = cache.totalPositionPnl * cache.sizeDeltaInTokens.toInt256() / position.sizeInTokens().toInt256();
221    
222            return (cache.positionPnlUsd, cache.sizeDeltaInTokens);
223:       }
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/position/PositionUtils.sol#L205-L223


## Tool used

Manual Review


## Recommendation

Modify each of the `sizeDeltaInTokens` calculations to multiply by `totalPositionPnl` prior to doing their divisions, then remove the multiplication from the `positionPnlUsd` calculation.



