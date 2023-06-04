IllIllI

medium

# `boundedSub()` reverts rather than returning a bounded value, when `type(int256).min` is used

## Summary

`boundedSub()` reverts if `b` is `type(int256).min`


## Vulnerability Detail

`boundedSub()` is supposed to return bounded values (`type(int256).min` or `type(int256).max`) rather than overflowing. If `b` is `type(int256).min`, it reverts rather than returning a value.


## Impact

`boundedSub()` is supposed to always return a value, so that calculations with large numbers, such as those having to do with funding amounts, do not cause orders to revert. 


## Code Snippet

The test below shows that the [function](https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/utils/Calc.sol#L118-L137) will revert under two separate conditions (A and B), both of which occur when `b` is `type(int256).min`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import "forge-std/console.sol";

contract It is Test {
    function testIt(int256 a, int256 b) external view returns (int256) {
        console.logInt(type(int256).min);
        console.logInt(type(int256).max);

        //vm.assume(0 == a && type(int256).min == b); // A
        //vm.assume(!(0 == a && type(int256).min == b)); // !A

        //vm.assume(type(int256).min == b); // B
        //vm.assume(type(int256).min != b); // !B

        //vm.assume(b >= 0);
    
        boundedSub2(a,b);
        boundedSub(a,b);
        assert(boundedSub(a,b) == boundedSub2(a,b));
    }

    function boundedSub(int256 a, int256 b) internal pure returns (int256 answer) {
        // if either a or b is zero or the signs are the same there should not be any overflow
        if (a == 0 || b == 0 || (a > 0 && b > 0) || (a < 0 && b < 0)) {
            return a - b; // @audit A) vm.assume(0 == a && type(int256).min == b);
        }

        // if adding `-b` to `a` would result in a value greater than the max int256 value
        // then return the max int256 value
        if (a > 0 && -b >= type(int256).max - a) { // @audit B) vm.assume(type(int256).min == b);
            return type(int256).max;
        }

        // if subtracting `b` from `a` would result in a value less than the min int256 value
        // then return the min int256 value
        if (a < 0 && -b <= type(int256).min - a) {
            return type(int256).min;
        }

        return a - b;
    }

    // based on https://github.com/OpenZeppelin/openzeppelin-contracts/blob/b8c8308d77beaa733104d1d66ec5f2962df81711/contracts/drafts/SignedSafeMath.sol#L41-L49
    function boundedSub2(int256 a, int256 b) internal pure returns (int256) {
        unchecked {
            int256 c = a - b;
            if (b >= 0 && c > a) {
                return type(int256).min;
            }
            if (b < 0 && c <= a) {
                return type(int256).max;
            }
            return c;
        }
    }
}
```

## Tool used

Manual Review


## Recommendation

Change the code to what I have in `boundedSub2()` which uses an `unchecked` block so that it can check for overflows/underflows, rather than reverting in those cases.

