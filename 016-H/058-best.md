Warm Gingerbread Kookaburra

high

# If token does not oversell, users cannot claim tax refund on their tax free allocation.

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
