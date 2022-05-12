# build a poa devnet

## 创建账户
创建一个devnet文件夹 在其中添加两个文件夹 node1 node2

创建账户：
geth --datadir node1/ account new node
这样在node1文件夹下就生成了一个keystore文件夹

devnet/node1/keystore/UTC--2022-05-12T09-06-16.717051000Z--42d1cb266254d66f4dcfdabc37a272ef214ff995

里面的txt文件的名字最后包含了你账户的地址 
文件里是你账户的信息（有些不知道是啥

我们把账户的地址echo到account文件中 以免后面复制粘贴

echo '42d1cb266254d66f4dcfdabc37a272ef214ff995' >> accounts.txt
echo '3d632d1d48059fb0a9695eca1a4666b0f1c85845' >> accounts.txt

同理也echo一下密码

## 创世区块配置

利用puppeth工具来创建
详细不写了

## 初始化节点
geth --datadir node1/ init foooox2.json
geth --datadir node2/ init foooox2.json

## p2p bootnode

创建bootnode：
bootnode -genkey boot.key

bootnode是用来帮值节点之间更好地用p2p进行联系和互相发现

你在运行以太坊节点的时候是可以用动态ip的 但是我见到的一些新项目（celestia和ceramic在运行节点的时候都要求你拥有共网ip）

bootnode是一定要有公网静态ip的 他不会同步链上信息 但是你可以向他查询有用的peer node

以太坊官方维护了一些bootnodes 这些节点的端口被写在geth的源码中: go-ethereum/params/bootnodes.go

你可以通过命令来向节点列表中增加bootnode节点

## 启动bootnode

bootnode -nodekey boot.key -verbosity 9 -addr :30310

对node1
geth --datadir node1/ --syncmode 'full' --port 30311 --rpc --rpcaddr 'localhost' --rpcport 8501 --rpcapi 'personal,db,eth,net,web3,txpool,miner' --bootnodes 'enode://e53780928a40317b88f8432b78d692a45079f21b19edc2e4322d1751df37baf6f9705b5c66e3e7c5f7e722f33a8dcde7a944ad15c7af3b58daed1ad238f58dc7@127.0.0.1:0?discport=30310' --networkid 1161 --gasprice '1' -unlock '0x42d1cb266254d66f4dcfdabc37a272ef214ff995' --password node1/password.txt --mine

networkid可以在devnet/foooox2.json里找到
unlock是你要解锁的地址