rvierdiiev

medium

# User can loose funds in case if swapping in DecreaseOrderUtils.processOrder will fail

## Summary
When user executes decrease order, then he provides `order.minOutputAmount` value, that should protect his from loses. This value is provided with hope that swapping that will take some fees will be executed. But in case if swapping will fail, then this `order.minOutputAmount` value will be smaller then user would like to receive in case when swapping didn't occur. Because of that user can receive less output amount.
## Vulnerability Detail
`DecreaseOrderUtils.processOrder` function executed decrease order and [returns order execution result](https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/order/DecreaseOrderUtils.sol#L37-L46) which contains information about output tokens and amounts that user should receive.

In case if only 1 output token is returned to user, then function will try to swap that amount according to the swap path that user has provided.
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/order/DecreaseOrderUtils.sol#L83-L116
```solidity
        try params.contracts.swapHandler.swap(
            SwapUtils.SwapParams(
                params.contracts.dataStore,
                params.contracts.eventEmitter,
                params.contracts.oracle,
                Bank(payable(order.market())),
                params.key,
                result.outputToken,
                result.outputAmount,
                params.swapPathMarkets,
                0,
                order.receiver(),
                order.uiFeeReceiver(),
                order.shouldUnwrapNativeToken()
            )
        ) returns (address tokenOut, uint256 swapOutputAmount) {
            `(
                params.contracts.oracle,
                tokenOut,
                swapOutputAmount,
                order.minOutputAmount()
            );
        } catch (bytes memory reasonBytes) {
            (string memory reason, /* bool hasRevertMessage */) = ErrorUtils.getRevertMessage(reasonBytes);

            _handleSwapError(
                params.contracts.oracle,
                order,
                result,
                reason,
                reasonBytes
            );
        }
    }
```
As you can see in case if swap succeeded, then `_validateOutputAmount` function will be called, that will check slippage. It will check that `swapOutputAmount` is received according to the slippage.
But in case if swap will not succeed, then `_handleSwapError` will be called.
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/order/DecreaseOrderUtils.sol#L208-L230
```solidity
    null(
        Oracle oracle,
        Order.Props memory order,
        DecreasePositionUtils.DecreasePositionResult memory result,
        string memory reason,
        bytes memory reasonBytes
    ) internal {
        emit SwapUtils.SwapReverted(reason, reasonBytes);

        _validateOutputAmount(
            oracle,
            result.outputToken,
            result.outputAmount,
            order.minOutputAmount()
        );

        MarketToken(payable(order.market())).transferOut(
            result.outputToken,
            order.receiver(),
            result.outputAmount,
            order.shouldUnwrapNativeToken()
        );
    }
```
As you can see in this case `_validateOutputAmount` function will be called as well, but it will be called with `result.outputAmount` this time, which is amount provided by decreasing of position.

Now i will describe the problem.
In case if user wants to swap his token, he knows that he needs to pay fees to the market pools and that this swap will eat some amount of output. So in case if `result.outputAmount` is 100$ worth of tokenA, it's fine if user will provide slippage as 3% if he has long swap path, so his slippage is 97$. 
But in case when swap will fail, then now this slippage of 97$ is incorrect as user didn't do swapping and he should receiev exactly 100$ worth of tokenA.

Also i should note here, that it's easy to make swap fail for keeper, it's enough for him to just not provide any asset price, so swap reverts. So keeper can benefit on this slippage issue.
## Impact
User can be frontruned to receive less amount in case of swapping error.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Maybe it's needed to have another slippage param, that should be used in case of no swapping.
