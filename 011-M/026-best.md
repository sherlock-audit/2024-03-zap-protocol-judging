Agreeable Blood Yeti

medium

# Hardcoded `USDB` addresss in initializer is wrong

## Summary
`USDB` Address that is hardcoded in initializer is wrong.
## Vulnerability Detail
If we take a look at the `Admin.sol` contract in `zap-launches-contracts` we can see that the `USDB` address [used in the initializer](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-launches-contracts/contracts/Admin.sol#L63) is wrong:

```solidity
    function initialize(address _owner) public initializer {
        _setupRole(DEFAULT_ADMIN_ROLE, _owner);
        _setRoleAdmin(OPERATOR, DEFAULT_ADMIN_ROLE);
        wallet = _owner;
        platformFee = 2 * (10 ** 17);
        platformTax = 500;
@>      USDB = 0xA9F81589Cc48Ff000166Bf03B3804A0d8Cec8114;
    }
```
[Blast Mainnet Scan.](https://blastscan.io/address/0xA9F81589Cc48Ff000166Bf03B3804A0d8Cec8114)

If the protocol deploys with the wrong address, the contracts will need to be re-deployed since it's hardcoded and there is no setter function to change it.
## Impact
Protocol will need to re-deploy contracts.
## Code Snippet
```solidity
    function initialize(address _owner) public initializer {
        _setupRole(DEFAULT_ADMIN_ROLE, _owner);
        _setRoleAdmin(OPERATOR, DEFAULT_ADMIN_ROLE);
        wallet = _owner;
        platformFee = 2 * (10 ** 17);
        platformTax = 500;
@>      USDB = 0xA9F81589Cc48Ff000166Bf03B3804A0d8Cec8114;
    }
```
## Tool used
Manual Review
## Recommendation
[Correct USDB Address on Blast.](https://blastscan.io/token/0x4300000000000000000000000000000000000003)
