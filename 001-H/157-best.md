Narrow Burgundy Barracuda

high

# Reentrancy in Vesting.sol:claim() will allow users to drain the contract due to executing .call() on user's address before setting s.index = uint128(i)

## Summary
Reentrancy in Vesting.sol:claim() will allow users to drain the contract due to executing .call() on user's address before setting s.index = uint128(I)

## Vulnerability Detail
Here is the Vesting.sol:claim() function:
```solidity
function claim() external {
        address sender = msg.sender;

        UserDetails storage s = userdetails[sender];
        require(s.userDeposit != 0, "No Deposit");
@>      require(s.index != vestingPoints.length, "already claimed");
        uint256 pctAmount;
        uint256 i = s.index;
        for (i; i <= vestingPoints.length - 1; i++) {
            if (block.timestamp >= vestingPoints[i][0]) {
                pctAmount += (s.userDeposit * vestingPoints[i][1]) / 10000;
            } else {
                break;
            }
        }
        if (pctAmount != 0) {
            if (address(token) == address(1)) {
@>              (bool sent, ) = payable(sender).call{value: pctAmount}("");
                require(sent, "Failed to send BNB to receiver");
            } else {
                token.safeTransfer(sender, pctAmount);
            }
@>          s.index = uint128(i);
            s.amountClaimed += pctAmount;
        }
    }
```
From the above, You'll notice the claim() function checks if the caller already claimed by checking if the s.index has already been set to vestingPoints.length. You'll also notice the claim() function executes .call() and transfer the amount to the caller before setting the s.index = uint128(i), thereby allowing reentrancy.

Let's consider this sample scenario:
- An attacker contract(alice) has some native pctAmount to claim and calls `claim()`.
- "already claimed" check will pass since it's the first time she's calling `claim()` so her s.index hasn't been set
- However before updating Alice s.index, the Vesting contract performs external .call() to Alice with the amount sent as well
- Alice reenters `claim()` again on receive of the amount
- bypass index "already claimed" check since this hasn't been updated yet
- contract performs external .call() to Alice with the amount sent as well again,
- Same thing happens again
- Alice ends up draining the Vesting contract

## Impact
Reentrancy in Vesting.sol:claim() will allow users to drain the contract

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Vesting.sol#L84
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Vesting.sol#L89

## Tool used

Manual Review

## Recommendation
Here is the recommended fix:
```diff
if (pctAmount != 0) {
+           s.index = uint128(i);
            if (address(token) == address(1)) {
                (bool sent, ) = payable(sender).call{value: pctAmount}("");
                require(sent, "Failed to send BNB to receiver");
            } else {
                token.safeTransfer(sender, pctAmount);
            }
-           s.index = uint128(i);
            s.amountClaimed += pctAmount;
        }
```
I'll also recommend using reentrancyGuard.