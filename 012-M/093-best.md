Cold Fossilized Penguin

medium

# Users can avoid new tax rates and ```maxAllocation``` when ```block.timestamp == params.PrivateStart```

## Summary

The 1 second overlap that occurs when ```TokenSale.setAllocationandTax()``` can be called and the ```TokenSale.deposit()``` function can be called means users can avoid any new tax rates or max allocation when ```TokenSale.setAllocationandTax()``` is called at the timestamp of ```params.PrivateStart```.

## Vulnerability Detail

When the operator calls ```TokenSale.setAllocationandTax()``` at time ```block.timestamp == params.PrivateStart```. This sets a new ```maxAllocation```, ```globalTaxRate``` and ```whitelistTxRate```. Since the ```checkingEpoch()``` function also sets the Epoch to private at ```block.timestamp == params.PrivateStart```, this means we can call the ```deposit``` function here. At this timeframe, the user can monitor the mempool for the transaction and then call the deposit function, before the changes take to the states of the tax rates and ```maxAllocation``` placed by the operator. As the current state of the epoch is ```epoch == Epoch.Private```, the ```_processPrivate``` is therefore able to pass. This means the depositor can use an out of date value for the ```maxAllocation```, ```globalTaxRate``` and ```whitelistTxRate``` variables.

## Impact

This means the user who is depositing can use the previous ```maxAllocation```, ```globalTaxRate``` and ```whitelistTxRate``` when the operator sets the Allocation or Tax Rates at ```block.timestamp == params.PrivateStart```. This is particularly unfair if the previous rates were more beneficial for the depositer. i.e. they had lower tax rates or a higher ```maxAllocation```. This undermines the integrity of the protocol as users who deposit after wont have the same rates or restrictions.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L119

https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L138

https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L174

## Tool used

Manual Review

## Recommendation

Change this line so that the operator can only change the allocation and tax values before the start time.

```diff
    function setAllocationAndTax(uint256[3] calldata _allocations) external {
-        require(block.timestamp <= params.privateStart, "Time lapsed");
+       require(block.timestamp < params.privateStart, "Time lapsed");
        require(admin.hasRole(OPERATOR, msg.sender), "TokenSale: OnlyOperator");
        maxAllocation = _allocations[0];
        globalTaxRate = _allocations[1];
        whitelistTxRate = _allocations[2];
    }

```
