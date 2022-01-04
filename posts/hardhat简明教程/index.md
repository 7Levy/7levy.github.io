# Hardhat简明教程


## 环境配置

### 安装yarn

```
npm install -g yarn
yarn --version
```

### 安装hardhat

```
yarn add -D hardhat #写入当前目录的devDependencies 
```

### 安装依赖

```
npm install --save-dev @nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers
```

## 创建项目

### 初始化项目

```
yarn hardhat
888    888                      888 888               888
888    888                      888 888               888
888    888                      888 888               888
8888888888  8888b.  888d888 .d88888 88888b.   8888b.  888888
888    888     "88b 888P"  d88" 888 888 "88b     "88b 888
888    888 .d888888 888    888  888 888  888 .d888888 888
888    888 888  888 888    Y88b 888 888  888 888  888 Y88b.
888    888 "Y888888 888     "Y88888 888  888 "Y888888  "Y888

Welcome to Hardhat v2.8.0

? What do you want to do? ...
> Create a basic sample project #示例项目
  Create an advanced sample project
  Create an advanced sample project that uses TypeScript
  Create an empty hardhat.config.js #纯净项目
  Quit
```

### 验证项目

```
yarn hardhat
Hardhat version 2.8.0

Usage: hardhat [GLOBAL OPTIONS] <TASK> [TASK OPTIONS]

GLOBAL OPTIONS:

  --config              A Hardhat config file.
  --emoji               Use emoji in messages.
  --help                Shows this message, or a task's help if its name is provided
  --max-memory          The maximum amount of memory that Hardhat can use.
  --network             The network to connect to.
  --show-stack-traces   Show stack traces.
  --tsconfig            A TypeScript config file.
  --verbose             Enables Hardhat verbose logging
  --version             Shows hardhat's version.


AVAILABLE TASKS:

  accounts      Prints the list of accounts
  check         Check whatever you need
  clean         Clears the cache and deletes all artifacts
  compile       Compiles the entire project, building all artifacts
  console       Opens a hardhat console
  flatten       Flattens and prints contracts and their dependencies
  help          Prints this message
  node          Starts a JSON-RPC server on top of Hardhat Network
  run           Runs a user-defined script after compiling the project
  test          Runs mocha tests

To get help for a specific task run: npx hardhat help [task]
```

### 项目结构

`contracts`:合约源文件

`scripts`:合约部署、交互等自动化脚本

`hardhat.config.js`:hardhat配置文件

`test`:合约测试文件

## 编译项目

### 修改编译器版本

```
module.exports = {
  solidity: "0.8.10",
};
```

### 编译源码

编译`contracts/`下的源文件，如果非初次编译，那么会编译受影响的文件和相关文件。

```
yarn hardhat compile
```

### 清除缓存与编译

```
yarn hardhat clean
```

### artifacts

`artifacts`:存放编译合约后生成的文件

- `contractName`:合约名称
- `abi`:abi的json描述
- `bytecode`:0x前缀的十六进制字节码，如果不可部署，则为"0x"\，运行时字节码和创建字节码的总称
- `depoyedBytecode`:运行时字节码

## 控制台

```
yarn hardhat console
```



## 合约测试

合约测试使用waffle框架，其整合了ether.js、mocha、chai。测试会执行`test/`下的测试脚本

### 测试脚本

```js
const { expect } = require("chai");  //chai断言库
const { ethers } = require("hardhat"); //加载hardhat库

//遵循mocha标准测试结构
describe("Greeter", function () {
  it("Should return the new greeting once it's changed", async function () {
    // 获得自定义地址
    const [owner,addr1] = await ethers.getSigner();
    // Promise异步-获取合约的实例工厂
    const Greeter = await ethers.getContractFactory("Greeter");
    // Promise异步-传入参数初始化合约，获取实例
    const greeter = await Greeter.deploy("Hello, world!");
    // Promise异步-等待部署成功
    await greeter.deployed();
    // 调用greet()方法,通过chai断言
    expect(await greeter.greet()).to.equal("Hello, world!");
    // 调用setGreeting方法
    const setGreetingTx = await greeter.connect(addr1).setGreeting("Hola, levy!");
    // wait until the transaction is mined
    await setGreetingTx.wait();
    // 判断设置的值是否正确
    expect(await greeter.greet()).to.equal("Hola, levy!");
  });
});
```

### Hardhat Network运行测试

```
yarn hardhat test
```

### Ganache测试

安装ganache plugin

```
npm install --save-dev @nomiclabs/hardhat-ganache
```

在hardhat.config.js加入

```
require("@nomiclabs/hardhat-ganache");
```

部署

```
yarn hardhat --network ganache test
```

### 测试网络部署

## 合约部署

### 部署脚本

```js
async function main() {
  // We get the contract to deploy
  const Greeter = await ethers.getContractFactory("Greeter");
  const greeter = await Greeter.deploy("Hello, Hardhat!");

  console.log("Greeter deployed to:", greeter.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

### 本地网络部署

先启用一个本地网络

```
yarn hardhat node
```

另启一个终端，部署在本地网络中

```
yarn hardhat run --network localhost scripts/deploy.js
```

### Ganache部署

安装ganache plugin

```
npm install --save-dev @nomiclabs/hardhat-ganache
```

在hardhat.config.js加入

```
require("@nomiclabs/hardhat-ganache");
```

部署

```
yarn hardhat run --network ganache scripts/deploy.js
```

### 测试网络部署

在[infura](https://infura.io/)注册以太坊网关，用于访问链上网络

在hardhat.config.js中配置网络

```
networks: {
    hardhat: {
    },
    rinkeby: {
      url: "<infura_api>",
      accounts: [privateKey1, privateKey2, ...]
    }
  },
```

部署

```
yarn hardhat run --network <your-network> scripts/deploy.js
```





