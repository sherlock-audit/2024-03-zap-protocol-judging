Warm Gingerbread Kookaburra

medium

# Loss of Blast USDB yield as it cannot be turned on.

## Summary
Loss of Blast USDB yield as it cannot be turned on. 

## Vulnerability Detail
The TokenSale contract is supposed to work with USDB and be deployed on Blast. On Blast, USDB is a rebasing token, such that it can be turned on and accrue yield over time. In order for a user/contract to start accruing yield on their USDB, they must call [USDB#configure](https://blastscan.io/address/0x4ef0d788470e2feb6559b93075ec5be51dba737d#code#F4#L221) 

```solidity
   function configure(YieldMode yieldMode) external returns (uint256) {
        _configure(msg.sender, yieldMode);

        emit Configure(msg.sender, yieldMode);

        return balanceOf(msg.sender);
    }
```

The current TokenSale contract has no way of making this call, thus making it impossible to accrue the extra yield from holding USDB, leading to a loss of funds.

## Impact
loss of yield 

## Code Snippet
 https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L30

## Tool used

Manual Review

## Recommendation
add a way to call USDB.configure