# Geth POA联盟链搭建


## 搭建以太坊联盟链

### 创建节点目录

```
[root@google01 private-POA-chian]# mkdir node-1 node-2 node-3
[root@google01 private-POA-chian]# ls
node-1  node-2  node-3
```

### 新建账户

```
[root@google01 private-POA-chian]# geth --datadir ./node-1/data account new
[root@google01 private-POA-chian]# geth --datadir ./node-2/data account new
[root@google01 private-POA-chian]# geth --datadir ./node-3/data account new
```

创建的地址

node-1：88b34c24E8F727b44C17065A26B76Bc31CF2c029

node-2：241f173D632C57773B552953Fe1f2821d24BE072

node-3：Aae8AD28560E70f81815ff5B5AD9d65dC4E7Ca22

### puppeth初始化区块

**输入链名称**

```
[root@google01 private-POA-chian]# puppeth
+-----------------------------------------------------------+
| Welcome to puppeth, your Ethereum private network manager |
|                                                           |
| This tool lets you create a new Ethereum network down to  |
| the genesis block, bootnodes, miners and ethstats servers |
| without the hassle that it would normally entail.         |
|                                                           |
| Puppeth uses SSH to dial in to remote servers, and builds |
| its network components out of Docker containers using the |
| docker-compose toolset.                                   |
+-----------------------------------------------------------+

Please specify a network name to administer (no spaces, hyphens or capital letters please)
> test

Sweet, you can set this via --network=test next time!

INFO [11-26|14:35:42.756] Administering Ethereum network           name=test
WARN [11-26|14:35:42.756] No previous configurations found         path=/root/.puppeth/test
```

**新建区块->选择POA算法->出块时间10秒**
```
What would you like to do? (default = stats)
 1. Show network stats
 2. Configure new genesis
 3. Track new remote server
 4. Deploy network components
> 2

What would you like to do? (default = create)
 1. Create new genesis from scratch
 2. Import already existing genesis
> 1

Which consensus engine to use? (default = clique)
 1. Ethash - proof-of-work
 2. Clique - proof-of-authority
> 2

How many seconds should blocks take? (default = 15)
> 10
```

**选择密封块账户**

```
Which accounts are allowed to seal? (mandatory at least one)
> 0x88b34c24E8F727b44C17065A26B76Bc31CF2c029
> 0x241f173D632C57773B552953Fe1f2821d24BE072
> 0xAae8AD28560E70f81815ff5B5AD9d65dC4E7Ca22
```

> 至于什么是seal a block可以参考stackexchange的回答https://ethereum.stackexchange.com/questions/6093/what-does-it-mean-to-seal-a-block

**选择区块奖励账户**

```
Which accounts should be pre-funded? (advisable at least one)
> 0x88b34c24E8F727b44C17065A26B76Bc31CF2c029
> 0x241f173D632C57773B552953Fe1f2821d24BE072
> 0xAae8AD28560E70f81815ff5B5AD9d65dC4E7Ca22
```

**设置预编译地址奖励及链ID**

```
Should the precompile-addresses (0x1 .. 0xff) be pre-funded with 1 wei? (advisable yes)
> yes
Specify your chain/network ID if you want an explicit one (default = random)
> 15
```

> 预编译地址的用处https://ethereum.stackexchange.com/questions/68056/puppeth-precompile-addresses
>
> "When running sha256, ripemd160 or ecrecover on a private blockchain, you might encounter Out-of-Gas. This is because these functions are implemented as “precompiled contracts” and only really exist after they receive the first message (although their contract code is hardcoded). Messages to non-existing contracts are more expensive and thus the execution might run into an Out-of-Gas error. A workaround for this problem is to first send Wei (1 for example) to each of the contracts before you use them in your actual contracts. This is not an issue on the main or test net."

**导出配置**

```
What would you like to do? (default = stats)
 1. Show network stats
 2. Manage existing genesis
 3. Track new remote server
 4. Deploy network components
> 2

 1. Modify existing configurations
 2. Export genesis configurations
 3. Remove genesis configuration
> 2

Which folder to save the genesis specs into? (default = current)
  Will create test.json, test-aleth.json, test-harmony.json, test-parity.json
> 
INFO [11-26|14:59:18.889] Saved native genesis chain spec          path=test.json
ERROR[11-26|14:59:18.889] Failed to create Aleth chain spec        err="unsupported consensus engine"
ERROR[11-26|14:59:18.889] Failed to create Parity chain spec       err="unsupported consensus engine"
INFO [11-26|14:59:18.890] Saved genesis chain spec                 client=harmony path=test-harmony.json
```

### 私链搭建

**加载配置**

```
geth --datadir node-1/data init test.json
geth --datadir node-2/data init test.json
geth --datadir node-3/data init test.json
```

**启动区块链**

```
geth --datadir ./node-1/data/ --port 1185
geth --datadir ./node-2/data/ --port 1186
geth --datadir ./node-3/data/ --port 1187

[root@google01 private-POA-chian]# screen -ls
There are screens on:
	4439.node-3	(Detached)
	4397.node-2	(Detached)
	4298.node-1	(Detached)
```

**节点建立通信**

- 登录node-1,获取节点信息

```
[root@google01 private-POA-chian]# geth attach ipc:node-1/data/geth.ipc
Welcome to the Geth JavaScript console!

instance: Geth/v1.10.12-stable-6c4dc6c3/linux-amd64/go1.17.2
coinbase: 0x88b34c24e8f727b44c17065a26b76bc31cf2c029
at block: 0 (Fri Nov 26 2021 14:48:33 GMT+0800 (CST))
 datadir: /home/fqc/projects/blockchain/ethereum/private-POA-chian/node-1/data
 modules: admin:1.0 clique:1.0 debug:1.0 eth:1.0 miner:1.0 net:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

To exit, press ctrl-d or type exit
> 
> admin.nodeInfo.enode
"enode://567c2b4de752396bab3fee6d2f31fb705c1a3b40f929cc4852961b5ddd15ad47c580ee2cb35a596351e125c8210c345aee19b104e5b66596c108c24ac9cb3329@115.29.190.53:1185"
```

- 登录node-2、node-3建立peer通信

```
admin.addPeer("enode://567c2b4de752396bab3fee6d2f31fb705c1a3b40f929cc4852961b5ddd15ad47c580ee2cb35a596351e125c8210c345aee19b104e5b66596c108c24ac9cb3329@127.0.0.1:1185")
```

- 登录node-1确定是否成功

```
> net.peerCount
> admin.peers
```

**测试**

- 登录node-1，查询账户余额

```
> web3.fromWei(eth.getBalance(eth.accounts[0]),'ether')
9.04625697166532776746648320380374280103671755200316906558262375061821325312e+56
```

- 转账交易

```
> personal.unlockAccount(eth.accounts[0])
Unlock account 0x88b34c24e8f727b44c17065a26b76bc31cf2c029
Passphrase: 
true
> eth.sendTransaction({from:eth.accounts[0], to:"241f173D632C57773B552953Fe1f2821d24BE072", value:20000})
"0x8216e70d056ff0e02701d75b0059489eb7b293128b8a19ae393590b5b05a7281"
```

- 查询交易。可以看到区块还未打包，因此需要开启挖矿节点

```
> eth.getTransaction("0xa86e66aa6b9b35672f0f9d235d258b681a13e03b232a97c6e994db7167f8f1e6")
{
  blockHash: null,
  blockNumber: null,
  from: "0x88b34c24e8f727b44c17065a26b76bc31cf2c029",
  gas: 21000,
  gasPrice: 1000000000,
  hash: "0xa86e66aa6b9b35672f0f9d235d258b681a13e03b232a97c6e994db7167f8f1e6",
  input: "0x",
  nonce: 4,
  r: "0x46ed1d6d2c8f28e5db451d061303051aca58470a1ed4126e95e5af58345d1bc7",
  s: "0x5a722e13a7524a8e0ca4ae4c75b3619575da921ece96bde2702c7cb720117383",
  to: "0x241f173d632c57773b552953fe1f2821d24be072",
  transactionIndex: null,
  type: "0x0",
  v: "0x41",
  value: 1e+66
}

> txpool.status #查看交易状态
{
  pending: 6,
  queued: 0
}
```

- 启动挖矿

```
miner.start()
eth.hashrate #查看挖矿状态
```


