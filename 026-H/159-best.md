Attractive Hazel Buffalo

high

# Users can get back their whole taxAmount whenever state.totalPrivateSold > (state.totalSupplyInValue)

## Summary
Users can cleverly deposit large number of tokens i.e upto their maxAllocation so that state.totalPrivateSold > (state.totalSupplyInValue) and as this happens they are eligible to claim their shares back and as well as their whole tax amount .So if a user wanted to only deposit some amount of tokens but now as they know that after totalPrivateSold exceeds totalSupplyvalue they would get their excess shares and wholetaxAmount if they deposit tokens in a way that makes the taxFreeAllc >= s.share. The users can view the values of both totalPrivateSold and totalSupplyInValue through view function.

## Vulnerability Detail
Lets understand the flow 
First deposit is called by the users which main function here is the _processPrivate function which is as follows 
```solidity
function _processPrivate(address _sender, uint256 _amount) internal {
        require(_amount > 0, "TokenSale: Too small");

        Staked storage s = stakes[_sender];
        uint256 amount = _amount * PCT_BASE;
        uint256 sum = s.amount + amount;

        uint256 maxAllocationOfUser = (calculateMaxAllocation(_sender)) *
            PCT_BASE;
        require(sum <= maxAllocationOfUser, "upto max allocation");
        uint256 taxFreeAllcOfUser = 0; // hardcode zero - all pools have ax

        uint256 userTaxAmount;

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
        s.amount += uint128(amount);
        state.totalPrivateSold += uint128(amount);
        usdc.safeTransferFrom(_sender, address(this), amount);

        /**@notice Forbid unstaking*/
        // stakingContract.setPoolsEndTime(_sender, uint256(params.privateEnd)); // TODO: uncomment
        emit DepositPrivate(_sender, _amount, address(this));
    }
```
According to the implementation  uint256 taxFreeAllcOfUser = 0
A user has to pay tax for every amount of tokens they deposit which is equal to the following userTaxAmount = (amount * userTxRate) / POINT_BASE

Now a situation arises when the  state.totalPrivateSold = (state.totalSupplyInValue) so if  now users deposit tokens such that  for exmple state.totalPrivateSold = 2*(state.totalSupplyInValue) and now the tokenSale is finished.
So now users eligible to call the claim function.
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
        s.claimed = true;
        usdc.safeTransfer(msg.sender, left);
        emit Claim(msg.sender, left);
    }
```
Main calculation is done in the following function  (s.share, left) = _claim(s);
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
So as we have assumed state.totalPrivateSold = 2*(state.totalSupplyInValue) 

so rate = 1/2(neglecting decimals as of now otherwise 5e17)
so now _s.share = 1/2(_s.amount)
left = 1/2(_s.amount)
(Just using some values obviously it can differ)

Now lets look at the following lines which are important 
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
As tax is paid for every deposit s.taxAmount would obviously be true
Now taxFreeAllc is calculated using the _maxTaxFreeAllocation function so assume it returns some value
Now issue lies in the if condition 
as we can see if taxFreeAllc >= s.share user if refunded whole s.taxAmount , as initially when a user calls deposit function he pays tax for whole amount of tokens because it was hardcoded  taxFreeAllcOfUser = 0 but now taxFreeAllc is taken into account so now users can carefully deposit tokens so that state.totalPrivateSold  becomes greater than (state.totalSupplyInValue) and such a ratio is acheived such that taxFreeAllc >= s.share and they get their taxAmount back and also deposit tokens.Now not only they get back their full taxAmount back they also reduce the share amount of other users and for those users whose taxFreeAllc < s.share they only get refundTaxAmount = (left * tax) / POINT_BASE which is correct according to the current implementation.


It is a logical error in the code because first they assume that user has to pay tax for every deposit but in claim they take into account taxFreeAllocation thus allowing users to not pay tax and get shares(deposit tokens) . This also can impact the protocol as some users can get back full tax amount instead they should only get back the tax which was applied to left amount. The above code is correct if initially in deposit function they hadn't hardcoded taxFreeAllocUser = 0 but it is not the case in the current codebase.
## Impact
Users can pay no tax amount and also reduce shares of other users.
## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L383
## Tool used

Manual Review

## Recommendation
refundTaxAmount should be equal to (left * tax) / POINT_BASE  in either cases because of the hardcoded taxFreeValue = 0