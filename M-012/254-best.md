IllIllI

medium

# Stable prices don't have their values validated like oracle prices do

## Summary

Stable prices don't have their values validated like oracle prices do, even though Chainlink prices are more reliable than a 'stable price', when there are shocks to the system


## Vulnerability Detail

As a part of the post-contest code changes, a check against the `MAX_ORACLE_REF_PRICE_DEVIATION_FACTOR` parameter, to ensure that the oracle prices weren't wildly inaccurate. The same was not done for stable prices.


## Impact

If one of the stable coins has a de-peg event (think USDC's recent one, but more sudden - USDT?), it's unlikely for the stable price to be updated in time, and traders that are fast or have automation will be able to take advantage of this discrepancy in price, at the expense of the other side of the trade.


## Code Snippet

When there's a stable price, its value is not validated against the prevailing market price - it's blindly accepted as the bid or ask:
```solidity
// File: gmx-synthetics/contracts/oracle/Oracle.sol : Oracle._setPricesFromPriceFeeds()   #1

639                uint256 stablePrice = getStablePrice(dataStore, token);
640    
641                Price.Props memory priceProps;
642    
643                if (stablePrice > 0) {
644                    priceProps = Price.Props(
645 @>                     price < stablePrice ? price : stablePrice,
646 @>                     price < stablePrice ? stablePrice : price
647                    );
648                } else {
649                    priceProps = Price.Props(
650                        price,
651                        price
652                    );
653                }
654    
655:               primaryPrices[token] = priceProps;
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/oracle/Oracle.sol#L635-L655


## Tool used

Manual Review


## Recommendation

Call `validateRefPrice()` as is done for Chainlink prices

