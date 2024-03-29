KingNFT

medium

# Keepers can steal additional execution fee from users

## Summary
The implementation of ````payExecutionFee()```` didn't take EIP-150 into consideration, a malicious keeper can exploit it to  drain out all execution fee users have paid, regardless of the actual execution cost.

## Vulnerability Detail
The issue arises on ````L55```` of ````payExecutionFee()````, as it's an ````external```` function, calling````payExecutionFee()```` is subject to EIP-150.
Only ````63/64```` gas is passed to the ````GasUtils```` sub-contract(````external library````), and the remaing ````1/64```` gas is reserved in the caller contract which will be refunded to keeper(````msg.sender````) after the execution of the whole transaction. But calculation of ````gasUsed ```` includes this portion of the cost as well.
```diff
File: contracts\gas\GasUtils.sol
46:     function payExecutionFee(
47:         DataStore dataStore,
48:         EventEmitter eventEmitter,
49:         StrictBank bank,
50:         uint256 executionFee,
51:         uint256 startingGas,
52:         address keeper,
53:         address user
54:     ) external { // @audit external call is subject to EIP-150
-55:         uint256 gasUsed = startingGas - gasleft();
+            uint256 gasUsed = startingGas - gasleft() * 64 / 63; // @audit the correct formula
56:         uint256 executionFeeForKeeper = adjustGasUsage(dataStore, gasUsed) * tx.gasprice;
57: 
58:         if (executionFeeForKeeper > executionFee) {
59:             executionFeeForKeeper = executionFee;
60:         }
61: 
62:         bank.transferOutNativeToken(
63:             keeper,
64:             executionFeeForKeeper
65:         );
66: 
67:         emitKeeperExecutionFee(eventEmitter, keeper, executionFeeForKeeper);
68: 
69:         uint256 refundFeeAmount = executionFee - executionFeeForKeeper;
70:         if (refundFeeAmount == 0) {
71:             return;
72:         }
73: 
74:         bank.transferOutNativeToken(
75:             user,
76:             refundFeeAmount
77:         );
78: 
79:         emitExecutionFeeRefund(eventEmitter, user, refundFeeAmount);
80:     }
```
A malicious keeper can exploit this issue to drain out all execution fee, regardless of the actual execution cost.
Let's take ````executeDeposit()```` operation as an example to show how it works:
```solidity
File: contracts\exchange\DepositHandler.sol
092:     function executeDeposit(
093:         bytes32 key,
094:         OracleUtils.SetPricesParams calldata oracleParams
095:     ) external
096:         globalNonReentrant
097:         onlyOrderKeeper
098:         withOraclePrices(oracle, dataStore, eventEmitter, oracleParams)
099:     {
100:         uint256 startingGas = gasleft();
101: 
102:         try this._executeDeposit(
103:             key,
104:             oracleParams,
105:             msg.sender
106:         ) {
107:         } catch (bytes memory reasonBytes) {
...
113:         }
114:     }

File: contracts\exchange\DepositHandler.sol
141:     function _executeDeposit(
142:         bytes32 key,
143:         OracleUtils.SetPricesParams memory oracleParams,
144:         address keeper
145:     ) external onlySelf {
146:         uint256 startingGas = gasleft();
...
171: 
172:         ExecuteDepositUtils.executeDeposit(params);
173:     }


File: contracts\deposit\ExecuteDepositUtils.sol
096:     function executeDeposit(ExecuteDepositParams memory params) external {
...
221: 
222:         GasUtils.payExecutionFee(
223:             params.dataStore,
224:             params.eventEmitter,
225:             params.depositVault,
226:             deposit.executionFee(),
227:             params.startingGas,
228:             params.keeper,
229:             deposit.account()
230:         );
231:     }

File: contracts\gas\GasUtils.sol
46:     function payExecutionFee(
47:         DataStore dataStore,
48:         EventEmitter eventEmitter,
49:         StrictBank bank,
50:         uint256 executionFee,
51:         uint256 startingGas,
52:         address keeper,
53:         address user
54:     ) external {
55:         uint256 gasUsed = startingGas - gasleft();
56:         uint256 executionFeeForKeeper = adjustGasUsage(dataStore, gasUsed) * tx.gasprice;
57: 
58:         if (executionFeeForKeeper > executionFee) {
59:             executionFeeForKeeper = executionFee;
60:         }
61: 
62:         bank.transferOutNativeToken(
63:             keeper,
64:             executionFeeForKeeper
65:         );
66: 
67:         emitKeeperExecutionFee(eventEmitter, keeper, executionFeeForKeeper);
68: 
69:         uint256 refundFeeAmount = executionFee - executionFeeForKeeper;
70:         if (refundFeeAmount == 0) {
71:             return;
72:         }
73: 
74:         bank.transferOutNativeToken(
75:             user,
76:             refundFeeAmount
77:         );
78: 
79:         emitExecutionFeeRefund(eventEmitter, user, refundFeeAmount);
80:     }

File: contracts\gas\GasUtils.sol
097:     function adjustGasUsage(DataStore dataStore, uint256 gasUsed) internal view returns (uint256) {
...
105:         uint256 baseGasLimit = dataStore.getUint(Keys.EXECUTION_GAS_FEE_BASE_AMOUNT);
...
109:         uint256 multiplierFactor = dataStore.getUint(Keys.EXECUTION_GAS_FEE_MULTIPLIER_FACTOR);
110:         uint256 gasLimit = baseGasLimit + Precision.applyFactor(gasUsed, multiplierFactor);
111:         return gasLimit;
112:     }
```
To simplify the problem, given
```solidity
EXECUTION_GAS_FEE_BASE_AMOUNT = 0
EXECUTION_GAS_FEE_MULTIPLIER_FACTOR = 1
executionFeeUserHasPaid = 200K Gwei
tx.gasprice = 1 Gwei
actualUsedGas = 100K
```
````actualUsedGas```` is the gas cost since ````startingGas````(L146 of ````DepositHandler.sol````) but before calling ````payExecutionFee()````(L221 of ````ExecuteDepositUtils.sol````)

Let's say, the keeper sets ````tx.gaslimit```` to make
```solidity
startingGas = 164K
```
Then the calculation of ````gasUsed````,  L55 of ````GasUtils.sol````, would be
```solidity
uint256 gasUsed = startingGas - gasleft() = 164K - (164K - 100K) * 63 / 64 = 101K
```
and
```solidity
executionFeeForKeeper = 101K * tx.gasprice = 101K * 1 Gwei = 101K Gwei
refundFeeForUser = 200K - 101K = 99K Gwei
```

As setting of ````tx.gaslimit```` doesn't affect the actual gas cost of the whole transaction, the excess gas will be refunded to ````msg.sender````. Now, the keeper increases ````tx.gaslimit```` to make ````startingGas = 6500K````, the calculation of ````gasUsed```` would be
```solidity
uint256 gasUsed = startingGas - gasleft() = 6500K - (6500K - 100K) * 63 / 64 = 200K
```
and
```solidity
executionFeeForKeeper = 200K * tx.gasprice = 200K * 1 Gwei = 200K Gwei
refundFeeForUser = 200K - 200K = 0 Gwei
```
We can see the keeper successfully drain out all execution fee, the user gets nothing refunded.

## Impact
Keepers can steal additional execution fee from users.

## Code Snippet
https://github.com/sherlock-audit/2023-04-gmx/blob/main/gmx-synthetics/contracts/gas/GasUtils.sol#L55

## Tool used

Manual Review

## Recommendation
The description in ````Vulnerability Detail```` section has been simplified. In fact, ````gasleft```` value should be adjusted after each external call during the whole call stack, not just in ````payExecutionFee()````.
