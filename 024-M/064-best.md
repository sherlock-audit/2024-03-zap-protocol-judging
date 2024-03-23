Warm Gingerbread Kookaburra

medium

# `StakingContract`  can be changed within the Admin contract, but not within `TokenSale`

## Summary
`StakingContract`  can be changed within the Admin contract, but not within `TokenSale` 

## Vulnerability Detail
When deploying a `TokenSale`, `stakingContract` within it is set to the current stakingContract address within the Admin contract.
```solidity
    function initialize(
        Params calldata _params,
        address _stakingContract,
        address _admin,
        uint256 _maxAllocation,
        uint256 _globalTaxRate,
        bool _isKYC,
        uint256 _whitelistTxRate
    ) external initializer {
        params = _params;
        stakingContract = IStaking(_stakingContract);
        admin = IAdmin(_admin);
        state.totalSupplyInValue = uint128(
            (uint256(_params.totalSupply) *
                uint256(_params.privateTokenPrice)) / 10 ** 18
            // removes USDB hardcode of six, awful
        );
        usdc = IERC20D(0xA9F81589Cc48Ff000166Bf03B3804A0d8Cec8114); //TODO change for mainnet
        marketingWallet = 0x6507fFd283c32386B6065EA89744Ade21515e91E; //TODO change for mainnet
        maxAllocation = _maxAllocation;
        globalTaxRate = _globalTaxRate;
        isKYCEnabled = _isKYC;
        whitelistTxRate = _whitelistTxRate;
    }
``` 

The problem is that within the Admin contract, there's a way to change the `stakingContract`, in case it gets migrated or changed.
```solidity
    function setStakingContract(
        address _address
    ) external override validation(_address) onlyAdmin {
        stakingContract = _address;
        _setupRole(STAKING, address(_address));
    }
```

However, the TokenSale contract lacks such function. 

In case StakingContract gets changed, all pre-existing TokenSales will still call the old one, with no way to change this. Since the TokenSale contract is expected to be able to lock unstaking within the StakingContract, it is crucial that it is able to do so.


## Impact
TokenSale contract may not be able to call the current stakingContract 

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Admin.sol#L260C1-L266C1

## Tool used

Manual Review

## Recommendation

add a `setStakingContract` function within TokenSale 