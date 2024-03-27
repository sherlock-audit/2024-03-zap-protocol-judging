# Issue H-1: Changing vesting points after vesting has started will lead to unexpected results 

Source: https://github.com/sherlock-audit/2024-03-zap-protocol-judging/issues/52 

## Found by 
bughuntoor, cawfree
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

# Issue H-2: Tax refund is calculated based on the wrong amount 

Source: https://github.com/sherlock-audit/2024-03-zap-protocol-judging/issues/57 

## Found by 
Krace, ZdravkoHr., bughuntoor, merlin, ydlee
## Summary
Tax refund is calculated based on the wrong amount

## Vulnerability Detail
After the private period has finished, users can claim a tax refund, based on their max tax free allocation.
```solidity
        (s.share, left) = _claim(s);
        require(left > 0, "TokenSale: Nothing to claim");
        uint256 refundTaxAmount;
        if (s.taxAmount > 0) {
            uint256 tax = userTaxRate(s.amount, msg.sender);
            uint256 taxFreeAllc = _maxTaxfreeAllocation(msg.sender) * PCT_BASE;
            if (taxFreeAllc >= s.share) {
                refundTaxAmount = s.taxAmount;
            } else {
                refundTaxAmount = (left * tax) / POINT_BASE;
            }
            usdc.safeTransferFrom(marketingWallet, msg.sender, refundTaxAmount);
        }
```
The problem is that in case `s.share > taxFreeAllc`, the tax refund is calculated wrongfully. Not only it should refund the tax on the unused USDC amount, but it should also refund the tax for the tax-free allocation the user has. 

Imagine the following. 
1. User deposits 1000 USDC.
2. Private period finishes, token oversells. Only half of the user's money actually go towards the sell (s.share = 500 USDC, s.left = 500 USDC)
3. The user has 400 USDC tax-free allocation
4. The user must be refunded the tax for the 500 unused USDC, as well as their 400 USDC tax-free allocation. In stead, they're only refunded for the 500 unused USDC. (note, if the user had 500 tax-free allocation, they would've been refunded all tax)


## Impact
Users are not refunded enough tax 

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L385

## Tool used

Manual Review

## Recommendation
change the code to the following: 
```solidity
                refundTaxAmount = ((left + taxFreeAllc) * tax) / POINT_BASE;
```

# Issue H-3: If token does not oversell, users cannot claim tax refund on their tax free allocation. 

Source: https://github.com/sherlock-audit/2024-03-zap-protocol-judging/issues/58 

## Found by 
bughuntoor, s1ce
## Summary
Users may not be able to claim tax refund

## Vulnerability Detail
Within TokenSale, upon depositing users, users have to pay tax. Then, users can receive a tax-free allocation - meaning they'll be refunded the tax they've paid on part of their deposit.

The problem is that due to a unnecessary require check, users cannot claim their tax refund, unless the token has oversold. 
```solidity
 function claim() external {
        checkingEpoch();
        require(
            uint8(epoch) > 1 && !admin.blockClaim(address(this)),
            "TokenSale: Not time or not allowed"
        );

        Staked storage s = stakes[msg.sender];
        require(s.amount != 0, "TokenSale: No Deposit"); 
        require(!s.claimed, "TokenSale: Already Claimed");

        uint256 left;
        (s.share, left) = _claim(s);
        require(left > 0, "TokenSale: Nothing to claim");  // @audit - problematic line 
        uint256 refundTaxAmount;
        if (s.taxAmount > 0) {
            uint256 tax = userTaxRate(s.amount, msg.sender);
            uint256 taxFreeAllc = _maxTaxfreeAllocation(msg.sender) * PCT_BASE;
            if (taxFreeAllc >= s.share) {
                refundTaxAmount = s.taxAmount;
            } else {
                refundTaxAmount = (left * tax) / POINT_BASE; // tax refund is on the wrong amount 
            }
            usdc.safeTransferFrom(marketingWallet, msg.sender, refundTaxAmount);
        }
        s.claimed = true;
        usdc.safeTransfer(msg.sender, left);
        emit Claim(msg.sender, left);
    }
```
```solidity

    function _claim(Staked memory _s) internal view returns (uint120, uint256) {
        uint256 left;
        if (state.totalPrivateSold > (state.totalSupplyInValue)) {
            uint256 rate = (state.totalSupplyInValue * PCT_BASE) /
                state.totalPrivateSold;
            _s.share = uint120((uint256(_s.amount) * rate) / PCT_BASE);
            left = uint256(_s.amount) - uint256(_s.share);
        } else {
            _s.share = uint120(_s.amount);
        }

        return (_s.share, left);
    }
```
`left` only has value if the token has oversold. Meaning that even if the user has an infinite tax free allocation, if the token has not oversold, they won't be able to claim a tax refund. 


## Impact
loss of funds

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L377

## Tool used

Manual Review

## Recommendation
Remove the require check 

# Issue H-4: Reentrancy in Vesting.sol:claim() will allow users to drain the contract due to executing .call() on user's address before setting s.index = uint128(i) 

Source: https://github.com/sherlock-audit/2024-03-zap-protocol-judging/issues/157 

## Found by 
0rpse, 0x4non, 0xMosh, 0xR360, 0xhashiman, 404666, AMOW, BengalCatBalu, DimaKush, GatewayGuardians, HonorLt, NickV, Silvermist, UbiquitousComputing, Varun\_05, ZdravkoHr., aman, bughuntoor, cats, cawfree, denzi\_, dipp, enfrasico, klaus, mike-watson, nilay27, no, novaman33, octeezy, psb01, s1ce, thank\_you, turvec
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

# Issue M-1: Vesting contract cannot work with ETH, although it's supposed to. 

Source: https://github.com/sherlock-audit/2024-03-zap-protocol-judging/issues/54 

## Found by 
0x4non, AMOW, GatewayGuardians, HonorLt, NickV, ZdravkoHr., bughuntoor, cats, enfrasico, klaus, merlin, s1ce, thank\_you, ydlee
## Summary
Vesting contract cannot work with native token, although it's supposed to. 

## Vulnerability Detail
Within the claim function, we can see that if `token`  is set to address(1), the contract should operate with ETH
```solidity
    function claim() external {
        address sender = msg.sender;

        UserDetails storage s = userdetails[sender];
        require(s.userDeposit != 0, "No Deposit");
        require(s.index != vestingPoints.length, "already claimed");
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
                (bool sent, ) = payable(sender).call{value: pctAmount}("");  // @audit - here
                require(sent, "Failed to send BNB to receiver");
            } else {
                token.safeTransfer(sender, pctAmount);
            }
            s.index = uint128(i);
            s.amountClaimed += pctAmount;
        }
    }
```

However, it is actually impossible for the contract to operate with ETH, since `updateUserDeposit` always attempts to do a token transfer. 
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
        token.safeTransferFrom(distributionWallet, address(this), amount); // @audit - this will revert
    }
```

Since when the contract is supposed to work with ETH, token is set to address(1), calling `safeTransferFrom` on that address will always revert, thus making it impossible to call this function.

## Impact
Vesting contract is unusable with ETH

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Vesting.sol#L64

## Tool used

Manual Review

## Recommendation
make the following check 
```solidity
        if (address(token) != address(1)) token.safeTransferFrom(distributionWallet, address(this), amount);
```


# Issue M-2: `updateUserDeposit` overwrites current values 

Source: https://github.com/sherlock-audit/2024-03-zap-protocol-judging/issues/55 

## Found by 
Matin, bughuntoor, cats, cawfree, ydlee
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

# Issue M-3: `destroy` cannot be used as `selfdestruct`  is deprecated 

Source: https://github.com/sherlock-audit/2024-03-zap-protocol-judging/issues/59 

## Found by 
AMOW, bigbick123456789000, bughuntoor, cats, enfrasico, xiao
## Summary
`destroy` cannot be used as `selfdestruct`  is deprecated 

## Vulnerability Detail
Within the TokenSale contract, there's a `destroy`  function, which allows for an admin to take all USDC from the contract, transfer them to their wallet and then `selfdestruct` the contract 

```solidity
    function destroy() external override {
        _onlyAdmin();
        uint256 amountUSDC = usdc.balanceOf(address(this));
        if (amountUSDC > 0) {
            usdc.safeTransfer(admin.wallet(), amountUSDC);
        }
        address payable wallet = payable(admin.wallet());
        selfdestruct(wallet);
    }
```

Since the latest dencun update, the selfdestruct opcode has been deprecated and is no longer usable (unless called in the same tx the contract has been created, which is not the case here)

Meaning that this functionality can no longer exist. 

## Impact
selfdestruct functionality is unusable 

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L193

## Tool used

Manual Review

## Recommendation
remove the destroy function altogether

# Issue M-4: Blocklisted investors can still claim USDC in `TokenSale.sol` 

Source: https://github.com/sherlock-audit/2024-03-zap-protocol-judging/issues/82 

## Found by 
AMOW, Silvermist, ZdravkoHr., audithare, s1ce, ydlee
## Summary
A wrong argument is passed when checking if a user is blacklisted for claiming in [`TokenSale.claim()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L367). Because the check is insufficient, blocked users can claim their USDC.

## Vulnerability Detail
[`Admin.setClaimBlock()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/Admin.sol#L271-L273) blocks users from claiming. The function accepts the address of the user to be blocked and adds it to the `blockClaim` mapping.

```solidity
    /**
     @dev Whitelist users
     @param _address Address of User
     */
    function setClaimBlock(address _address) external onlyRole(OPERATOR) {
        blockClaim[_address] = true;
    }
```

The check in `Admin.claim()` wrongly passes `address(this)` as argument when calling `Admin.blockClaim`.

```solidity
        require(
            uint8(epoch) > 1 && !admin.blockClaim(address(this)),
            "TokenSale: Not time or not allowed"
        );
```
In this context, `address(this)` will be the address of the token sale contract and the require statement can be bypassed even by a blocked user.
## Impact
The whole functionality for blocking claims doesn't work properly.

## Code Snippet
```solidity
    function claim() external {
        checkingEpoch();
        require(
            uint8(epoch) > 1 && !admin.blockClaim(address(this)),
            "TokenSale: Not time or not allowed"
        );

        Staked storage s = stakes[msg.sender];
        require(s.amount != 0, "TokenSale: No Deposit");
        require(!s.claimed, "TokenSale: Already Claimed");

        uint256 left;
        (s.share, left) = _claim(s);
        require(left > 0, "TokenSale: Nothing to claim");
        uint256 refundTaxAmount;
        if (s.taxAmount > 0) {
            uint256 tax = userTaxRate(s.amount, msg.sender);
            uint256 taxFreeAllc = _maxTaxfreeAllocation(msg.sender) * PCT_BASE;
            if (taxFreeAllc >= s.share) {
                refundTaxAmount = s.taxAmount;
            } else {
                refundTaxAmount = (left * tax) / POINT_BASE;
            }
            usdc.safeTransferFrom(marketingWallet, msg.sender, refundTaxAmount);
        }
        s.claimed = true;
        usdc.safeTransfer(msg.sender, left);
        emit Claim(msg.sender, left);
    }
```
## Tool used

Manual Review

## Recommendation
Pass the address of the user.

```diff
        require(
-            uint8(epoch) > 1 && !admin.blockClaim(address(this)),
+            uint8(epoch) > 1 && !admin.blockClaim(msg.sender)),
            "TokenSale: Not time or not allowed"
        );
```

