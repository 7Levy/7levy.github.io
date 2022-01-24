# Master Solidity-极限优化GAS


> 本文作者：[@7Levy](https://github.com/7Levy)
>
> 参考链接：
>
> - [How to optimize gas cost in a Solidity smart contract? 6 tips](https://eattheblocks.com/how-to-optimize-gas-cost-in-a-solidity-smart-contract-6-tips/)
> - https://ethereum.stackexchange.com/questions/99971/what-does-enable-optimization-mean-in-remix-and-what-does-it-do/99973
> - https://ethereum.stackexchange.com/questions/16766/any-reason-not-to-use-browser-soliditys-enable-optimization
> - https://eips.ethereum.org/EIPS/eip-1167
> - https://github.com/optionality/clone-factory
> - https://ethereum.stackexchange.com/questions/28813/how-to-write-an-optimized-gas-cost-smart-contract/28848
> - https://blog.polymath.network/solidity-tips-and-tricks-to-save-gas-and-reduce-bytecode-size-c44580b218e6


## 优化GAS

### 最小化链上数据

对一个Dapp重要的数据必须放在链上，将非关键部分的代码放在链下。

### Solidity optimization

在我们编译solidity智能合约的时候，我们需要指定一个参数来告诉solidity编译器生成优化的字节码。如果你不使用优化参数的话，那么就会花费更多的gas。

在实际的开发中，这种优化方式会更长的编译时间，默认情况下是关闭的。以Hardhat为例，我们可以在配置文件hardhat.config.js中启用，并设置你所期望的运行次数，runs指定了部署代码的每个操作码在合约生命周期内执行的频率。runs较小时(如1)，初始字节码较短，部署成本低，但会导致后期合约执行成本高。runs较大时(如1000)，初始字节码较长且复杂，部署成本更高，但对合约的调用会更便宜。对于这个参数，我们需要针对所写的合约来设置，如果合约只是一次性的代码，设置成1即可。对于ERC20代币之类的合约，设置较大的数字能显著地减少调用成本。

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

solidity数据存储在32字节的slots中，采用紧密打包的模式存放数据。因此我们可以通过内存对齐的方式，来减少变量占用的插槽数。如下面的例子。

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

### 短路规则

以f(x)||g(x)为例，如果f(x)为true，那么就不会执行g(x)函数。因此，在涉及这类操作时，我们可以把成本更高的操作码放在后面。当f(x)成本大于g(x)，我们可以按以下规则编码。

以f(x)||g(x)为例，如果f(x)为true，那么就不会执行g(x)函数。因此，在涉及这类操作时，我们可以把成本更高的操作码放在后面。当f(x)成本大于g(x)，我们可以按以下规则编码。

- OR：f(x) || g(x)
- AND：f(x) && g(x)

在成本接近的时候，我们还可以根据返回值的几率来设置短路规则。如果g(x)返回true的概率大于f(x)，OR操作需要把g(x)放在前面，如果g(x)返回false 的概率大于f(x)，AND操作需要把g(x)放在前面。

### 循环中使用局部变量

```
 uint a = 0;
 function test ( uint x ){
     for ( uint i = 0 ; i < x ; i++)
         sum += i; 
 }
```



```
 uint sum = 0;
 function p3 ( uint x ){
     uint temp = 0;
     for ( uint i = 0 ; i < x ; i++)
         temp += i; }
     sum += temp;
```







