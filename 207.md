J4de

medium

# Malicious users may cancel the limit order before the keeper to consume the keeper's gas fee

## Summary

The limit order can be canceled at any time, and malicious users may cancel before the keeper is executed to consume the keeper's gas fee.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/exchange/OrderHandler.sol#L130-L136

There is no time limit for the cancellation of the current price order, and it can be canceled at any time.

## Impact

Loss of keeper funds

## Code Snippet

https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/exchange/OrderHandler.sol#L130-L136

## Tool used

Manual Review

## Recommendation

It is recommended to add some delay to the cancellation of the limit order, or it cannot be canceled immediately like the market order
