Warm Gingerbread Kookaburra

medium

# `destroy` cannot be used as `selfdestruct`  is deprecated

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