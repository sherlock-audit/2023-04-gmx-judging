IllIllI

high

# Stop-loss orders do not become marketable orders

## Summary

Stop-loss orders do not execute if the market has moved since the order was submitted.


## Vulnerability Detail

This is the normal behavior for a stop-loss order in financial markets: "A stop-loss order becomes a market order as soon as the stop-loss price is reached. Since the stop loss becomes a market order, execution of the order is guaranteed, but not the price." - https://www.wikihow.com/Set-Up-a-Stop%E2%80%90loss-Order . In the GMX system, stop-loss orders will be unexecutable if the price has moved since the order was submitted to the mempool, because the primary and secondary prices are required to straddle the trigger price.

Another way this could happen is if there's an oracle outage, and the oracles come back after the trigger price has been passed.

## Impact

Since the stop-loss will revert if a keeper tries to execute it, essentially the order becomes wedged and there will be no stoploss protection.

Consider a scenario where the price of token X is $100, and a user who is long is in a profit above $99, but will have a loss at $98, so they set a stoploss with a trigger price of $99 and submit the order to the mempool. By the time the block gets mined 12 seconds later, the primary and secondary prices are $97/$96, and the order becomes unexecutable.


## Code Snippet

Stop-loss orders revert if the primary price and secondary price don't straddle the trigger price
```solidity
// File: gmx-synthetics/contracts/order/BaseOrderUtils.sol : BaseOrderUtils.ok   #1

270    
271            if (orderType == Order.OrderType.StopLossDecrease) {
272                uint256 primaryPrice = oracle.getPrimaryPrice(indexToken).pickPrice(shouldUseMaxPrice);
273                uint256 secondaryPrice = oracle.getSecondaryPrice(indexToken).pickPrice(shouldUseMaxPrice);
274    
275                // for stop-loss decrease orders:
276                //     - long: validate descending price
277                //     - short: validate ascending price
278                bool shouldValidateAscendingPrice = !isLong;
279    
280 @>             bool ok = shouldValidateAscendingPrice ?
281                    (primaryPrice <= triggerPrice && triggerPrice <= secondaryPrice) :
282                    (primaryPrice >= triggerPrice && triggerPrice >= secondaryPrice);
283    
284                if (!ok) {
285                    revert Errors.InvalidStopLossOrderPrices(primaryPrice, secondaryPrice, triggerPrice, shouldValidateAscendingPrice);
286:               }
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/order/BaseOrderUtils.sol#L270-L286

The revert is allowed to cause the order to fail:
```solidity
// File: gmx-synthetics/contracts/exchange/OrderHandler.sol : OrderHandler._handleOrderError()   #2

263                // the transaction is reverted for InvalidLimitOrderPrices and
264                // InvalidStopLossOrderPrices errors since since the oracle prices
265                // do not fulfill the specified trigger price
266                errorSelector == Errors.InvalidLimitOrderPrices.selector ||
267 @>             errorSelector == Errors.InvalidStopLossOrderPrices.selector
268            ) {
269                ErrorUtils.revertWithCustomError(reasonBytes);
270:           }
```
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/exchange/OrderHandler.sol#L263-L270


## Tool used

Manual Review


## Recommendation

Don't revert if both the primary and secondary prices are worse than the trigger price.


