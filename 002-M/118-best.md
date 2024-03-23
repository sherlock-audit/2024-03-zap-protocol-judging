Melodic Garnet Hamster

medium

# Any user can call `Admin.sol::destroyInstance()` and destroy a Token Sale Instance.

## Summary

Any user can call the `destroyInstance()` function in Admin.sol in `zap-contracts-labs` and destroy a token sale instance.

## Vulnerability Detail

`destroyInstance()` in `Admin.sol` exists to destroy existing, and incoming instances. This function is external and can be called by any user. This function does not implement a check whether the `msg.sender` is an admin/operator. 

The function calls `Admin.sol::_removeFromSales()` and then calls `TokenSale.sol::destroy()` function.

The `TokenSale.sol::destroy()` function calls `_onlyAdmin()` to check if the `msg.sender` is the admin, since the msg.sender inside this function is the `Admin.sol` contract (cross contract function call). The check passes and transfers assets to to the `admin.wallet()` and self destructs.

### Proof of Concept

Add the following function in `TokenSale.sol` to use the mockUSDC for testing purposes

```solidity
function setUSDC(address _newUSDC) public {
        usdc = IERC20D(_newUSDC);
    }
```

Add the following function after creating a foundry setup
```solidity
function testDeletePoolByAnyone() public {
        TokenSale(payable(tokenSaleInstance1)).setUSDC(address(mockUSDC));

        vm.prank(bob);
        adminContract.destroyInstance(payable(address(tokenSaleInstance1)));
        vm.stopPrank();
    }
```

[`Admin.t.sol` with the setup and test](https://pastebin.com/dQ7bVMuM)


## Impact

Although the function makes sure that an ongoing Token Sale Instance can not be destroyed but any user can grief the protocol by destroying newly created Instances causing the creator to redeploy the instance. This causes the creator to spend more gas to recreate the same instance.

## Code Snippet

[`Admin.sol::destroyInstance()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Admin.sol#L138-L143)

```solidity
function destroyInstance(
        address _instance
    ) external onlyExist(_instance) onlyIncoming(_instance) {
        _removeFromSales(_instance);
        ITokenSale(_instance).destroy();
    }
```

[`TokenSale.sol::destroy()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L186-L194)

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

## Tool used

Manual Review , Foundry

## Recommendation

Add the following `onlyAdmin` Modifier on `Admin.sol::destroyInstance()`

```diff
function destroyInstance(
        address _instance
-    ) external onlyExist(_instance) onlyIncoming(_instance)
+    ) external onlyExist(_instance) onlyIncoming(_instance) onlyAdmin {
        _removeFromSales(_instance);
        ITokenSale(_instance).destroy();
    }
```