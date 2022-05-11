# poa consensus

## 创建节点
```shell
make all #编译之后出现文件夹bin
cd build/bin
```
在bin文件夹下创立三个文件夹node0、node1、node2
给三个节点创建账号密码
```shell
geth --datadir node0 account new
```
创建了三个账号：

```shell
Public address of the key:   0x335572b5fa4D1f8Bc6310002E35a1187A676cEA9
Path of the secret key file: node0/keystore/UTC--2022-05-11T12-51-02.847378000Z--335572b5fa4d1f8bc6310002e35a1187a676cea9
```

```shell
Public address of the key:   0x175d78f97FB658906a589ecDbf13a2d0B6Bbd5f6
Path of the secret key file: node2/keystore/UTC--2022-05-11T12-52-21.711665000Z--175d78f97fb658906a589ecdbf13a2d0b6bbd5f6
```

```shell
Public address of the key:   0x27f69FEB9FaF2d4c76e4107A65F8271F39AC2004
Path of the secret key file: node1/keystore/UTC--2022-05-11T12-53-14.224955000Z--27f69feb9faf2d4c76e4107a65f8271f39ac2004
```

## puppeth基础配置

puppeth是私人网络管理的交互式工具 他允许你根据传入的创世区块信息、启动节点、矿工、共识引擎来创建一个私人网络

配置比较简单 按照指示来就可以

1. 选择configure new genesis 输入一系列配置
2. 
3. Manage existing genesis 然后导出配置文件

不过其中有个设置栈是：Which accounts are allowed to seal? (mandatory at least one)

seal是什么意思？ ==》 原来seal就是在测试网上挖矿

生成的配置文件放在了build/bin/foooox.json

可以看到他给一些默认地址转了1wei eth 而且给你之前填入需要prefund的地址转了0x200000000000000000000000000000000000000000000000000000000000000 wei

## 初始化节点
```shell
geth --datadir node0 init build/bin/foooox.json 
```

这个命令是通过传入的初始化的json配置文件来初始化节点并写入genesis state

## 启动节点

在三个终端下启动节点

```shell
cd build/bin/
```

```shell
geth --datadir node0 --port 30000 --nodiscover --unlock '0' --password "node0/keystore/UTC--2022-05-11T12-51-02.847378000Z--335572b5fa4d1f8bc6310002e35a1187a676cea9"
```

```shell
geth --datadir node1 --port 30001 --nodiscover --unlock '0' --password "node1/keystore/UTC--2022-05-11T12-53-14.224955000Z--27f69feb9faf2d4c76e4107a65f8271f39ac2004"
```

```shell
geth --datadir node2 --port 30002 --nodiscover --unlock '0' --password "node2/keystore/UTC--2022-05-11T12-52-21.711665000Z--175d78f97fb658906a589ecdbf13a2d0b6bbd5f6"
```
成功之后会出现：
```go
INFO [05-11|21:53:55.914] Started P2P networking                   self="enode://1e034ad860b0c9aa779c9c34fcf42b9bea72338b8909919b7a39ec64e0240ea127f04b195dc53b43730091d39495d53177f47618615ed21df9a41ab54d7c6d9c@127.0.0.1:30000?discport=0"
WARN [05-11|21:53:55.914] ------------------------------------------------------------------- 
WARN [05-11|21:53:55.914] Referring to accounts by order in the keystore folder is dangerous! 
WARN [05-11|21:53:55.914] This functionality is deprecated and will be removed in the future! 
WARN [05-11|21:53:55.914] Please use explicit addresses! (can search via `geth account list`) 
WARN [05-11|21:53:55.914] ------------------------------------------------------------------- 
Fatal: Failed to unlock account 0 (could not decrypt key with given password)
```

这里的unlock是什么意思？

不知道要怎么解锁 后面再看看/*todo*/

默认情况下，geth中的账号都是locked的，也就是你没有办法使用这些账号进行发送交易。

你需要使用geth rpc来解锁account（web3的插件不支持这个操作

```shell
Started P2P networking                   self="enode://26ff926ee12979eccf42e0fc1f2987078a1a32674da55abe4c59a7180ace4e1822b8292ab17351617ddd77f1de6395f54342b0ae2edba023a9e3c564ff05b303@127.0.0.1:30002?discport=0"
Fatal: Failed to unlock account 0 (could not decrypt key with given password)
```


## 添加节点

admin.addPeer("enode://1e034ad860b0c9aa779c9c34fcf42b9bea72338b8909919b7a39ec64e0240ea127f04b195dc53b43730091d39495d53177f47618615ed21df9a41ab54d7c6d9c@127.0.0.1:30000?discport=0")
