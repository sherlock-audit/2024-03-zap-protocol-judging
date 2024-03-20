Lively Sandstone Vulture

medium

# Calls to `addToBlackList(address,address[])` can be frontrun.

## Summary

Calls to [`addToBlackList(address,address[])`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/Admin.sol#L149C14-L149C28) can be frontrun.

## Vulnerability Detail

The [`Admin`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Admin.sol#L149C14-L149C28) exposes blacklisting functionality which is designed to prevent [`OPERATOR`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/Admin.sol#L23C29-L23C37) defined accounts from participating in a [`TokenSale`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol):

```solidity
/**
 * @dev add users to blacklist
 * @param _blacklist - the list of users to add to the blacklist
 */
function addToBlackList(
    address _instance,
    address[] memory _blacklist
)
external
override
onlyIncoming(_instance)
onlyExist(_instance)
onlyRole(OPERATOR)
{
    require(_blacklist.length <= 500, "TokenSale: Too large array");
    for (uint256 i = 0; i < _blacklist.length; i++) {
        blacklist[_instance][_blacklist[i]] = true;
    }
}
```

The blacklist is enforced when attempting to make a [`deposit(uint256)`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L162):

```solidity
/**
 * @dev invest usdc to the tokensale
 */
function deposit(uint256 _amount) external {
    console.log("kyc enabled", isKYCEnabled);
    if (isKYCEnabled) {
        require(admin.isKYCDone(msg.sender) == true, "KYC not done");
    }
@>  address sender = msg.sender;
@>  require(
@>      !admin.blacklist(address(this), sender),
@>      "TokenSale: Blacklisted"
@>  );
    checkingEpoch();

    require(epoch == Epoch.Private, "TokenSale: Incorrect time");
    require(_amount > 0, "TokenSale: 0 deposit");

    if (userDepositIndex[sender] == 0) {
        usersOnDeposit.push(sender);
        userDepositIndex[sender] = usersOnDeposit.length;
    }
    if (epoch == Epoch.Private) {
        _processPrivate(sender, _amount);
    }
}
```

However, this call can be frontrun. If a blacklisted account [detects a pending transaction in the Blast mempool](https://docs.blast.io/building/transaction-finality#pending), they can make a call to [`deposit(uint256)`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L162) in an effort to secure a token allocation.

Importantly, the remainder of [`TokenSale`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol) functions remain operational to the sanctioned address, as the [`deposit(uint256)`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L162) function is the only call which is gated. This allows the blocked address to participate in the remainder of the [`TokenSale`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol) normally.

Similar issues can be observed for actively-participating accounts which are blacklisted retroactively.

## Impact

Blacklisted addresses can participate in a [`TokenSale`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol).

## Code Snippet

### 📄 [TokenSale.sol](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol)

## Tool used

Manual Review

## Recommendation

The sponsor should seek to impose additional controls on a [`TokenSale`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol) which reduce the incentive for blocked addresses to subvert the access control mechanism, such as by preventing shares from being [`claim()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L364C14-L364C21)ed, and exposing the ability for an [`OPERATOR`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/Admin.sol#L23C29-L23C37) to liquidate the stake of a blacklisted account:

```solidity
/**
 * @notice Liquidates the investment of a blacklisted account.
 * @dev Will revert if the account is not blacklisted or has not staked.
 * @param account The blacklisted.
 */
function liquidateBlacklistedAccount(address account) external {

   require(admin.hasRole(OPERATOR, msg.sender), "TokenSale: OnlyOperator");
   require(admin.blacklist(address(this), account), "TokenSale: AccountIsNotBlacklisted");
   
   Staked memory s = stakes[account];
   
   /// @dev Ensure the account participated else we risk corrupting the `usersOnDeposit`.
   require(s.amount > 0, "TokenSale: BlacklistedAccountDidNotParticipate");
   
   /// @dev Ensure non-blocked accounts can still participate with any remaining token allocation.
   state.totalPrivateSold -= uint128(amount);

   /// @dev Remove the blacklisted account's claim to the round.
   delete stakes[account];
   
   address temp = usersOnDeposit[usersOnDeposit.length - 1];
   uint256 purgeIndex = userDepositIndex[account] - 1;
   usersOnDeposit[purgeIndex] = temp;
   usersOnDeposit.pop();
   
   userDepositIndex[temp] = purgeIndex;
   delete userDepositIndex[account];
   
   /// @dev Claim the staked assets back to the `marketingWallet`.
   usdc.safeTransfer(marketingWallet, s.amount);
}
```
