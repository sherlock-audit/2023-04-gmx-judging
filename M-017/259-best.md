IllIllI

medium

# Funding fee accounting is incorrect when the number of sides of OI increases to two

## Summary

Funding fee accounting is incorrect if the number of sides of open interest increases from one to two


## Vulnerability Detail

The code special cases the scenario where there is [only one side](https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/market/MarketUtils.sol#L1019-L1023) with open interest, and doesn't require that side to pay while there is no other said to pay the funding to. It does this by not updating the `fundingFactorPerSecond`. However, once things get into that state, there is no code that delineates when non-payments stop and when payments begin.


## Impact

The code assumes that the non-paying side now has to pay funding for the whole period, even if the other side is just now joining. If the non-paying period was days long, the side that existed now has to pay that full amount, even if the other side only will get the portion for when they had the position. If the position is small enough, it's possible for it to be liquidated due to the fees.


## Code Snippet

Funding payments are [calculated](https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/market/MarketUtils.sol#L1340-L1374) based on when the position was created, and the latest value when the existing position changes:

```solidity
// File: gmx-synthetics/contracts/pricing/PositionPricingUtils.sol : PositionPricingUtils.getFundingFees()   #1

457            address longToken,
458            address shortToken,
459            int256 latestLongTokenFundingAmountPerSize,
460            int256 latestShortTokenFundingAmountPerSize
461        ) internal pure returns (PositionFundingFees memory) {
462            PositionFundingFees memory fundingFees;
463    
464            fundingFees.latestLongTokenFundingAmountPerSize = latestLongTokenFundingAmountPerSize;
465            fundingFees.latestShortTokenFundingAmountPerSize = latestShortTokenFundingAmountPerSize;
466    
467            int256 longTokenFundingFeeAmount = MarketUtils.getFundingFeeAmount(
468 @>             fundingFees.latestLongTokenFundingAmountPerSize,
469 @>             position.longTokenFundingAmountPerSize(),
470                position.sizeInUsd()
471            );
472    
473            int256 shortTokenFundingFeeAmount = MarketUtils.getFundingFeeAmount(
474 @>             fundingFees.latestShortTokenFundingAmountPerSize,
475 @>             position.shortTokenFundingAmountPerSize(),
476                position.sizeInUsd()
477            );
478    
479            // if the position has negative funding fees, distribute it to allow it to be claimable
480            if (longTokenFundingFeeAmount < 0) {
481:               fundingFees.claimableLongTokenAmount = (-longTokenFundingFeeAmount).toUint256();
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/pricing/PositionPricingUtils.sol#L457-L481


It doesn't know about any temporary stoppages of funding, because it's strictly a time-based global value, with no gaps:

```solidity
// File: gmx-synthetics/contracts/market/MarketUtils.sol : MarketUtils.   #2

1784        function getSecondsSinceFundingUpdated(DataStore dataStore, address market) internal view returns (uint256) {
1785            uint256 updatedAt = dataStore.getUint(Keys.fundingUpdatedAtKey(market));
1786            if (updatedAt == 0) { return 0; }
1787 @>         return Chain.currentTimestamp() - updatedAt;
1788:       }
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/market/MarketUtils.sol#L1784-L1788


## Tool used

Manual Review


## Recommendation

Only allow a certain static number of positions, of a minimum size, to be opened while there is no other side. Once the other side gets opened, have a function that will trigger an update of the positions' funding amount per size. Be sure to be able to support the case where the other side later disappears, and things need to be re-triggered.

