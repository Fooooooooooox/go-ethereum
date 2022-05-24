# difference between call callcode delegatecall

## call

```go
func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
	// Fail if we're trying to execute above the call depth limit
    // 超出栈的深度就报错
	if evm.depth > int(params.CallCreateDepth) {
		return nil, gas, ErrDepth
	}
	// Fail if we're trying to transfer more than the available balance
	if value.Sign() != 0 && !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
		return nil, gas, ErrInsufficientBalance
	}
	snapshot := evm.StateDB.Snapshot()
	p, isPrecompile := evm.precompile(addr)

    //这种情况应该是第一次创建合约
	if !evm.StateDB.Exist(addr) {
		if !isPrecompile && evm.chainRules.IsEIP158 && value.Sign() == 0 {
			// Calling a non existing account, don't do anything, but ping the tracer
            // 如果开启了debug模式 就开启了tracer模式 栈深度为0 start 深度大于0 enter
			if evm.Config.Debug {
				if evm.depth == 0 {
					evm.Config.Tracer.CaptureStart(evm, caller.Address(), addr, false, input, gas, value)
					evm.Config.Tracer.CaptureEnd(ret, 0, 0, nil)
				} else {
					evm.Config.Tracer.CaptureEnter(CALL, caller.Address(), addr, input, gas, value)
					evm.Config.Tracer.CaptureExit(ret, 0, nil)
				}
			}
			return nil, gas, nil
		}
		evm.StateDB.CreateAccount(addr)
	}
	evm.Context.Transfer(evm.StateDB, caller.Address(), addr, value)

	// Capture the tracer start/end events in debug mode
	if evm.Config.Debug {
		if evm.depth == 0 {
			evm.Config.Tracer.CaptureStart(evm, caller.Address(), addr, false, input, gas, value)
			defer func(startGas uint64, startTime time.Time) { // Lazy evaluation of the parameters
				evm.Config.Tracer.CaptureEnd(ret, startGas-gas, time.Since(startTime), err)
			}(gas, time.Now())
		} else {
			// Handle tracer events for entering and exiting a call frame
			evm.Config.Tracer.CaptureEnter(CALL, caller.Address(), addr, input, gas, value)
			defer func(startGas uint64) {
				evm.Config.Tracer.CaptureExit(ret, startGas-gas, err)
			}(gas)
		}
	}

	if isPrecompile {
		ret, gas, err = RunPrecompiledContract(p, input, gas)
	} else {
		// Initialise a new contract and set the code that is to be used by the EVM.
		// The contract is a scoped environment for this execution context only.
		code := evm.StateDB.GetCode(addr)
		if len(code) == 0 {
			ret, err = nil, nil // gas is unchanged
		} else {
			addrCopy := addr
			// If the account has no code, we can abort here
			// The depth-check is already done, and precompiles handled above
			contract := NewContract(caller, AccountRef(addrCopy), value, gas)
			contract.SetCallCode(&addrCopy, evm.StateDB.GetCodeHash(addrCopy), code)
			ret, err = evm.interpreter.Run(contract, input, false)
			gas = contract.Gas
		}
	}
	// When an error was returned by the EVM or when setting the creation code
	// above we revert to the snapshot and consume any gas remaining. Additionally
	// when we're in homestead this also counts for code storage gas errors.
	if err != nil {
		evm.StateDB.RevertToSnapshot(snapshot)
		if err != ErrExecutionReverted {
			gas = 0
		}
		// TODO: consider clearing up unused snapshots:
		//} else {
		//	evm.StateDB.DiscardSnapshot(snapshot)
	}
	return ret, gas, err
}
```

call传入的参数是caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int
caller： 该次交易的发起者
addr： 调用的合约地址
input： 传入合约函数的参数
gas： 传入的gas
value： 传入的eth

返回值是ret []byte, leftOverGas uint64, err error
ret：放着byte元素的数组 用来存放返回结果
leftOverGas：剩余的gas


1. 创建账户
    !evm.StateDB.Exist(addr)是什么意思？
   为了防止dos攻击或者资源浪费，以太坊有个设定，没有产生过交易的账户是不被记录到state中的。所以这里有个判断，如果这个账户不在state中存在就创建一下。
    exist是用来表示这个账户是否在state中 如果存在为1 如果不存在设为0（如果是suicided account也是1）

    createAccount函数
    newObj, prev := s.createObject(addr)是生成新账户
```go
func (s *StateDB) CreateAccount(addr common.Address) {
	newObj, prev := s.createObject(addr)
    // 如果账户已经存在 就把余额转移到新的账户 
	if prev != nil {
		newObj.setBalance(prev.data.Balance)
	}
}
```

2. transfer函数
   这里不知道为什么要transfer 初始化为什么要transfer先？
evm.Context.Transfer(evm.StateDB, caller.Address(), addr, value)

3. debug模式
   debug模式是使用tracer来记录 后面具体看 现在先略过

4. isPrecompile
   如果是precompile的合约 就可以直接将input gas等传入 然后直接执行了
   如何判断是不是precompile合约？
   ==> 
   p, isPrecompile := evm.precompile(addr)
   evm.precompile传入合约的地址，返回一个bool值来表示这个合约是否被precompiled
   可以看到不同的eth版本下有不一样的precompile合约

```go
func (evm *EVM) precompile(addr common.Address) (PrecompiledContract, bool) {
    // 这是一个map数组 键是common.Address 表示合约的地址，PrecompiledContract是一个自定义的interface 里面实现两种method：计算所需要的gas的函数RequiredGas和执行函数Run
	var precompiles map[common.Address]PrecompiledContract
	switch {
	case evm.chainRules.IsBerlin:
		precompiles = PrecompiledContractsBerlin
	case evm.chainRules.IsIstanbul:
		precompiles = PrecompiledContractsIstanbul
	case evm.chainRules.IsByzantium:
		precompiles = PrecompiledContractsByzantium
	default:
		precompiles = PrecompiledContractsHomestead
	}
	p, ok := precompiles[addr]
	return p, ok
}
```
拿berlin的例子来看一下：
```go
// PrecompiledContractsBerlin contains the default set of pre-compiled Ethereum
// contracts used in the Berlin release.
var PrecompiledContractsBerlin = map[common.Address]PrecompiledContract{
	common.BytesToAddress([]byte{1}): &ecrecover{},
	common.BytesToAddress([]byte{2}): &sha256hash{},
	common.BytesToAddress([]byte{3}): &ripemd160hash{},
	common.BytesToAddress([]byte{4}): &dataCopy{},
	common.BytesToAddress([]byte{5}): &bigModExp{eip2565: true},
	common.BytesToAddress([]byte{6}): &bn256AddIstanbul{},
	common.BytesToAddress([]byte{7}): &bn256ScalarMulIstanbul{},
	common.BytesToAddress([]byte{8}): &bn256PairingIstanbul{},
	common.BytesToAddress([]byte{9}): &blake2F{},
}
```

可以看到他在PrecompiledContractsBerlin的map数组填充了一系列的合约
键是十六进制的合约地址，比如你打印common.BytesToAddress([]byte{1}) 其实就是0x0000000000000000000000000000000000000001
值是一系列的struct的指针 其中包含了requiredgas和run方法
比如：&ecrecover{}是这样定义的：
```go
type ecrecover struct{}
func (c *ecrecover) RequiredGas(input []byte) uint64 {
	return params.EcrecoverGas
}
func (c *ecrecover) Run(input []byte) ([]byte, error) {
    ···
}
```
这些PrecompiledContract结构就是自带run方法的合约，这个run方法是以go函数的形式直接写在geth代码中的，可以直接执行，而不用走solidity解释合约这一过程。
所以这些合约其实是库合约，比如ecrecover 就是椭圆曲线合约，sha256hash是hash函数合约，这些合约是智能合约开发的基础，基本上每个智能合约都会调用到这些基础的库合约，把这些库合约直接用go的形式写在geth代码中就不用每个智能合约再部署一遍了，节约了资源。














