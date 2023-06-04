IllIllI

high

# Traders can get prices prior to their orders using trigger prices

## Summary

Traders can get prices prior to their orders using trigger prices


## Vulnerability Detail

A change since the prior contest was to allow limit orders to take place using prices in the same block in which the order was submitted, and to use the trigger price as one of the prices available to orders. If a trader sees that a single block contains a range of prices, the trader can submit an order with a trigger price at a better price than the current market price, and the trigger price will be used to fill the order.


## Impact

Traders will get a free look into the future at the expense of the other side of the trade. Price gaps in crypto are very common, so this will be a frequent occurance.


## Code Snippet

Increase:
```solidity
// File: gmx-synthetics/contracts/order/IncreaseOrderUtils.sol : IncreaseOrderUtils.validateOracleBlockNumbers()   #1

96             if (orderType == Order.OrderType.LimitIncrease) {
97                 // since the oracle blocks are only validated against the orderUpdatedAtBlock
98                 // it is possible to cause a limit increase order to become executable by
99                 // having the order have an initial collateral amount of zero then opening
100                // a position and depositing collateral if the limit order is desired to be executed
101                // for this case, when the limit order price is reached, the order should be frozen
102                // the frozen order keepers should only execute frozen orders if the latest prices
103                // fulfill the limit price
104 @>             if (!minOracleBlockNumbers.areGreaterThanOrEqualTo(orderUpdatedAtBlock)) {
105                    revert Errors.OracleBlockNumbersAreSmallerThanRequired(minOracleBlockNumbers, orderUpdatedAtBlock);
106                }
107:               return;
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/order/IncreaseOrderUtils.sol#L96-L107

and decrease orders can now be executed at the order's submission block:
```solidity
// File: gmx-synthetics/contracts/order/DecreaseOrderUtils.sol : DecreaseOrderUtils.validateOracleBlockNumbers()   #2

141            if (
142                orderType == Order.OrderType.LimitDecrease ||
143                orderType == Order.OrderType.StopLossDecrease
144            ) {
145                uint256 latestUpdatedAtBlock = orderUpdatedAtBlock > positionIncreasedAtBlock ? orderUpdatedAtBlock : positionIncreasedAtBlock;
146 @>             if (!minOracleBlockNumbers.areGreaterThanOrEqualTo(latestUpdatedAtBlock)) {
147                    revert Errors.OracleBlockNumbersAreSmallerThanRequired(minOracleBlockNumbers, latestUpdatedAtBlock);
148                }
149:               return;
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/order/DecreaseOrderUtils.sol#L136-L156

and the price used is the trigger price:
```solidity
// File: gmx-synthetics/contracts/order/BaseOrderUtils.sol : BaseOrderUtils.getExecutionPrice()   #3

351            // for limit orders, customIndexTokenPrice contains the triggerPrice and the best oracle
352            // price, we first attempt to fulfill the order using the triggerPrice
353:           uint256 price = customIndexTokenPrice.pickPrice(shouldUseMaxPrice);
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/order/BaseOrderUtils.sol#L351-L353


## Tool used

Manual Review


## Recommendation

Undo the change to allow the current block, and instead require a delay between when the order was last increased/submitted, and when an update is allowed, similar to REQUEST_EXPIRATION_BLOCK_AGE for the cancellation of market orders, and track a third price from prior to the order block for ascend/descend purposes.

