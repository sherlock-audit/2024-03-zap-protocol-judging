Salty Cedar Lizard

high

# updateVestingPoints can disturb token distribution

## Summary
Calling updateVestingPoints after vestingPoints[0][0] had passed will disturb reward distribution
## Vulnerability Detail

Calling `updateVestingPoints` after vestingPoints[0][0] block.timestamp has passed could lead to unexpected reward distribution resulting in loss for either of the parties. After calling `claim`, UserDetails.index is updated and stored according to vestingPoints.length - 1 [here](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/Vesting.sol#L75-L77) . Consider this scenario:

1st index indicating block.timestamp; 2nd index indicating % of deposit, sum should always be = 100
vestingPointsV1 = [0, 20], [1, 25], [2, 55] 
vestingPointsV2 = [3, 10], [4, 15], [5, 25], [6, 50]
in block.timestamp = 7 users who have never called claim would not be affected at all since their total claim is 100% (10+15+25+50). However, Alice, who had called `claim` at block.timestamp = 1, would have claimed 45% and in block.timestamp = 7 would be eligible to 75% more (25+50) resulting in payout != 100% and loss of funds for the protocol. 

Reverse scenario would be loss for Alice if vestingPointsV2 = [3, 30], [4, 31], [5, 39] as she would be eligible for only 39% claim on top of her 45%, total claim = 84% != 100% 

## Impact
Loss of funds / Unexpected behaviour
## Code Snippet
```solidity
        uint256 i = s.index; // catching stored index
        for (i; i <= vestingPoints.length - 1; i++) { // incrementing user index
            if (block.timestamp >= vestingPoints[i][0]) {
                pctAmount += (s.userDeposit * vestingPoints[i][1]) / 10000; 
                ...
            }
        }
        ...
    token.safeTransfer(sender, pctAmount);
            }
            s.index = uint128(i); // storing previously incremented index
```
        
## Tool used

Manual Review

## Recommendation
Forbid calling `updateVestingPoints` after vestingPoints[0][0] has passed. 