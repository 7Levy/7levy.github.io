# Master Solidity-极限优化合约Gas



## 合约Gas优化

### Solidity optimization

在我们编译solidity智能合约的时候，我们需要指定一个参数来告诉solidity编译器生成优化的字节码。如果你不使用优化参数的话，那么就会花费更多的gas。

在实际的开发中，这种优化方式会更长的编译时间，默认情况下是关闭的。以Hardhat为例，我们可以在配置文件hardhat.config.js中启用，并设置你所期望的运行次数，runs假定代码的每个操作码在合约生命周期内执行的次数。runs较小时(如1)，初始字节码较短，部署成本低，但会导致后期合约执行成本高。runs较大时(如1000)，初始字节码较长且复杂，部署成本更高，但对合约的调用会更便宜。对于这个参数，我们需要针对所写的合约来设置，如果合约只是一次性的代码，设置成1即可。对于ERC20代币之类的合约，设置较大的数字能显著地减少调用成本。

```JavaScript
module.exports = {
  solidity: {
    version: "0.8.1",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,
      },
    },
  },
};
```

### 优化变量声明顺序

solidity数据存储在32字节的slots中，采用紧密打包的模式存放数据。因此我们可以通过内存对齐的方式，来减少变量占用的插槽数(需要打开optimizer功能)。如下面的例子。

```
//占用16+16+32，总共2个slots
uint128 a;
uint128 b;
uint256 b;

// 占用32+32+16，总共3个slots
uint128 a;
uint256 b;
uint128 b;
```

### 使用字面量代替重复计算的值

在合约中的一些操作，如预先计算合约地址，计算keccack256哈希值，获取创建字节码等，我们可以在编译测试合约的时候获取到字面值，在代码中直接写入它们的字面量，这样就不需要每次调用函数来获取值。

```
//字面量
bytes32 constant hash = '68b3465833';

//计算
bytes32 constant hash = keccack256(abi.encodePacked('xxxx'));
```

### 使用代理合约克隆合约

如果需要克隆多个相同的智能合约，我们需要使用ERC1167标准定义的最小代理合约来将代码执行操作转发到引用的合约上，并在自己的上下文使用它。这样我们就不要每次部署同样的合约代码。这能大大地减少我们的部署成本。

### 内联汇编

当你编译一个智能合约的时候，它会被转换为EVM操作码。我们称之为字节码。我们不用直接编写EVM字节码，我们可以使用solidity内联汇编的方式来优化一些操作。如使用creat2预先计算合约地址。虽然内联汇编的方式会带来更多的复杂和不确定性，但也能提供更细粒度的代码控制和执行成本。如下代码所示，使用内联汇编的方式我们可以控制布尔值的赋值成本。

```
contract GasOptimize{
    function isTrue()public pure{
        bool a;
        assembly{
            a:=true//21198
            a:=not(0)//21190
            a:=0x1//21187
        }
    }
}
```

除此之外，我们还可以使用assembly打包变量到一个插槽中。

```
//打包uint64变量_a,_b,_c,_d到一个slot中
function encode(uint64 _a, uint64 _b, uint64 _c, uint64 _d) internal pure returns (bytes32 x) {
    assembly {
        let y := 0
        mstore(0x20, _d)
        mstore(0x18, _c)
        mstore(0x10, _b)
        mstore(0x8, _a)
        x := mload(0x20)
    }
}
```

```
//解码
function decode(bytes32 x) internal pure returns (uint64 a, uint64 b, uint64 c, uint64 d) {
    assembly {
        d := x
        mstore(0x18, x)
        a := mload(0)
        mstore(0x10, x)
        b := mload(0)
        mstore(0x8, x)
        c := mload(0)
    }
}
```

### 短路规则

以f(x)||g(x)为例，如果f(x)为true，那么就不会执行g(x)函数。因此，在涉及这类操作时，我们可以把成本更高的操作码放在后面。当f(x)成本小于g(x)，我们可以按以下规则编码。

- OR：f(x) || g(x)
- AND：f(x) && g(x)

在成本接近的时候，我们还可以根据返回值的几率来设置短路规则。如果g(x)返回true的概率大于f(x)，OR操作需要把g(x)放在前面，如果g(x)返回false 的概率大于f(x)，AND操作需要把g(x)放在前面。

### 循环中使用局部变量

第一段代码如下，sum在每次循环中都读取和写入，storage的执行成本相对于memory来说更高。

```
    uint256 sum = 0;
    function test(uint256 x) public {
        for (uint256 i = 0; i < x; i++) sum += i;
    }
```

所以我们可以在循环中引入memory类型的临时变量以减少循环执行的成本。

```
    uint256 sum = 0;
    function p3(uint256 x) public {
        uint256 temp = 0;
        for (uint256 i = 0; i < x; i++) temp += i;
        sum += temp;
    }
```

### 不使用默认值去初始化变量

变量在没有被初始化的时候，会有一个缺省值。如果显式地使用它的默认值初始化它，这只会浪费你的gas。

```
uint256 hello = 0; //bad, expensive
uint256 world; //good, cheap
```

### 使用较短的reason字符串

在require语句中我们会附加错误原因，这些字符串也会占用部署的字节码的空间。每个字符串至少需要32字节，我们需要尽量保持reason的精简。

```
require(balance >= amount, "Insufficient balance"); //good

require(balance >= amount, "To whomsoever it may con·cern. I am writing this error message to let you know that the amount you are trying to transfer is unfortunately more than your current balance. Perhaps you made a typo or you are just trying to be a hacker boi. In any case, this transaction is going to revert. Please try again with a lower amount. Warm regards, EVM"; //bad
```

###单行交换和三元运算符

我们可以使用单行交换的方式来改变变量值，不需要临时变量等来交换值。

```
(hello, world) = (world, hello)
```

solidity支持三元运算符的方式来赋值，这可以减少冗余的if-else代码。以下为uniswap的token地址排序参考代码。

```
(token0, token1) = tokenA < tokenB? (tokenA, tokenB):(tokenB, tokenA);
```

### 减少内联造成的字节码冗余浪费

modifier修饰符会将代码放入_位置处，这是一种内联操作，虽然能提高效率，但会造成字节码的浪费。EIP170规定合约的大小限制在24KB以内，如果一个代码被内联多次，那么合约的大小会大大地增加。

我们可以在modifier中直接调用internal函数，internal函数作为独立的函数调用，运行时代价略高，但减少了冗余的字节码。同时，internal函数可以避免堆栈过深的错误，在internal函数中创建的变量和原始函数不共享受限堆栈。可以参考polymath的具体实现[DataStore.sol](https://github.com/PolymathNetwork/polymath-core/pull/548/commits/2dc0286f4e96241eed9603534607431a8a84ba35#diff-8b6746c2c4e7c9e3fca67d62718d70e8)。

### 尽量使用256位的变量

EVM一次只对32字节/256位进行操作，如果使用uint8之类的变量，EVM会进行转换操作，转换操作需要额外的汽油。

- 使用uint256代替uint8
- 使用bytes32代替string/bytes、bytes1等

注：小变量在结构体和数组中以及其他可以紧密打包的场景下更有效。

### 使用external修饰符

对于public函数，输入参数会复制到memory中，会花费gas。如果你的函数只在外部调用，应该显式地标记成external。external函数不会复制到memory中，而是直接从calldata中读取。当输入参数很大时，可以节省大量的时间。

### 删除不需要的变量

在Ethereum中，如果你释放了存储空间，你会得到gas退款。我们可以使用delete关键字或者是设置为默认值来删除变量。

### 在字节码中直接存储值

对于不需要修改的变量，我们可以使用constant关键字将其硬编码为常量。constant变量会直接在字节码中存储，读取该变量不需要SLOAD操作，可以减少读取时的gas花费。

### 无状态合约

Stateless作为一种Dapp的设计模式可以大大的减少gas成本，详情可查看此篇文章[Stateless Smart Contracts](https://medium.com/@childsmaidment/stateless-smart-contracts-21830b0cd1b6)。常用的3种方案就是如下：

- web3请求到主网数据后，读取transaction的input，将数据持久化到中心化后端
- 合约将数据存储到event中，web3从event中读取数据
- 利用ipfs等存储网络存储数据，将源数据的hash上链







> 本文作者：[@7Levy](https://github.com/7Levy)
>
> 参考链接：
>
> 专题(一)
>
> - [How to optimize gas cost in a Solidity smart contract? 6 tips](https://eattheblocks.com/how-to-optimize-gas-cost-in-a-solidity-smart-contract-6-tips/)
> - https://ethereum.stackexchange.com/questions/99971/what-does-enable-optimization-mean-in-remix-and-what-does-it-do/99973
>
> 专题(二)
>
> - https://ethereum.stackexchange.com/questions/16766/any-reason-not-to-use-browser-soliditys-enable-optimization
> - https://eips.ethereum.org/EIPS/eip-1167
> - https://github.com/optionality/clone-factory
> - https://ethereum.stackexchange.com/questions/28813/how-to-write-an-optimized-gas-cost-smart-contract/28848
> - https://arxiv.org/pdf/1703.03994.pdf
>
> 专题(三)
>
> - https://blog.polymath.network/solidity-tips-and-tricks-to-save-gas-and-reduce-bytecode-size-c44580b218e6
> - https://mudit.blog/solidity-gas-optimization-tips/
> - https://ethereum.stackexchange.com/questions/19380/external-vs-public-best-practices
>
> 专题(四)
>
> - https://medium.com/coinmonks/8-ways-of-reducing-the-gas-consumption-of-your-smart-contracts-9a506b339c0a
> - https://mirror.xyz/0xmobius.eth/xAomQ_AVuYsK9V7VotByi320MqfDM4d_pDiCZ4w-Eok





