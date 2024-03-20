Warm Gingerbread Kookaburra

high

# TokenSale does not call `stakingContract.setPoolsEndTime` although it's supposed to.

## Summary
TokenSale does not call `stakingContract.setPoolsEndTime` although it's supposed to.

## Vulnerability Detail
If we look at the code of `_processPrivate`, we'll see that in the end it's supposed to make a call to the staking contract and forbid unstaking. However, it is left commented out, meaning that this call is actually never made, allowing for unstaking to happen. 

```solidity
        /**@notice Forbid unstaking*/
        // stakingContract.setPoolsEndTime(_sender, uint256(params.privateEnd)); // TODO: uncomment
        emit DepositPrivate(_sender, _amount, address(this));
```

Despite lacking context for the code of the staking contract, we can assume that locking the unstaking is an important part of it, hence the severity of the issue 

## Impact
Unstaking not locked, even though supposed to

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L250

## Tool used

Manual Review

## Recommendation
uncomment the line 