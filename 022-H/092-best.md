Soaring Holographic Millipede

high

# Attacker can drain `marketingWallet` with reentrancy

## Summary
`TokenSale::claim` function has reentrancy vulnerability. This vulnerability allows an attacker to drain the `marketingWallet`.

## Vulnerability Detail
If `state.totalPrivateSold` is bigger than `state.totalSupplyInValue` and if there is collected tax in `marketingWallet`, attacker can drain all money in this wallet. `TokenSale::claim` function changes `s.claimed = true` after sending token making this function vulnerable to reentrancy attack. 

## Impact
Attacker can steal all taxes collacted in `marketingWallet`. 

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L364-L392

## Tool used
Manual Review

## Recommendation
Make state changes before transfer. 
```diff
    function claim() external {
        checkingEpoch();
        require(
            uint8(epoch) > 1 && !admin.blockClaim(address(this)),
            "TokenSale: Not time or not allowed"
        );

        Staked storage s = stakes[msg.sender];
        require(s.amount != 0, "TokenSale: No Deposit");
        require(!s.claimed, "TokenSale: Already Claimed");
+	s.claimed = true;
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
-       s.claimed = true;
        usdc.safeTransfer(msg.sender, left);
        emit Claim(msg.sender, left);
    }
```