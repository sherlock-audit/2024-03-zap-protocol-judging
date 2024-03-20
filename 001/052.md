Warm Gingerbread Kookaburra

high

# Changing vesting points after vesting has started will lead to unexpected results

## Summary
Changing vesting points after vesting has started will lead to unexpected results 

## Vulnerability Detail
Within the contract, there's a `updateVestingPoints`  function which allows change of the Vesting Points. 
```solidity
    function updateVestingPoints(
        uint128[2][] memory _vestingPoints
    ) external onlyRole(DEFAULT_ADMIN_ROLE) {
        uint256 sum;
        (vestingPoints, sum) = ascendingSort(_vestingPoints);
        require(sum == 10000, "sum not 10000");
        require(block.timestamp <= _vestingPoints[0][0], "Time lapsed");
    }
```
There's a require check at the end which makes sure that the the new vesting points will not have started. However, this isn't sufficient, as there would be problems if the vesting has already started (with the old vesting points).

Imagine the following scenario: 
1. There's 10 total vesting points, each of them giving 10% of the total vesting amount
2. Half of the vesting has gone by. A user has claimed for the first 5 points, claiming 50% of their vesting. Their index is 5. 
3. Vesting points are changed to new values. Now there are now only 2 vesting points.
4.  Since the user's index is higher than the total  number of vesting points, user cannot claim any more tokens.

In the end, the user has lost 50% of their vesting amount. 

Note: the change of vesting points can lead to both overdistributing funds and loss of funds.

## Impact
Loss of funds 

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Vesting.sol#L94C1-L101C6

## Tool used

Manual Review

## Recommendation
Add the following check in the beginning of the function 
```solidity
        require(block.timestamp <= vestingPoints[0][0], "Time lapsed");
```
