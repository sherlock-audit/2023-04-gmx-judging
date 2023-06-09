IllIllI

high

# Limit swap orders can be used to get a free look into the future

## Summary

Users can cancel their limit swap orders to get a free look into prices in future blocks


## Vulnerability Detail

This is a part of the same issue that was described in the [last contest](https://github.com/sherlock-audit/2023-02-gmx-judging/issues/130). The sponsor fixed the bug for `LimitDecrease` and `StopLossDecrease`, but not for `LimitSwap`.

Any swap limit order submitted in block range N can't be executed until block range N+2, because the block range is forced to be after the submitted block range, and keepers can't execute until the price has been archived, which necessarily won't be until _after_ block range N+1. Consider what happens when half of the oracle's block ranges are off from the other half, e.g.:

```text
    1 2 3 4 5 6 7 8 9 < block number
O1: A B B B B C C C D
02: A A B B B B C C C
^^ grouped oracle block ranges
```

At block 1, oracles in both groups (O1 and O2) are in the same block range A, and someone submits a large swap limit order (N). At block 6, oracles in O1 are in N+2, but oracles in O2 are still in N+1. This means that the swap limit order will execute at the median price of block 5 (since the earliest group to have archive prices at block 6 for N+1 will be O1) and market swap order submitted at block 6 in the other direction will execute at the median price of block 6 since O2 will be the first group to archive a price range that will contain block 6. By the end of block 5, the price for O1 is known, and the price that O2 will get at block 6 can be predicted with high probability (e.g. if the price has just gapped a few cents), so a trader will know whether the two orders will create a profit or not. If a profit is expected, they'll submit the market order at block 6. If a loss is expected, they'll cancel the swap limit order from block 1, and only have to cover gas fees.

Essentially the logic is that limit swap orders will use earlier prices, and market orders (with swaps) will use later prices, and since oracle block ranges aren't fixed, an attacker is able to know both prices before having their orders executed, and use large order sizes to capitalize on small price differences.


## Impact

There is a lot of work involved in calculating statistics about block ranges for oracles and their processing time/queues, and ensuring one gets the prices essentially when the keepers do, but this is likely less work than co-located high frequency traders in traditional finance have to do, and if there's a risk free profit to be made, they'll put in the work to do it every single time, at the expense of all other traders.


## Code Snippet

Market orders can use the current block, but limit orders must use the next block:
```solidity
// File: gmx-synthetics/contracts/order/SwapOrderUtils.sol : SwapOrderUtils.validateOracleBlockNumbers()   #1

57            if (orderType == Order.OrderType.MarketSwap) {
58 @>             OracleUtils.validateBlockNumberWithinRange(
59                    minOracleBlockNumbers,
60                    maxOracleBlockNumbers,
61                    orderUpdatedAtBlock
62                );
63                return;
64            }
65    
66            if (orderType == Order.OrderType.LimitSwap) {
67 @>             if (!minOracleBlockNumbers.areGreaterThan(orderUpdatedAtBlock)) {
68                    revert Errors.OracleBlockNumbersAreSmallerThanRequired(minOracleBlockNumbers, orderUpdatedAtBlock);
69                }
70                return;
71            }
72    
73            revert Errors.UnsupportedOrderType();
74:       }
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/order/SwapOrderUtils.sol#L57-L74


## Tool used

Manual Review


## Recommendation

All orders should follow the same block range rules

