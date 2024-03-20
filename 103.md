Rough Marmalade Alligator

medium

# The `Vesting::removeOtherERC20Tokens` does not check for `distributionToken`  removal.

## Summary
The function `removeOtherERC20Tokens` is meant to remove other tokens stuck in the contract , but there is not check to prevent removal of `distributionToken`.

## Vulnerability Detail
First I need to clear it that This function must be called by trusted user and the users is not suppose to withdraw `distributionToken` but  the issue here is that the function is intended to withdraw other tokens,  there for it is must to add check to prevent removal of `distributionToken` as it is done [here](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L202).


## Impact

The `distributionToken` will be removed.

## Code Snippet
In File `Vesting.sol`.
```solidity
   function removeOtherERC20Tokens(
        address _tokenAddress
    ) external onlyRole(DEFAULT_ADMIN_ROLE) {
        // @audit : the underliying token can also be withdraw here but the function is not meant to do it.
        IERC20(_tokenAddress).safeTransfer(
            distributionWallet,
            IERC20(_tokenAddress).balanceOf(address(this))
        );
    }
```

## Tool used

Manual Review

## Recommendation

```diff
@@ -121,6 +126,11 @@ contract Vesting is Initializable, AccessControl {
     function removeOtherERC20Tokens(
         address _tokenAddress
     ) external onlyRole(DEFAULT_ADMIN_ROLE) {
+        // @audit : the underlying token can also be withdraw here but the function is not meant to do it.
+        require(
+            _tokenAddress != address(token),
+            "TokenSale: Can't withdraw distribution"
+        );

```
