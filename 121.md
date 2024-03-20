Melodic Garnet Hamster

high

# Insufficient Validation on user inputs for `Admin.sol::createPoolNew` in `zap-launches-contracts` can lead to instances susceptible to DoS

## Summary

The `zap-launches-contracts` includes `Admin.sol` which provides permissionless creation of Token Sale Instances. Users can create TokenSales for either ETH based or USDB based using the `Admin.sol::createPoolNew()` function. The argument `_maxAllocation` and `_params.liqudityPercentage` can cause DoS when the creator/user calls the `takeUSDBRaised()` function in the instance due to insufficient validations on user input in `Admin.sol::createPoolNew()`.

## Vulnerability Detail

The user inputs `_params`, `_config`, `_maxAllocation`, `_isKYC`, and `isETHBased`. Out of these parameters the `_maxAllocation` and `_params.liqudityPercentage` can introduce DoS for the creator/user when calling `takeUSDBRaised()`. 

The checks performed in [`_checkingParams()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-launches-contracts/contracts/Admin.sol#L208-L222) are insufficient which create possibility of DoS in the newly created instances.

### Case 1:

[Case 1 Code](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-launches-contracts/contracts/TokenSaleUSDB.sol#L197-L204)

The `_params.liquidityPercentage` is used to calculate the `earned` variable inside `takeUSDBRaised()`
```solidity
earned = (earned * (POINT_BASE - params.liqudityPercentage)) / POINT_BASE;
```
This earned value is then used in the following line of code to transfer the amount which is calculated
```solidity
USDB.safeTransfer(creator, earned - earning);
```

If the `_params.liquidityPercentage` is passed in by the user between `9955` to `10000` (inclusive), the `earned` value will be less than `earning` and the function `USDB.safeTransfer()` will revert.

#### Mathematical example
This is due to Rounding

```solidity
uint256 earned = state.totalSold > state.totalSupplyInValue
            ? state.totalSupplyInValue
            : state.totalSold;
```

Suppose `earned` here is `45` then we calculate earning and earned(again) inside the `if (earned > 0)`
```solidity
uint earning = (earned * (platformTax)) / POINT_BASE;
```
Suppose `earning` is `2`

```solidity
earned = (earned * (POINT_BASE - params.liqudityPercentage)) / POINT_BASE;
```
`earned = (45 * (10000 - 9556)) / 10000` becomes `1.998` which rounds down to `1`.
Substituting `earned - earning` i.e `1 - 2`, will result in an underflow causing the function to revert.

Note: Values above are taken from the PoC given down below

### Case 2:

[Case 2 Code](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-launches-contracts/contracts/TokenSaleUSDB.sol#L208)

The `_maxAllocation` argument decides the `state.totalSupplyInValue` and `tokenPrice` of the newly created Instance. On certain values of `_maxAllocation` i.e `_maxAllocation < 10000` The `tokenPrice` can become 0.

The `_maxAllocation` argument is used to calculate the `tokenPrice` using
```solidity
tokenPrice =
    (_maxAllocation * 10 ** IERC20D(params.tokenAddress).decimals()) / _params.totalSupply;
```

If the `tokenPrice` in the instance becomes 0, important division which uses `tokenPrice` as denominator will panic revert such as

```solidity
uint unsoldTokens = (unsold * (10 ** ETH_DECIMAL)) / tokenPrice;
```
this line will throw a panic error : division or modulo by zero causing the function to revert. 

### Proof of Concept

Add the following setUp in `Admin.t.sol`

```solidity
contract MockERC20D is ERC20 {
    constructor() ERC20("MockERC20D", "MERC20D") {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}

interface ITokenSaleUSDB {
    function deposit(uint256 amount) external;

    function claim() external;

    function setUSDB(address) external;

    function takeUSDBRaised() external;
}

contract AdminTest is Test {
    address deployer = makeAddr("deployer");

    Admin admin;

    address creator = makeAddr("creator");
    address bob = makeAddr("bob");
    address alice = makeAddr("alice");

    address operator = makeAddr("operator");

    address wallet_ = makeAddr("wallet_");

    TokenSaleETH tokenSaleETH;
    TokenSaleUSDB tokenSaleUSDB;

    MockERC20D erc20d;

    uint256 constant STARTING_AMOUNT = 1 ether;

    address tokenSaleInstance;

    receive() external payable {}

    function setUp() public {
        deployer = address(this);

        admin = new Admin();

        admin.initialize(deployer);

        tokenSaleUSDB = new TokenSaleUSDB();

        admin.setMasterContractUSDB(address(tokenSaleUSDB));

        admin.grantRole(keccak256("OPERATOR"), operator);
        admin.grantRole(keccak256("OPERATOR"), address(this));

        erc20d = new MockERC20D();

        vm.deal(creator, 1 ether);
        erc20d.mint(creator, 100000e18);
        erc20d.mint(bob, 10000e18);
        erc20d.mint(alice, 10000e18);

        address tokenAddress = address(erc20d);
        address router = address(0x456);
        // Set up your ITokenSale.Params and ITokenSale.Config structs
        // Example values for Params
        ITokenSale.Params memory params = ITokenSale.Params({
            tokenAddress: tokenAddress,
            router: router,
            totalSupply: 10000e18,
            saleStart: uint96(block.timestamp + 1 days),
            saleEnd: uint96(block.timestamp + 5 days),
            liqudityPercentage: 9550,
            tokenLiquidity: 100e18,
            baseLine: 1e18,
            burnUnsold: false
        });

        // Example values for Config
        ITokenSale.Config memory config = ITokenSale.Config({
            LPLockin: 30 days,
            vestingPeriod: 90 days,
            vestingDistribution: 180 days,
            NoOfVestingIntervals: 6,
            FirstVestPercentage: 2000,
            LiqGenerationTime: uint96(block.timestamp + 11 days)
        });

        // Assume platformFee is set to 0.2 ether in the Admin contract
        uint256 platformFee = 0.2 ether;

        // Simulate sending the platform fee with the function call
        vm.prank(creator);
        erc20d.approve(address(admin), 100000e18);
        vm.stopPrank();

        vm.prank(creator);
        tokenSaleInstance = admin.createPoolNew{value: platformFee}(
            params,
            config,
            1000,
            false,
            false
        );
        vm.stopPrank();

        ITokenSaleUSDB(tokenSaleInstance).setUSDB(address(erc20d));
    }
}
```

Add the following test into the test contract
```solidity
function testUSDBRaised() public {
        vm.warp(block.timestamp + 1 days);

        // Approve and deposit for Bob

        vm.prank(bob);
        erc20d.approve(address(tokenSaleInstance), 10000);
        vm.stopPrank();

        vm.prank(bob);
        ITokenSaleUSDB(tokenSaleInstance).deposit(25);
        vm.stopPrank();

        // Approve and deposit for Alice
        vm.prank(alice);
        erc20d.approve(address(tokenSaleInstance), 10000);
        vm.stopPrank();

        vm.prank(alice);
        ITokenSaleUSDB(tokenSaleInstance).deposit(20);
        vm.stopPrank();

        // Move time forward to allow claiming
        vm.warp(block.timestamp + 6 days);

        ITokenSaleUSDB(tokenSaleInstance).takeUSDBRaised();
    }
```

### Case 1: 
Set the `liqudityPercentage` > `9555` and run the test.

### Case 2:
Set the `liqudityPercentage` <= `9555` and set `_maxAllocation` < `10000` run the test.



## Impact

The function `takeUSDBRaised()` is an important function which sets 
```solidity
isRaiseClaimed = true;
```  
This function is present in both the possible instances that can be created from the `Adminsol::createPoolNew()`


If `isRaiseClaimed` is not set to true, other users of the tokenSale will not be allowed to complete the calls to `claim()`, `addLiq()`, `claimLP()`, `vesting()`  function as these require `isRaiseClaimed` to be set as true.

This causes DoS for most of the functions of the Token Sale contracts and users are not able to execute their desired functions.

## Code Snippet

[Admin.sol::createPoolNew()](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-launches-contracts/contracts/Admin.sol#L288-L316)

[Admin.sol::_checkingParams](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-launches-contracts/contracts/Admin.sol#L208-L222)

[TokenSaleETH::takeUSDBRaised()](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-launches-contracts/contracts/TokenSaleETH.sol#L166-L188)

[TokenSaleUSDB::takeUSDBRaised()](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-launches-contracts/contracts/TokenSaleUSDB.sol#L192-L217)

## Tool used

Manual Review, Foundry

## Recommendation

Add the necessary checks for `_maxAllocation` and calculate/set the maximum allowed `_params.liquidityPercentage` in `Admin.sol::_checkingParams`

```diff
-function _checkingParams(ITokenSale.Params memory _params, ITokenSale.Config memory _config) internal view
+function _checkingParams(ITokenSale.Params memory _params, ITokenSale.Config memory _config, uint256 _maxAllocation) internal view
{
        require(_params.totalSupply > 0, "TokenSale: TotalSupply > 0");
        require(
            _params.saleStart >= block.timestamp,
            "TokenSale: Start time > 0"
        );
        require(
            _params.saleEnd > _params.saleStart,
            "TokenSale: End time > start time"
        );
-       require(_params.liqudityPercentage > 0 && _params.liqudityPercentage <= POINT_BASE,
-            "TokenSale: LiqudityPercentage Invalid");
+       require(_params.liqudityPercentage > 0 && _params.liqudityPercentage <= 9555, 
+            "TokenSale: LiqudityPercentage Invalid");
        require(_config.NoOfVestingIntervals > 0, "NoOfVestingIntervals can't be Zero");
        require(_config.FirstVestPercentage > 0, "FirstVestPercentage can't be Zero");
+       require(_maxAllocation >= 10000, "_maxAllocation can't be less than 10000");
    }
```