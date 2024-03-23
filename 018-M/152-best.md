Jumpy Violet Ram

medium

# Max allocations can be bypassed with multiple addresses because of guaranteed allocations

## Summary
[`TokenSale._processPrivate()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L226) ensures that a user cannot deposit more than their allocation amount. However, each address can deposit up to at least `maxAllocations`. This can be leveraged by a malicious user by using different addresses to claim all tokens without even staking.

## Vulnerability Detail
The idea of the protocol is to give everyone the right to have at least `maxAlocations` allocations. By completing missions, users level up and unlock new tiers. This process will be increasing their allocations. The problem is that when a user has no allocations, they have still a granted amount of `maxAllocations`.

[`TokenSale.calculateMaxAllocation`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L259C1-L267C6) returns $max(maxTierAlloc(), maxAllocation)$

For a user with no allocations, `_maxTierAlloc()` will return 0. The final result will be that this user have `maxAllocation` allocations (because maxAllocation > 0).
```solidity
        if (userTier == 0 && giftedTierAllc == 0) {
            return 0;
        }
```

Multiple Ethereum accounts can be used by the same party to take control over the IDO and all its allocations, on top of that without even staking.

*NOTE*: setting `maxAllocation = 0` is not a solution in this case because the protocol wants to still give some allocations to their users.

## Impact
Buying all allocations without staking. This also violates a key property that only ION holders can deposit.

## Code Snippet
```solidity
    function calculateMaxAllocation(address _sender) public returns (uint256) {
        uint256 userMaxAllc = _maxTierAllc(_sender);

        if (userMaxAllc > maxAllocation) {
            return userMaxAllc;
        } else {
            return maxAllocation;
        }
    }
```
## Tool used

Manual Review

## Recommendation
A possible solution may be to modify `calculateMaxAllocation` in the following way:
```diff
    function calculateMaxAllocation(address _sender) public returns (uint256) {
        uint256 userMaxAllc = _maxTierAllc(_sender);
+       if (userMaxAllc == 0) return 0;

         if (userMaxAllc > maxAllocation) {
            return userMaxAllc;
        } else {
            return maxAllocation;
        }
    }
```