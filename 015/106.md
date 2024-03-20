Keen Latte Chicken

medium

# The sum of vesting points is not guaranteed to be equal to 1000 in `Vesting.initialize`.

## Summary
In `Vesting.initialize`, the vesting points are sorted, but the sum of vesting points is not guaranteed to be equal to 1000. This leads to users receive less tokens than expected.

## Vulnerability Detail
In `Vesting.initialize`, the vesting points are sorted, but the sum of vesting points is not checked to be equal to 1000.
```solidity
33:    function initialize(
34:        address operator,
35:        address _tokenSale,
36:        IERC20 _token,
37:        address _distributionWallet,
38:        uint128[2][] memory _vestingPoints
39:    ) public initializer {
40:        _setupRole(DEFAULT_ADMIN_ROLE, operator);
41:        tokensale = _tokenSale;
42:@>      (vestingPoints, ) = ascendingSort(_vestingPoints);
43:        token = _token;
44:        distributionWallet = _distributionWallet;
45:    }
```
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Vesting.sol#L33-L45

However, the sum of vesting points is assumed to be 10000 when calculating the claimed amount in `Vesting.claim`, as the denominator is hardcoded to 10000 on line 77. It means that if total vesting points is smaller than 10000 in `Vesting.initialize`, user will receive less tokens than they should.
```solidity
67:    function claim() external {
68:        address sender = msg.sender;
69:
70:        UserDetails storage s = userdetails[sender];
71:        require(s.userDeposit != 0, "No Deposit");
72:        require(s.index != vestingPoints.length, "already claimed");
73:        uint256 pctAmount;
74:        uint256 i = s.index;
75:        for (i; i <= vestingPoints.length - 1; i++) {
76:            if (block.timestamp >= vestingPoints[i][0]) {
77:@>              pctAmount += (s.userDeposit * vestingPoints[i][1]) / 10000;
78:            } else {
79:                break;
80:            }
81:        }
82:        if (pctAmount != 0) {
83:            if (address(token) == address(1)) {
84:                (bool sent, ) = payable(sender).call{value: pctAmount}("");
85:                require(sent, "Failed to send BNB to receiver");
86:            } else {
87:@>              token.safeTransfer(sender, pctAmount);
88:            }
89:            s.index = uint128(i);
90:            s.amountClaimed += pctAmount;
91:        }
92:    }
```
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Vesting.sol#L67-L92

## Impact
User receives less tokens than they should if the total vesting points is not equal to 10000.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Vesting.sol#L67-L92

## Tool used

Manual Review

## Recommendation
Add a check to ensure that the sum of vesting points is equal to 10000 after sorting the vesting points in `Vesting.initialize`.
```solidity
    function initialize(...) {
...
-       (vestingPoints, ) = ascendingSort(_vestingPoints);
+       (vestingPoints, uint256 sum) = ascendingSort(_vestingPoints);
+       require(sum == 10000, "sum not 10000");
...
    } 
```