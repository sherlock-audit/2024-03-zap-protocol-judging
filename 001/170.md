Fancy Cerulean Kookaburra

high

# Users are able to claim more tokens than they are entitled to by calling Vesting.claim multiple times.

## Summary

An incorrect condition (`require(s.index != vestingPoints.length, "already claimed")`) in the `Vesting.sol` contract's `claim` function allows a user that has already fully claimed to claim multiple times and steal tokens.

## Vulnerability Detail

When a user has fully claimed, their userDetails.index is updated to be equal to `vestingPoints.length - 1`.

When they call `claim` again the `require(s.index != vestingPoints.length, "already claimed")` always passes since index cannot exceed `vestingPoints.length - 1`.

The loop will trigger for vestingPoints[1][0] only since the `i <= vestingPoints.length - 1` is always true when i=s.index=1.

The user then receives additional tokens they are not entitled to.

## Impact

Users that have fuilly claimed may call the `claim` function multiple times to claim additional tokens. Other users may be unable to claim if there are not enough tokens in the contract.


## Code Snippet

[Vesting.sol#L72](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Vesting.sol#L72)

## Tool used

Manual Review

## Recommendation

Change `require(s.index != vestingPoints.length, "already claimed")` to `require(s.index != vestingPoints.length - 1 , "already claimed")`.