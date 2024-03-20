Warm Gingerbread Kookaburra

high

# User will get wrong amount of tax refunded, if their tax rate changes

## Summary
User will get wrong amount of tax refunded, if their tax rate changes

## Vulnerability Detail
Upon every deposit in TokenSale, every user has to pay a certain amount of tax, based on their tax rate. 
```solidity
       if (sum > taxFreeAllcOfUser) {
            uint256 userTxRate = userTaxRate(sum, _sender);
            if (s.amount < taxFreeAllcOfUser) {
                userTaxAmount =
                    ((sum - taxFreeAllcOfUser) * userTxRate) /
                    POINT_BASE;
            } else {
                userTaxAmount = (amount * userTxRate) / POINT_BASE;
            }
        }

        if (userTaxAmount > 0) {
            s.taxAmount += userTaxAmount;
            usdc.safeTransferFrom(_sender, marketingWallet, userTaxAmount);
        }
```
After the private period finishes, users can get a tax refund. The problem is that if the user has used part of their non-tax-free allocation, their current tax rate is used to calculate the refund, and not their old one.

```solidity
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

Imagine the following scenario: 
1. Global tax rate is 1%
2. User makes 1000 USDC deposit, meaning that they pay 10 USDC in tax
3. Global tax rate is changed to 10% 
4. User has 500 USDC as max taxfree allocation, meaning that he must get a refund on the tax of these 500 USDC
5. When calculating the 500 tax refund, it will based on the new 10% tax rate. The user will be refunded 50 USDC, even though initially he paid only 10 USDC in tax 

## Impact
Loss of funds.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L381

## Tool used

Manual Review

## Recommendation
correctly calculate the tax refund 