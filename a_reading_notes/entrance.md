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
   1. 导入了一堆command 例如 （具体定义在chaincmd.go==》chaincmd里面是一堆你通过命令行方式传递的配置信息
   2. initcommand 初始化命令。接收通过flag形式传递的参数作为你节点的配置信息（比如lightmode archive fast 测试网等flag设置）。接收传入genesis file （可以指定网络是主网还是测试网
   3. dumpgenesis 是把创世区块的配置信息以json格式输出到stdout标准输出文件中（估计是生成一个配置文件到你的geth目录里 config.json之类的东西，里面是你设置的flag参数（更方便
   4. importcommand应该是方便你导入以前已经同步好了的数据
   5. exportcommand是导出区块数据到文件中
   6. initgenesis从你指定的genesis path中读取到关于genesis配置的json文件（genesis的参数设置在params/config.go里，里面有chainconfig可以指定你选择哪一个分支（比如hometead daofork) ==> 同步这种废弃的分支来做啥？==》一般是用来测试
   
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

