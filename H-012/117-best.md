J4de

high

# `MarketUtils.sol#getFundingFeeAmount` loss of precision

## Summary

The `numerator` variable of the `getFundingFeeAmount` function is divided first and then multiplied, resulting in a loss of 15 precision

## Vulnerability Detail

```solidity
File: market/MarketUtils.sol
1345         int256 fundingDiffFactor = (latestFundingAmountPerSize - positionFundingAmountPerSize);
1346
1347         // divide the positionSizeInUsd by Precision.FLOAT_PRECISION_SQRT as the fundingAmountPerSize values
1348         // are stored based on FLOAT_PRECISION_SQRT values instead of FLOAT_PRECISION to reduce the chance
1349         // of overflows
1350         // adjustedPositionSizeInUsd will always be zero if positionSizeInUsd is less than Precision.FLOAT_PRECISION_SQRT
1351         // position sizes should be validated to be above a minimum amount
1352         // to prevent gaming by using small positions to avoid paying funding fees
1353         uint256 adjustedPositionSizeInUsd = positionSizeInUsd / Precision.FLOAT_PRECISION_SQRT;
1354
1355         int256 numerator = adjustedPositionSizeInUsd.toInt256() * fundingDiffFactor;
1356
1357         if (numerator == 0) { return 0; }
1358
1359         // if the funding fee amount is negative, it means this amount should be claimable
1360         // by the user
1361         // round the funding fee amount down for this case
1362         if (numerator < 0) {
1363             return numerator / Precision.FLOAT_PRECISION.toInt256();
1364         }
```

- `positionSizeInUsd` decimals: 30
- `Precision.FLOAT_PRECISION_SQRT` decimals: 15
- `fundingDiffFactor` decimals: 15
- `Precision.FLOAT_PRECISION` decimals: 30

So `adjustedPositionSizeInUsd` decimals is 30 - 15 = 15, and `numerator` decimals is 15 + 15 = 30. Dividing the `numerator` by `10**30` will lose all precision.

## Impact

Users may get more `FundingFee` or lose some `FundingFee`

## Code Snippet

https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/market/MarketUtils.sol#L1363

https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/market/MarketUtils.sol#L1373

## Tool used

Manual Review

## Recommendation

The numerator is already 30 decimals, no need to divide by `10**30`
