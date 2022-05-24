# difference between call callcode delegatecall

## we dive deep into call first cuz it's basic and fundemental
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

4. solidity内置合约：precompile
   
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

值是一系列的struct的 其中包含了requiredgas和run方法

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

所以这些合约其实是库合约，也就是solidity支持的内置函数，比如ecrecover 就是椭圆曲线合约，sha256hash是hash函数合约，这些合约是智能合约开发的基础，基本上每个智能合约都会调用到这些基础的库合约，把这些库合约直接用go的形式写在geth代码中就不用每个智能合约再部署一遍了，节约了资源。

整理一下逻辑：

先判断是否在state中存在

如果不存在 且满足不是precompile、evm.chainRules.IsEIP158 && value.Sign()就返回nil（这应该是某种错误的情况

如果是precomile合约 就在state中创建这个合约地址 然后直接运行 计算gas 返回res（因为是precompile库合约，所以状态中是没有code存储的，这里是直接创建一个新的地址来代表库合约）

如果不是precomile合约 就需要initialize新合约

5. initialise a new contract
   
   合约地址不在state中、不是precompile说明这是一个从未被调用过的智能合约 这是首次调用 需要先创建他

   逻辑如下：

```go
else {
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
```
   调用statedb拿到该地址的code

   如果code为空 就直接return nil gas不变

   否则调用NewContract来创建合约执行的环境

   
   newcontract的逻辑：

   传入caller地址、 合约地址等 返回contract指针
```go
// NewContract returns a new contract environment for the execution of EVM.
func NewContract(caller ContractRef, object ContractRef, value *big.Int, gas uint64) *Contract {
	c := &Contract{CallerAddress: caller.Address(), caller: caller, self: object}

	if parent, ok := caller.(*Contract); ok {
		// Reuse JUMPDEST analysis from parent context if available.
		c.jumpdests = parent.jumpdests
	} else {
		c.jumpdests = make(map[common.Hash]bitvec)
	}

	// Gas should be a pointer so it can safely be reduced through the run
	// This pointer will be off the state transition
	c.Gas = gas
	// ensures a value is set
	c.value = value

	return c
}
```
Contract是一个结构体，其中包含：

CallerAddress：合约创建者的地址

caller： 

self： （这两个不知道具体是什么

jumpdests： jumpdest数组

analysis： 

Code： 存放code的byte数组

CodeHash： code的hash

CodeAddr：

Input： 

Gas：

value： 

最重要的应该是code和jumpdests相关的东西

继续看NewContract

如果可以从parent context中获取JUMPDEST analysis就从parent.jumpdests中获取，否则新建一个map数组来存放。

这个JUMPDEST analysis是什么？

6. JUMPDEST
   
   JUMPDEST是虚拟机的跳转指令

   jumpdests是一个map数组 键是hash 值是bitvec

   bitvec指bit vector，是用来干嘛的？

   bitvec是用来看程序是指令还是数据，如果是unset bit，也就是bitvec为0，说明这是一个opcode指令，set bit说明这是数据（比如是push指令的argument数值）

   bitvec 支持set1 setN set8等函数 例如set1：

```go
func (bits bitvec) set1(pos uint64) {
	bits[pos/8] |= 1 << (pos % 8)
}
```
7. bitvec
所以bitvec到底是用来做啥的？

我查了一下资料：

https://blog.csdn.net/johnny710vip/article/details/24394471?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-24394471-blog-69524075.pc_relevant_default&spm=1001.2101.3001.4242.2&utm_relevant_index=4

这个属于数据结构的知识 bitvec在充分利用小空间存储大量数据方面非常具有优势 

Linux内核中很多地方都是用了位图

bitvec又叫做位向量 是一个只包含0和1的数组

举个例子来说明bitvec的需求场景：

如何用一个对象来表示“是”或者“否”这两种情况？

一般首先会想到：用int型变量来表示 

另外 也可以使用var来表示 虽然占的内存空间小了一些 但是还是非常浪费

计算机中最小的数据位是非0即1的二进制位 其实对于我们需要的对象 使用一个二进制位就可以了 

但是计算机中不能用一个变量来表示一个位 所以就把多个位组合起来成为一个基本的数据类型 对组合体进行操作来实现对位的操作


理解一下 evm中的bitvec是用来表示合约操作的，他会分析整个操作，然后用bitvec中的0和1来表示操作是指令还是数据。

有了每个合约操作的bitvec数组之后，就可以查看某个位置是code segment（指令）还是数据

```go
func (bits *bitvec) codeSegment(pos uint64) bool {
	return (((*bits)[pos/8] >> (pos % 8)) & 1) == 0...
}
```
pos是外部输入的参数 暂时我也不清楚是怎么用的 （/todo/ 后面会了补充上来


也可以利用bitvec来找出合约指令中所有数据的位置：
```go
// codeBitmap collects data locations in code.
func codeBitmap(code []byte) bitvec {
	// The bitmap is 4 bytes longer than necessary, in case the code
	// ends with a PUSH32, the algorithm will push zeroes onto the
	// bitvector outside the bounds of the actual code.
	bits := make(bitvec, len(code)/8+1+4)
	return codeBitmapInternal(code, bits)
}
```

8. interpreter.Run
看完contract返回的结构体，回到最初call的逻辑.
在拿到newcontract返回的合约执行环境结构体之后，把callcode中的内容写入到contract结构体中。
然后就是整个call中最主要的环节：run

Run函数还挺长的 所以我分段拿其中重要的逻辑分析


（run放另一个文件里了




