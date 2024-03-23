Warm Gingerbread Kookaburra

medium

# `updateUserDeposit` overwrites current values

## Summary
`updateUserDeposit` overrides current values 

## Vulnerability Detail
If we look at `updateUserDeposit`, we'll see that the amount to be vested to the user will get overwritten, instead of increased. 
```solidity
    function updateUserDeposit(
        address[] memory _users,
        uint256[] memory _amount
    ) public onlyRole(DEFAULT_ADMIN_ROLE) {
        require(_users.length <= 250, "array length should be less than 250");
        require(_users.length == _amount.length, "array length should match");
        uint256 amount;
        for (uint256 i = 0; i < _users.length; i++) {
            userdetails[_users[i]].userDeposit = _amount[i];
            amount += _amount[i];
        }
        token.safeTransferFrom(distributionWallet, address(this), amount);
    }
```
Despite, the idea of the function being to update the existing value, If the user has prior balance, it will be lost and the tokens will be forever stuck. 

## Impact
Loss of funds

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Vesting.sol#L61

## Tool used

Manual Review

## Recommendation
Increase the current values instead of overwriting them
