# Entrance for Geth

## 先找到项目入口cmd/geth/main.go
``` go
// geth的入口函数
func main() {
	if err := app.Run(os.Args); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```
app.Run(os.Args)启动整个项目，os.Args是拿到你命令行输入的参数，如果报错，把错误输出到Stderr

## 准备函数
在具体看app.Run函数之前，先看一下启动前的两个准备函数：
```go
func init() {
	// Initialize the CLI app and start Geth
	app.Action = geth
	app.HideVersion = true // we have a command to print the version
	app.Copyright = "Copyright 2013-2022 The go-ethereum Authors"
	app.Commands = []cli.Command{
		// See chaincmd.go:
		initCommand,
		importCommand,
		exportCommand,
		importPreimagesCommand,
		exportPreimagesCommand,
		removedbCommand,
		dumpCommand,
		dumpGenesisCommand,
		// See accountcmd.go:
		accountCommand,
		walletCommand,
		// See consolecmd.go:
		consoleCommand,
		attachCommand,
		javascriptCommand,
		// See misccmd.go:
		makecacheCommand,
		makedagCommand,
		versionCommand,
		versionCheckCommand,
		licenseCommand,
		// See config.go
		dumpConfigCommand,
		// see dbcmd.go
		dbCommand,
		// See cmd/utils/flags_legacy.go
		utils.ShowDeprecated,
		// See snapshot.go
		snapshotCommand,
	}
	sort.Sort(cli.CommandsByName(app.Commands))

	app.Flags = append(app.Flags, nodeFlags...)
	app.Flags = append(app.Flags, rpcFlags...)
	app.Flags = append(app.Flags, consoleFlags...)
	app.Flags = append(app.Flags, debug.Flags...)
	app.Flags = append(app.Flags, metricsFlags...)

	app.Before = func(ctx *cli.Context) error {
		return debug.Setup(ctx)
	}
	app.After = func(ctx *cli.Context) error {
		debug.Exit()
		prompt.Stdin.Close() // Resets terminal mode.
		return nil
	}
}
```

1. func init() 会在main函数调用之前执行，主要内容包含
   1. app.Action = geth，启动geth
   2. 导入了一堆command 例如 （具体定义在chaincmd.go==》chaincmd里面是一堆你通过命令行方式传递的配置信息
   3. initcommand 初始化命令。接收通过flag形式传递的参数作为你节点的配置信息（比如lightmode archive fast 测试网等flag设置）。接收传入genesis file （可以指定网络是主网还是测试网
   4. dumpgenesis 是把创世区块的配置信息以json格式输出到stdout标准输出文件中（估计是生成一个配置文件到你的geth目录里 config.json之类的东西，里面是你设置的flag参数（更方便
   5. importcommand应该是方便你导入以前已经同步好了的数据
   6. exportcommand是导出区块数据到文件中
   7. initgenesis从你指定的genesis path中读取到关于genesis配置的json文件（genesis的参数设置在params/config.go里，里面有chainconfig可以指定你选择哪一个分支（比如hometead daofork) ==> 同步这种废弃的分支来做啥？==》一般是用来测试
   
```go
// prepare manipulates memory cache allowance and setups metric system.
// This function should be called before launching devp2p stack.
func prepare(ctx *cli.Context) {
	// If we're running a known preset, log it for convenience.
	switch {
	case ctx.GlobalIsSet(utils.RopstenFlag.Name):
		log.Info("Starting Geth on Ropsten testnet...")

	case ctx.GlobalIsSet(utils.SepoliaFlag.Name):
		log.Info("Starting Geth on Sepolia testnet...")

	case ctx.GlobalIsSet(utils.RinkebyFlag.Name):
		log.Info("Starting Geth on Rinkeby testnet...")

	case ctx.GlobalIsSet(utils.GoerliFlag.Name):
		log.Info("Starting Geth on Görli testnet...")

	case ctx.GlobalIsSet(utils.DeveloperFlag.Name):
		log.Info("Starting Geth in ephemeral dev mode...")

	case !ctx.GlobalIsSet(utils.NetworkIdFlag.Name):
		log.Info("Starting Geth on Ethereum mainnet...")
	}
	// If we're a full node on mainnet without --cache specified, bump default cache allowance
	if ctx.GlobalString(utils.SyncModeFlag.Name) != "light" && !ctx.GlobalIsSet(utils.CacheFlag.Name) && !ctx.GlobalIsSet(utils.NetworkIdFlag.Name) {
		// Make sure we're not on any supported preconfigured testnet either
		if !ctx.GlobalIsSet(utils.RopstenFlag.Name) &&
			!ctx.GlobalIsSet(utils.SepoliaFlag.Name) &&
			!ctx.GlobalIsSet(utils.RinkebyFlag.Name) &&
			!ctx.GlobalIsSet(utils.GoerliFlag.Name) &&
			!ctx.GlobalIsSet(utils.DeveloperFlag.Name) {
			// Nope, we're really on mainnet. Bump that cache up!
			log.Info("Bumping default cache on mainnet", "provided", ctx.GlobalInt(utils.CacheFlag.Name), "updated", 4096)
			ctx.GlobalSet(utils.CacheFlag.Name, strconv.Itoa(4096))
		}
	}
	// If we're running a light client on any network, drop the cache to some meaningfully low amount
	if ctx.GlobalString(utils.SyncModeFlag.Name) == "light" && !ctx.GlobalIsSet(utils.CacheFlag.Name) {
		log.Info("Dropping default light client cache", "provided", ctx.GlobalInt(utils.CacheFlag.Name), "updated", 128)
		ctx.GlobalSet(utils.CacheFlag.Name, strconv.Itoa(128))
	}

	// Start metrics export if enabled
	utils.SetupMetrics(ctx)

	// Start system runtime metrics collection
	go metrics.CollectProcessMetrics(3 * time.Second)
}
```

2. prepare() 会在p2p网络启动前执行
   1. log一下你选择的network（是哪个测试网or主网
   2. 如果你是测试网，是不需要共识机制来进行同步数据和挖矿的，所以也不需要生成cache这些文件。如果你选择运行轻节点（light node），cache是定期删除来保证你节点是轻量的
   3. 启动计时器

## 启动节点函数geth

```go
func geth(ctx *cli.Context) error {
	if args := ctx.Args(); len(args) > 0 {
		return fmt.Errorf("invalid command: %q", args[0])
	}//geth本身无法传入指定参数

	prepare(ctx)
	stack, backend := makeFullNode(ctx)
	defer stack.Close()

	startNode(ctx, stack, backend, false)
	stack.Wait()
	return nil
}
```
*cli.Context包含你传入的app（就是geth），command，flag等

调用prepare准备函数
根据传入的配置信息启动节点
makeFullNode(ctx) 函数加载配置开启以太坊节点后端和api后端(节点后端作为stack 是用来执行交易的)
```go
func makeFullNode(ctx *cli.Context) (*node.Node, ethapi.Backend) {
	stack, cfg := makeConfigNode(ctx)
	if ctx.GlobalIsSet(utils.OverrideArrowGlacierFlag.Name) {
		cfg.Eth.OverrideArrowGlacier = new(big.Int).SetUint64(ctx.GlobalUint64(utils.OverrideArrowGlacierFlag.Name))
	}
	if ctx.GlobalIsSet(utils.OverrideTerminalTotalDifficulty.Name) {
		cfg.Eth.OverrideTerminalTotalDifficulty = utils.GlobalBig(ctx, utils.OverrideTerminalTotalDifficulty.Name)
	}
	backend, eth := utils.RegisterEthService(stack, &cfg.Eth)
	// Warn users to migrate if they have a legacy freezer format.
	if eth != nil {
		firstIdx := uint64(0)
		// Hack to speed up check for mainnet because we know
		// the first non-empty block.
		ghash := rawdb.ReadCanonicalHash(eth.ChainDb(), 0)
		if cfg.Eth.NetworkId == 1 && ghash == params.MainnetGenesisHash {
			firstIdx = 46147
		}
		isLegacy, _, err := dbHasLegacyReceipts(eth.ChainDb(), firstIdx)
		if err != nil {
			log.Error("Failed to check db for legacy receipts", "err", err)
		} else if isLegacy {
			log.Warn("Database has receipts with a legacy format. Please run `geth db freezer-migrate`.")
		}
	}

	// Configure GraphQL if requested
	if ctx.GlobalIsSet(utils.GraphQLEnabledFlag.Name) {
		utils.RegisterGraphQLService(stack, backend, cfg.Node)
	}
	// Add the Ethereum Stats daemon if requested.
	if cfg.Ethstats.URL != "" {
		utils.RegisterEthStatsService(stack, backend, cfg.Ethstats.URL)
	}
	return stack, backend
}
```
一系列Override函数会把原有的设置都清空
backend, eth := utils.RegisterEthService(stack, &cfg.Eth) 
registerethservice函数会返回ethapi.Backend作为api后端和 *eth.Ethereum全节点实例

全节点和轻节点的设置会有不同：
1. *eth.Ethereum全节点实例
如果你选择了运行轻节点，会调用les.new，返回的*eth.Ethereum就会为nil。如果是全节点，调用的是new，正常返回*eth.Ethereum全节点实例

2. stack.RegisterAPIs(tracers.APIs(backend.ApiBackend))注册api后端

仔细看下makefullnode中返回的两个东西的定义：

## stack
stack的类型是*node.Node geth最重要的几个功能（账户管理 p2p rpc http等外部端口都在这

```go
type Node struct {
	eventmux      *event.TypeMux//里面是一些event的定义（例如w（等待），writerSem， readerSem）这些事件都是单线程的 如果一个readersem在执行 其他的程序就要等read程序执行完毕 开放线程锁之后才调用）
	config        *Config
	accman        *accounts.Manager//accountmanager是管理账户 让账户可以和多个backend交互 签名交易==>这个后面要仔细看下/todo/
	log           log.Logger
	keyDir        string            // key store directory存储死要的文件？
	keyDirTemp    bool              // If true, key directory will be removed by Stop
	dirLock       fileutil.Releaser // prevents concurrent use of instance directory
	stop          chan struct{}     // Channel to wait for termination notifications
	server        *p2p.Server       // Currently running P2P networking layer
	startStopLock sync.Mutex        // Start/Stop are protected by an additional lock
	state         int               // Tracks state of node lifecycle

	lock          sync.Mutex
    // lifecycles 维护节点运行所需要的后端的实例和服务（具体怎么实现？
	lifecycles    []Lifecycle // All registered backends, services, and auxiliary services that have a lifecycle
	rpcAPIs       []rpc.API   // List of APIs currently provided by the node
	http          *httpServer //
	ws            *httpServer //
	httpAuth      *httpServer //
	wsAuth        *httpServer //
	ipc           *ipcServer  // Stores information about the ipc http server
	inprocHandler *rpc.Server // In-process RPC request handler to process the API requests

	databases map[*closeTrackingDB]struct{} // All open databases
}
```

## backend
geth的后端
其实里面本质就是对整个以太坊数据库的操作函数
里面除了定义了gas上限 timeout阈值等基本设置外 还提供了三种api

Blockchain API
Transaction pool API
Filter API

```go
type Backend interface {
	// General Ethereum API
    // sync process是同步的时候查看进度的函数 里面有起始区块、当前区块、最高区块、下载的状态个数、账户数等等
	SyncProgress() ethereum.SyncProgress
	SuggestGasTipCap(ctx context.Context) (*big.Int, error)
	FeeHistory(ctx context.Context, blockCount int, lastBlock rpc.BlockNumber, rewardPercentiles []float64) (*big.Int, [][]*big.Int, []*big.Int, []float64, error)
    // 数据库 database是一个interface，里面有高级数据库操作的所有函数（比如读写，批量操作，迭代等
	ChainDb() ethdb.Database
	AccountManager() *accounts.Manager
	ExtRPCEnabled() bool
    // gas上限（防止dos攻击
	RPCGasCap() uint64            // global gas cap for eth_call over rpc: DoS protection
    // timeout阈值
	RPCEVMTimeout() time.Duration // global timeout for eth_call over rpc: DoS protection
    // 小费上限
	RPCTxFeeCap() float64         // global tx fee cap for all transaction related APIs
	UnprotectedAllowed() bool     // allows only for EIP155 transactions.

	// Blockchain API
    // 这些是跟区块操作有关的api 例如：设置区块头 列出区块头 列出当前区块/hash等等
	SetHead(number uint64)
	HeaderByNumber(ctx context.Context, number rpc.BlockNumber) (*types.Header, error)
	HeaderByHash(ctx context.Context, hash common.Hash) (*types.Header, error)
	HeaderByNumberOrHash(ctx context.Context, blockNrOrHash rpc.BlockNumberOrHash) (*types.Header, error)
	CurrentHeader() *types.Header
	CurrentBlock() *types.Block
	BlockByNumber(ctx context.Context, number rpc.BlockNumber) (*types.Block, error)
	BlockByHash(ctx context.Context, hash common.Hash) (*types.Block, error)
	BlockByNumberOrHash(ctx context.Context, blockNrOrHash rpc.BlockNumberOrHash) (*types.Block, error)
	StateAndHeaderByNumber(ctx context.Context, number rpc.BlockNumber) (*state.StateDB, *types.Header, error)
	StateAndHeaderByNumberOrHash(ctx context.Context, blockNrOrHash rpc.BlockNumberOrHash) (*state.StateDB, *types.Header, error)
	GetReceipts(ctx context.Context, hash common.Hash) (types.Receipts, error)
	GetTd(ctx context.Context, hash common.Hash) *big.Int
	GetEVM(ctx context.Context, msg core.Message, state *state.StateDB, header *types.Header, vmConfig *vm.Config) (*vm.EVM, func() error, error)
	SubscribeChainEvent(ch chan<- core.ChainEvent) event.Subscription
	SubscribeChainHeadEvent(ch chan<- core.ChainHeadEvent) event.Subscription
	SubscribeChainSideEvent(ch chan<- core.ChainSideEvent) event.Subscription

	// Transaction pool API
    // 和交易相关的api（例如发送交易 获取某一笔交易 获取交易池中的交易 获取交易状态等
	SendTx(ctx context.Context, signedTx *types.Transaction) error
	GetTransaction(ctx context.Context, txHash common.Hash) (*types.Transaction, common.Hash, uint64, uint64, error)
	GetPoolTransactions() (types.Transactions, error)
	GetPoolTransaction(txHash common.Hash) *types.Transaction
	GetPoolNonce(ctx context.Context, addr common.Address) (uint64, error)
	Stats() (pending int, queued int)
	TxPoolContent() (map[common.Address]types.Transactions, map[common.Address]types.Transactions)
	TxPoolContentFrom(addr common.Address) (types.Transactions, types.Transactions)
	SubscribeNewTxsEvent(chan<- core.NewTxsEvent) event.Subscription

	// Filter API
    // 过滤api
    // Bloom filters are used in Ethereum to minimize the number of block queries that clients need to make.
    // 比如你query一笔交易的详细信息 他会返回logbloom。logsBloom is a 256 bytes string, and it’s not really a log in the classical sense. It’s the bloom filter for the logs of the block, and it allows to filter the hash of each element that is in the block. The objective is to minimize the number of queries that clients need to make by storing some events like historical transactions in the bloom.
	BloomStatus() (uint64, uint64)
	GetLogs(ctx context.Context, blockHash common.Hash) ([][]*types.Log, error)
	ServiceFilter(ctx context.Context, session *bloombits.MatcherSession)
	SubscribeLogsEvent(ch chan<- []*types.Log) event.Subscription
	SubscribePendingLogsEvent(ch chan<- []*types.Log) event.Subscription
	SubscribeRemovedLogsEvent(ch chan<- core.RemovedLogsEvent) event.Subscription

	ChainConfig() *params.ChainConfig
    // 共识engine 包含pos和poa
	Engine() consensus.Engine
}
```


## app.Run

