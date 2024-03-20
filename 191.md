Energetic Daisy Ladybug

high

# `state.totalPrivateSold`  and `state.totalSupplyInValue` don't have same units so comparison will result in unexpected behavior

## Summary
From the `ITokenSale.State` we can see that `totalPrivateSold` and `totalSupplyInValue` variables which tracks the amount deposited and total initial supply initialized for tokensale. 

From the implememtation of tokensale, `totalSupplyInValue` remains in token decimals while `totalPrivateSold` is scaled up to decimals * 1e18 which can cause serious damage to the protocol


## Vulnerability Detail
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
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L394

`claim()` calls above internal  `_claim`  function which compares `state.totalPrivateSold > state.totalSupplyInValue` to compute rate and shares. There are multiple places where this comparison is crucial.
 
Now let's try to examine units of both params.

**totalSupplyInValue**

For tokensupplly of 1000000 * 1e18 (token as 18 decimals) and token price is 150USDC
( total supply with 18 decimal) * (USDC with 6 decimal) / 18 decimal => so the end result will be in 6 decimal
that means total supply valuation in USDC.
for example
(1000000 * 1e18) * (150 * 1e6) / (1e18) = 150000000 * 1e6

**totalPrivateSold**

Let's deposit sometokens. 150 USDC in this example
User calls deposit function to deposit the tokens
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L162

At line [221](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L221) amount is scaled up by multiplying with 1e18 

and this amount is added to the line [247](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L247) making totalPrivateSold inflated.

If we observe the variables
amount = 150*1e6 * PCT_BASE (PCT_BASE=1e18)
sum = amount ( for first deposit)
and then amount is added to the totalPrivateSold without downscaling 

I believe amount was upscaled for comparison with maxAllocationOfUser hence s.amount and amount need to be in the USDC * PCT_BASE  precision but `totalPrivateSold` is compared with 
`totalSupplyInValue` which is in different units so it should be handled differently

Both variables ( `totalPrivateSold` and `totalSupplyInValue`) should have same units(i.e decimals) for correct comparison 

## Impact
incorrect comparison 

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L394

https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L162

https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L221

(https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L247

## Tool used

Manual Review

## Recommendation
 Sync the decimals of `totalPrivateSold` and `totalSupplyInValue`