Elegant Aquamarine Goblin

medium

# TokenSale.sol#calculateMaxAllocation() Incorrect return values

## Summary
A user can have 0 allocations and get the`maxAllocation` or a user can exceed the maximum allocations.

## Vulnerability Detail
`maxAllocation` is the max baseline a user can invest in a pool. However, swapped return values allow a user to have 0 allocations and get the`maxAllocation` or to exceed the maximum allocations.
 
```js
    function calculateMaxAllocation(address _sender) public returns (uint256) {
        uint256 userMaxAllc = _maxTierAllc(_sender);

        if (userMaxAllc > maxAllocation) {
            return userMaxAllc;
        } else {
            return maxAllocation;
        }
    }
```
## Impact
A user can have 0 allocations and get the`maxAllocation` or a user can exceed the maximum allocations 

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L259-L267

## Tool used

Manual Review

## Recommendation
```diff
    function calculateMaxAllocation(address _sender) public returns (uint256) {
        uint256 userMaxAllc = _maxTierAllc(_sender);

        if (userMaxAllc > maxAllocation) {
-            return userMaxAllc;
+            return maxAllocation;
        } else {
-            return maxAllocation;
+            return userMaxAllc;

        }
    }
```
