ten-on-ten

medium

# Un-intended overflow in calculating mid price

## Summary

```solidity
    function midPrice(Props memory props) internal pure returns (uint256) {
        return (props.max + props.min) / 2;
    }
```

this function computes average of two uint256, it is possible (however, rare) that both values are less than `type(uint256).max` and hence average also being less than `type(uint256).max` , this function would revert instead of returning correct value.

## Vulnerability Detail

Function will revert with `Panic(17)` integer  overflow.

## Impact

This function is used in `ExecuteDepositUtils` and `SwapUtils`, the call execution can revert at those places.

## Code Snippet

https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/price/Price.sol#L25-L27

## Tool used

Manual Review

## Recommendation

use https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/math/Math.sol#L100-L103
