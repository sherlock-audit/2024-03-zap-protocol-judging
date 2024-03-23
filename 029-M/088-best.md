Lively Sandstone Vulture

medium

# Rounding errors in `TokenSale` can lead to insolvency.

## Summary

Rounding errors when computing refunds for an over-allocated [`TokenSale`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol) can result in marginal insolvency.

## Vulnerability Detail

1. Let's issue a private token sale, and use a [`totalSupplyInValue`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/interfaces/ITokenSale.sol#L41C17-L41C35) of `1 ether`:

$$
totalSupplyInValue = 1 ether
$$

2. Now let's assume that three users each make a `0.8 ether` deposit (to simplify things, we'll assume there are no taxes applied. Taxes interact with the [`marketingWallet`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L37C20-L37C35) and induce fee calculations which are peripheral to the following setup, so this is an admissable simplification):

$$
totalPrivateSold = 2.4 ether
$$

3. Once the round is [`Finished`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/interfaces/ITokenSale.sol#L21C9-L21C17), because the [`TokenSale`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol) is overallocated, each user calls [`claim()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L364C14-L364C21) redeems at a rate of:

```solidity
uint256 rate =
  (state.totalSupplyInValue * PCT_BASE) / state.totalPrivateSold; // 416666666666666666
```

4. Next, let's calculate a single user's [`share`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L399) and [`left`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L400C13-L400C17):

```solidity
 _s.share = uint120((uint256(_s.amount) * rate) / PCT_BASE); // 333333333333333332
 _s.left = uint256(_s.amount) - uint256(_s.share); // 466666666666666668
```

> Each user receives `333333333333333332 wei` of token being sold and is permitted to claim `466666666666666668 wei` of their original deposit, resulting in a total obligation of `1400000000000000004 wei` in refunds.

5. Since `state.totalPrivateSold > state.totalSupplyInValue`, the owner can call [`takeUSDCRaised()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L337C14-L337C30) to extract the [`totalSupplyInValue`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/interfaces/ITokenSale.sol#L41C17-L41C35) (`1 ether`):

```solidity
if (state.totalPrivateSold > state.totalSupplyInValue) {
    earned = uint256(state.totalSupplyInValue); /// @audit owner_claims_full_sale
} else {
    earned = uint256(state.totalPrivateSold);
}
```

> ⚠️ This leaves `1.4 ether` remaining in the contract, however the **total obligation** is `1400000000000000004 wei`.

In this scenario, only two of the three investors are able to be refunded. The [`TokenSale`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol) has become insolvent.

The final investor's attempt to [`claim()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L364C14-L364C21) will `revert`, and runs the risk of being misconstrued as "locked funds" in a call to [`takeLocked()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L416C14-L416C26) after the [`30 days` grace period](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L419).

## Impact

Due to rounding errors, some addresses may not be able to receive their due refunds, as the contract will not have the necessary underlying token balance to execute a transfer.

## Code Snippet

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

## Tool used

Manual Review

## Recommendation

### 📄 [TokenSale.sol](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol)

When computing refunds for an over-allocated [`TokenSale`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol), do not compute the refund obligation based upon an individual user's stake.

Instead, compute their share of the global over-allocated stake.

```diff
- left = uint256(_s.amount) - uint256(_s.share);
+ left = ((state.totalPrivateSold - state.totalSupplyInValue) * rate) / PCT_BASE;
```

> Using this calculation, each user would be issued a `left` of `466666666666666664 wei` instead of `466666666666666668 wei`.

This results in a total obligation of `1399999999999999992 wei`, which does not result in insolvency.