# Saved by the Bug: How One Small Error "Protected" a Contract from Being Hacked

> **Disclaimer:** This is a study case of a broken contract. It does not involve real-world risk or active hack possibilities. Publishing this is safe and purely educational.

I was recently diving into verified contracts on Etherscan and stumbled across an absolute goldmine for hackers. It’s a contract designed to perform arbitrage trades using Aave flashloans, but it’s... let’s just say it's not 100% secure.  

This contract is deployed under the address `0x422F9a7B66C1d95FAA749A019ADb85405cE6235f`. You can check it on Etherscan to get a more detailed view. I just pasted below the main/vulnerable contract. To keep simplicity, you can just focus on the `executeOperation` and `dualDexTrade` functions. Try to find some vulnerabities on this before reading the below explanation.

```solidity
// "SPDX-License-Identifier: MIT"
pragma solidity 0.6.12;

import {FlashLoanReceiverBase} from "./FlashLoanReceiverBase.sol";
import {ILendingPool} from "./ILendingPool.sol";
import {ILendingPoolAddressesProvider} from "./ILendingPoolAddressesProvider.sol";
import {IERC20} from "./IERC20.sol";

pragma experimental ABIEncoderV2;

interface IUniswapV2Router {
    function getAmountsOut(uint256 amountIn, address[] memory path)
        external
        view
        returns (uint256[] memory amounts);

    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);
}

interface IUniswapV2Pair {
    function token0() external view returns (address);
    function token1() external view returns (address);
    function swap(
        uint256 amount0Out,
        uint256 amount1Out,
        address to,
        bytes calldata data
    ) external;
}

/** !!!
    Never keep funds permanently on your FlashLoanReceiverBase contract as they could be 
    exposed to a 'griefing' attack, where the stored funds are used by an attacker.
    !!!
 */
contract DEXFlashLoan is FlashLoanReceiverBase {
    address public Rtoken;
    address public admin;
    address public owner;
    uint256 public fee;

    struct Trade {
        address swap1;
        address swap2;
        address token;
        address user;
    }

    constructor(
        address _addressProvider,
        address _Rtoken,
        address _admin
    )
        public
        FlashLoanReceiverBase(ILendingPoolAddressesProvider(_addressProvider))
    {
        Rtoken = _Rtoken;
        admin = _admin;
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    modifier onlyAdmin() {
        require(msg.sender == admin, "Not admin");
        _;
    }

    /**
        This function is called after your contract has received the flash loaned amount
     */
    function executeOperation(
        address[] calldata assets,
        uint256[] calldata amounts,
        uint256[] calldata premiums,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        Trade memory tradedata = abi.decode(params, (Trade));
        
        for (uint i = 0; i < assets.length; i++) {
            uint different = dualDexTrade(
                tradedata.swap1,
                tradedata.swap2,
                assets[i],
                tradedata.token,
                amounts[i]
            );
            uint amountOwing = amounts[i].add(premiums[i]);
            IERC20(assets[i]).approve(address(LENDING_POOL), amountOwing);
        }

        return true;
    }

    function ValobitFlashloan(
        address[] calldata assets,
        uint256[] calldata amounts,
        address[] calldata routers,
        address token
    ) public {
        address receiverAddress = address(this);
        uint256[] memory modes = new uint256[](assets.length);
        Trade memory trades = Trade(
            routers[0],
            routers[1],
            token,
            address(msg.sender)
        );

        for (uint i = 0; i < assets.length; i++) {
            modes[i] = 0;
        }

        address onBehalfOf = address(this);
        bytes memory params = abi.encode(trades);
        uint16 referralCode = 0;

        LENDING_POOL.flashLoan(
            receiverAddress,
            assets,
            amounts,
            modes,
            onBehalfOf,
            params,
            referralCode
        );
    }

    function swap(
        address router,
        address _tokenIn,
        address _tokenOut,
        uint256 _amount
    ) private {
        IERC20(_tokenIn).approve(router, _amount);
        address[] memory path;
        path = new address[](2);
        path[0] = _tokenIn;
        path[1] = _tokenOut;
        uint deadline = block.timestamp + 300;
        IUniswapV2Router(router).swapExactTokensForTokens(
            _amount,
            1,
            path,
            address(this),
            deadline
        );
    }

    function getAmountOutMin(
        address router,
        address _tokenIn,
        address _tokenOut,
        uint256 _amount
    ) public view returns (uint256) {
        address[] memory path;
        path = new address[](2);
        path[0] = _tokenIn;
        path[1] = _tokenOut;
        uint256[] memory amountOutMins = IUniswapV2Router(router).getAmountsOut(
            _amount,
            path
        );
        return amountOutMins[path.length - 1];
    }

    function estimateDualDexTrade(
        address _router1,
        address _router2,
        address _token1,
        address _token2,
        uint256 _amount
    ) external view returns (uint256) {
        uint256 amtBack1 = getAmountOutMin(_router1, _token1, _token2, _amount);
        uint256 amtBack2 = getAmountOutMin(_router2, _token2, _token1, amtBack1);
        return amtBack2;
    }

    function dualDexTrade(
        address _router1,
        address _router2,
        address _token1,
        address _token2,
        uint256 _amount
    ) public returns (uint) {
        uint startBalance = IERC20(_token1).balanceOf(address(this));
        uint token2InitialBalance = IERC20(_token2).balanceOf(address(this));
        
        swap(_router1, _token1, _token2, _amount);
        
        uint token2Balance = IERC20(_token2).balanceOf(address(this));
        uint tradeableAmount = token2Balance - token2InitialBalance;
        
        swap(_router2, _token2, _token1, tradeableAmount);
        
        uint endBalance = IERC20(_token1).balanceOf(address(this));
        require(endBalance > startBalance, "Trade Reverted, No Profit Made");
        
        uint different = endBalance - startBalance;
        return different;
    }

    function estimateTriDexTrade(
        address _router1,
        address _router2,
        address _token1,
        address _token2,
        uint256 _amount
    ) external view returns (uint256) {
        uint amtBack1 = getAmountOutMin(_router1, _token1, _token2, _amount);
        uint amtBack2 = getAmountOutMin(_router2, _token2, _token1, amtBack1);
        return amtBack2;
    }

    function getBalance(address _tokenContractAddress)
        external
        view
        returns (uint256)
    {
        return IERC20(_tokenContractAddress).balanceOf(address(this));
    }

    function recoverEth() external onlyOwner {
        payable(msg.sender).transfer(address(this).balance);
    }

    function changerewardtoken(address _rToken) public onlyAdmin returns (bool) {
        Rtoken = _rToken;
        return true;
    }

    function changefee(uint _fee) public onlyAdmin returns (bool) {
        fee = _fee;
        return true;
    }

    function changeadmin(address _admin) public onlyOwner returns (bool) {
        admin = _admin;
        return true;
    }

    function recoverTokens(address tokenAddress)
        external
        uint256
        onlyOwner
        returns (bool)
    {
        IERC20(tokenAddress).transfer(
            msg.sender,
            IERC20(tokenAddress).balanceOf(address(this))
        );
        return true;
    }

    receive() external payable {}
}
```

## Why was this a goldmine for bad actors?

Let’s look at the "Oopsie-Daisies" list:

* **Zero Access Control:** The `executeOperation` function (the callback called by Aave) has **no checks** on who is calling it. Anyone can trigger it and pass whatever parameters they want.
* **Arbitrary External Calls:** The `dualDexTrade` function is public and also accepts arbitrary parameters. It performs external calls to addresses provided by the caller. (Insert 💀 emoji here).
* **The Irony:** The contract includes a massive comment warning: *"Never keep funds permanently on your contract."* Despite this, the contract was sitting there with over **120 USDT** inside.

---

## The Attack Plan

On paper, this is a free 120 USDT gift. Here is how a hacker would naturally approach this:

1.  **Deploy a Malicious Router:** Create a contract that mimics a UniswapV2Router, specifically implementing a fake `swapExactTokensForTokens`.
2.  **Create a Junk Token:** Deploy a custom ERC20 with zero value to facilitate the "swap."
3.  **Trigger the Exploit:** Call `dualDexTrade` directly. Set `router1` and `router2` to the malicious router, `token1` to the victim's USDT, and `token2` to the junk token.
4.  **The Steal:**
    * `dualDexTrade` calls `swap`.
    * `swap` calls `approve` on the USDT, giving the malicious router permission to spend the victim’s 120 USDT.
    * The malicious router returns a fake `amounts` array to keep the victim contract happy.
    * In the second "swap" (Junk -> USDT), the malicious router sends a tiny bit of "dust" USDT back to the victim so the `require(endBalance > startBalance)` check passes.
5.  **Profit:** Now that the malicious router has an active allowance for the victim's USDT, the hacker just calls `transferFrom` and walks away 120 USDT richer.

---

## The Twist: Why the Attack Fails

You run the PoC, get your popcorn ready, and... **it reverts.** But why?

If the victim held literally almost any other token, this would work. But the victim holds **USDT**. 

USDT is notorious for not being 100% ERC20 compliant. Specifically, its `approve` function **do not return a boolean value.** However, the victim contract uses a standard `IERC20` interface:
`IERC20(_tokenIn).approve(router, _amount);`

In Solidity, when you define a function in an interface to return a `bool`, the EVM expects the target contract to return such boolean value. If the target (USDT) returns **nothing**, the calling contract interprets this as a failure and reverts the entire transaction.

So, ironically, a technical flaw in USDT's implementation is the only thing keeping those funds "safe" from a hacker. Of course, it also means the funds are stuck there forever unless the owner has a way to recover them that doesn't rely on that specific `approve` call!

---

## Practical Takeaways

1.  **Trust No One:** Never perform external calls to arbitrary addresses provided as parameters. Bad actors will use your contract as a proxy for their own schemes.
2.  **Use SafeERC20:** If you expect to interact with "weird" or non-standard tokens (like USDT), always use OpenZeppelin’s **SafeERC20** library or similars. It handles the missing return values gracefully.

**Bonus Thought:**
Even if we couldn't steal the funds directly due to the USDT bug, an attacker could still perform a **griefing attack**. They could trigger a flashloan from Aave and set the victim contract as the receiver. The victim would be forced to pay the flashloan premium, slowly draining their balance until it's gone. 

Now, go drink some water, keep hacking, and stay safe. See you in the next one!