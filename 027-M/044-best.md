Cheesy Menthol Walrus

high

# Native ETH in the `Vesting.sol` contract cannot accrue yield

## Summary
Native ETH residing in the `Vesting.sol` contract cannot accrue yield due to lacking implementations to do so. Blast is a yield-bearing chain. But the default yield mode for native ETH is VOID.  Since the contract does not configure the yield mode, yield is never accrued.

## Vulnerability Detail
The vesting contract is meant to receive ETH. This can be confirmed by the `claim()` function which sends native ETH from the contract to sender as seen in the line below.

```javascript
(bool sent, ) = payable(sender).call{value: pctAmount}(""); 
```

Since there are native ETH residing in the contract, protocol is losing out on the yield meant to be accrued on Blast, and there is no way to activate it.

## Impact
Loss of yield in vesting contract.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Vesting.sol#L84

## Tool used

Manual Review

## Recommendation
Add a wrapper around the vesting contract which allows interoperability with Blast chain.

