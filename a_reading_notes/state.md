## stateobject

stateobject 对应着一个以太坊账户 
交易是通过改变staeobject的内容来实现的

以太坊中有两种账户：
1. 外部账户（EOA)
2. 合约账户（Contract） 部署智能合约时创建的合约地址
这两种账户都是由同样的stateaccount类型来维护的，但其中的参数有不同


交易的流程：
1. 获取账户的stateobject
2. 对stateobject存储的值进行读、写等操作
3. 调用CommitTrie把你修改过的状态树存储到数据库中

stateobject的定义：
```go
type stateObject struct {
	address  common.Address地址
	addrHash common.Hash // hash of ethereum address of the account 地址的hash值
	data     types.StateAccount 
	db       *StateDB

	// DB error.
	// State objects are used by the consensus core and VM which are
	// unable to deal with database-level errors. Any error that occurs
	// during a database read is memoized here and will eventually be returned
	// by StateDB.Commit.
	dbErr error

	// Write caches.
	trie Trie // storage trie, which becomes non-nil on first access
	code Code // contract bytecode, which gets set when code is loaded

	originStorage  Storage // Storage cache of original entries to dedup rewrites, reset for every transaction
	pendingStorage Storage // Storage entries that need to be flushed to disk, at the end of an entire block
	dirtyStorage   Storage // Storage entries that have been modified in the current transaction execution
	fakeStorage    Storage // Fake storage which constructed by caller for debugging purpose.

	// Cache flags.
	// When an object is marked suicided it will be delete from the trie
	// during the "update" phase of the state transition.
	dirtyCode bool // true if the code was updated
	suicided  bool
	deleted   bool
}
```

其中最重要的是stateaccount，它被存储在main account trie中，包含了一个账户的最基本信息）里面有以下信息：
1. 随机数
   表示该账户发送的交易序号，随着账户发送的交易数量的增加而单调增加。每次发送一个交易，Nonce的值就会加1
2. 余额 
   这里的余额指的是链上的Global/Native Token Ether
3. 在storage trie中的默克尔根
   Root 表示当前账户的下Storage层的 Merkle Patricia Trie的Root。只有合约账户才会有这个root，外部账户为空
4. codehash
   codehash是合约账户才会有的（部署合约时的合约code 一旦部署无法修改）。

db是存储状态的数据库，支持对账户信息进行读、写、改等操作
trie是可以修改的状态树，是合约账户调用函数的时候使用的，合约中永久改变的值会被记录在这里（比如nft合约的ipfs地址啥的），包含了以下四种storage：
1. originStorage （原始的storage 每次交易前都会reset）
2. pendingStorage（处在区块链最末端的storage 等待被加入磁盘）
3. dirtyStorage（dirty storage是用来存储被改变了的状态 上传的时候只要上传dirty storage）
4. fakeStorage（在debugging的时候使用）
   如果是debug形式的，加载的东西会被放到fakestorage中，修改后的state会被临时更新到temporary state。

code是你在调用合约时使用的code，会在合约加载的时候被存放到内存中，用来调用。

合约账户执行了特定的行为后，在内存中这个stateobject会被带上不同的flag
1. dirtyCode 如果合约账户处在更新状态中为true （账户的codehash应该是不可改变的，但是以太坊官方出了一个更新合约的方法，本质应该还是新建了一个合约但是把原来的数据转移过去了，后面可以详细研究一下/todo/
2. suicided 
3. deleted

（所以这里的trie，code，storage，cache flag都是给合约账户的

## account state的流程（四种storage的调用流程）

1. newObject 
   传入地址 返回一个object对象 顺便如果里面有值是nil把他填成空的对象

2. touch 
   传入newObject生成的*stateObject，调用s.db.journal.append将这个地址写入数据库的journal中，也就是加入cache缓存。
   journal就像日志，里面会记录你对state的修改（快照就是基于他生成的
   /todo/ ripemd是另一种加密方式
   如果这个地址是ripemd（("0000000000000000000000000000000000000003")，把他写入dirty-cache，This method is an ugly hack to handle the RIPEMD。

3. getTrie
   传入*stateObject中的db， 调用db.OpenStorageTrie(s.addrHash, s.data.Root)生成storage trie
   首先尝试从prefetcher中拿到storage trie（prefetcher是存储在statedb中的，为了方便获取trie。如果失败，再通过地址的hash从db中获取，创建storage trie。

4. GetState
   拿到storage trie后从中获取state（也就是account storage trie的键值对）
   你输入key的hash之后会返回对应的value   
   获取分为三层：
   1. 如果storage trie中设置了fake storage，直接返回fake storage中的值（fake是在debug的时候生成的
   2. 如果有dirty storage，返回dirty中的值
   3. 否则调用GetCommittedState，返回original value（没有改变状态）
   
   GetCommittedState是从已经提交到以太坊上的storage trie中获取值，获取也分三层：
   1. 先尝试从fake storage中获取
   2. 尝试从pending storage和cache中获取（pending是已经提交了但是还未写入磁盘的状态）
   3. 如果上面的途径都失败了，从快照中获取
   4. 如果快照失败了（轻节点不保存完整快照，只保存最近的快照），直接从数据库中获取
   
5. SetState
   更新account storage的值
   

6. finalise
   发送每一笔交易最终都会调用finalize
   它把dirty storage内的slots移到pending area，在pending area进行hash和commit
   
7. updateTrie
   1. 调用finalise（发送每一笔交易最终都会调用finalize）
   finalise把dirty storage内的slots移到pending area。
   在pending area会进行hash和commit。
   1. 创建一个usedStorage的map对象 usedStorage := make([][]byte, 0, len(s.pendingStorage))，遍历pending storage内的所有键值对（跳过和原来相比没有发生改变的），在对pending storage内的值进行rlp encode编码之后，写入originStorage。
8. 如果有state 快照的话，把更改后的storage存到缓存

