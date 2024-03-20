Interesting Cloth Poodle

high

# Incorrect Precision of PCT_BASE in `TokenSale.sol` will cause failure in `deposit` of USDC

## Summary
In `TokenSale.sol` the user deposit can happen via USDC, but the base multiplier for deposit amount is `PCT_BASE=10 ** 18`. 
This will severy functionality breakdown because USDC only works with 6 decimals.

## Vulnerability Detail
In `TokenSale.sol` we have a const PCT_BASE defined as:
```solidity
uint256 constant PCT_BASE = 10 ** 18; // @audit high the PCT_BASE is 10**18, for USDC should be 10**6
```
In the `deposit` method, we have the following code:
```solidity
function deposit(uint256 _amount) external {
   // rest of the code
   if (epoch == Epoch.Private) {
        _processPrivate(sender, _amount);
    }
}
```
where `_processPrivate` is defined as:
```solidity
function _processPrivate(address _sender, uint256 _amount) internal {
      require(_amount > 0, "TokenSale: Too small");

      Staked storage s = stakes[_sender];
      uint256 amount = _amount * PCT_BASE; // @audit _amount will be multiplied by 10**18
      /**
      * Rest of the code 
      */
      usdc.safeTransferFrom(_sender, address(this), amount); // amount transferred would be 10**12 times the actual deposit for USDC 

      /**@notice Forbid unstaking*/
      // stakingContract.setPoolsEndTime(_sender, uint256(params.privateEnd)); // TODO: uncomment
      emit DepositPrivate(_sender, _amount, address(this));
  }
```
## Impact
The deposit will most likely always FAIL with USDC as the actual amount will be `10 ** 12` intended deposit and most likely greater than the balence of User. 
This will Severly break down the functionality of TokenSale with USDC as deposits, which as confirmed by the Sponsor is a valid token to deposit.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L33

https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L182

https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L221-L248

## POC
Below is a foundry test for simulating the `deposit` with a `mockUSDC` token with 6 decimals. I also `initialize` the constructor argument in the `TokenSale` to accomodate for MockUSDC as it is currently hardcoded.


<details>
<summary> Foundry Test Code </summary>

```solidity
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {TokenSale} from "../src/TokenSale.sol";
import {ITokenSale} from "../src/interfaces/ITokenSale.sol";
import {IAirdrops} from "../src/interfaces/IAirdrops.sol";
import {IERC20D} from "../src/interfaces/IERC20D.sol";
import {IStaking} from "../src/interfaces/IStaking.sol";
import {MockUSDC} from "./mocks/mockUSDC.sol";
import {Admin} from "../src/Admin.sol";
// import {Staking} from "../src/Staking.sol";

contract TokenSaleTest is Test {
    TokenSale public tokenSale;
    address public user1 = makeAddr('user1');
    address public user2 = makeAddr('user2');
    address public adminOwner = makeAddr('adminOwner');
    address public operator = makeAddr('operator');
    address public staking = makeAddr('staking');
    MockUSDC public usdc; // 0xA9F81589Cc48Ff000166Bf03B3804A0d8Cec8114
    // // Prepare the bytecode for MockUSDC
    // bytes mockUSDCBytecode = type(MockUSDC).creationCode;
    Admin public adminContract;
   bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;
   bytes32 public constant OPERATOR = keccak256("OPERATOR");
    // Staking staking = new Staking();

    function setUp() public {
        ITokenSale.Params memory params;
        params.totalSupply = uint96(10**18);
        params.privateStart = uint32(block.timestamp);
        params.privateTokenPrice = uint96(10**18);
        params.privateEnd = uint32(block.timestamp + 100);
        tokenSale = new TokenSale();
        usdc = new MockUSDC();
        
        // setup the admin contract, owner, operator and set userKYC
        adminContract = new Admin();
        adminContract.initialize(adminOwner);
        vm.startPrank(adminOwner);
        adminContract.addOperator(operator);
        vm.stopPrank();
        // set kyc for user1 and user2
        vm.startPrank(operator);
        address[] memory users = new address[](2);
        users[0] = user1;
        users[1] = user2;
        adminContract.setUserKYC(users);
        vm.stopPrank();

        // setup the token sale contract
        tokenSale.initialize(params, staking, address(adminContract), 100*10**18, 0, true, 0, address(usdc));
        console.log(usdc.balanceOf(user1));
        usdc.mint(user1, 100*10**6);
        console.log(usdc.balanceOf(user1));
    }

    function stakingMockCalls() public{
        vm.mockCall(
            address(staking), 
            abi.encodeWithSelector(IStaking.getUserState.selector, address(user1)),
            abi.encode(1,1,1,1)
        );

        vm.mockCall(
            address(staking), 
            abi.encodeWithSelector(IStaking.getAllocationOf.selector, address(user1)),
            abi.encode(10)
        );

         // Mock for getAllocations function
        uint256 arg1 = 2; // Example argument 1
        uint256 arg2 = 1; // Example argument 2
        uint128 returnValue = 9; // Example return value

        vm.mockCall(
            address(staking), 
            abi.encodeWithSelector(IStaking.getAllocations.selector, arg1,arg2),
            abi.encode(returnValue)
        );

    }
    function test_deposit() public{
        vm.startPrank(user1);
        // tokenSale.deposit(1); // user wants to depoist 1 usdc
        console.log("kyc", adminContract.isKYCDone(user1));
        stakingMockCalls();
        usdc.approve(address(tokenSale), type(uint256).max);
        tokenSale.deposit(1); // user wants to depoist 1 usdc
        vm.stopPrank();
    }
    
}

```
</details>

<details> 
<summary> Console Output</summary>

```solidity
[128165] TokenSale::deposit(1)
    │   ├─ [631] Admin::isKYCDone(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   │   └─ ← true
    │   ├─ [2827] Admin::blacklist(TokenSale: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   │   └─ ← false
    │   ├─ [0] staking::getUserState(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   │   └─ ← 1, 1, 1, 1
    │   ├─ [0] staking::getAllocationOf(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   │   └─ ← 10
    │   ├─ [0] staking::getAllocations(2, 1)
    │   │   └─ ← 9
    │   ├─ [3322] MockUSDC::transferFrom(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], TokenSale: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 1000000000000000000 [1e18])
    │   │   └─ ← revert: ERC20: transfer amount exceeds balance
    │   └─ ← revert: ERC20: transfer amount exceeds balance
    └─ ← revert: ERC20: transfer amount exceeds balance
```

</details>

## Tool used

Manual Review, Foundry

## Recommendation
Instead of having a fixed value of `uint256 constant PCT_BASE = 10 ** 18`, have a mapping of acceptable Tokens that can be deposited into the contract