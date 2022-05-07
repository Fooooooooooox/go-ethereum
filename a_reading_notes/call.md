# 合约

## 创建合约
```go
// create creates a new contract using code as deployment code.
func (evm *EVM) create(caller ContractRef, codeAndHash *codeAndHash, gas uint64, value *big.Int, address common.Address, typ OpCode) ([]byte, common.Address, uint64, error) {
	// Depth check execution. Fail if we're trying to execute above the
	// limit.
	if evm.depth > int(params.CallCreateDepth) {
		return nil, common.Address{}, gas, ErrDepth
	}
	if !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
		return nil, common.Address{}, gas, ErrInsufficientBalance
	}
	nonce := evm.StateDB.GetNonce(caller.Address())
	if nonce+1 < nonce {
		return nil, common.Address{}, gas, ErrNonceUintOverflow
	}
	evm.StateDB.SetNonce(caller.Address(), nonce+1)
	// We add this to the access list _before_ taking a snapshot. Even if the creation fails,
	// the access-list change should not be rolled back
	if evm.chainRules.IsBerlin {
		evm.StateDB.AddAddressToAccessList(address)
	}
	// Ensure there's no existing contract already at the designated address
	contractHash := evm.StateDB.GetCodeHash(address)
	if evm.StateDB.GetNonce(address) != 0 || (contractHash != (common.Hash{}) && contractHash != emptyCodeHash) {
		return nil, common.Address{}, 0, ErrContractAddressCollision
	}
	// Create a new account on the state
	snapshot := evm.StateDB.Snapshot()
	evm.StateDB.CreateAccount(address)
	if evm.chainRules.IsEIP158 {
		evm.StateDB.SetNonce(address, 1)
	}
	evm.Context.Transfer(evm.StateDB, caller.Address(), address, value)

	// Initialise a new contract and set the code that is to be used by the EVM.
	// The contract is a scoped environment for this execution context only.
	contract := NewContract(caller, AccountRef(address), value, gas)
	contract.SetCodeOptionalHash(&address, codeAndHash)

	if evm.Config.Debug {
		if evm.depth == 0 {
			evm.Config.Tracer.CaptureStart(evm, caller.Address(), address, true, codeAndHash.code, gas, value)
		} else {
			evm.Config.Tracer.CaptureEnter(typ, caller.Address(), address, codeAndHash.code, gas, value)
		}
	}

	start := time.Now()

	ret, err := evm.interpreter.Run(contract, nil, false)

	// Check whether the max code size has been exceeded, assign err if the case.
	if err == nil && evm.chainRules.IsEIP158 && len(ret) > params.MaxCodeSize {
		err = ErrMaxCodeSizeExceeded
	}

	// Reject code starting with 0xEF if EIP-3541 is enabled.
	if err == nil && len(ret) >= 1 && ret[0] == 0xEF && evm.chainRules.IsLondon {
		err = ErrInvalidCode
	}

	// if the contract creation ran successfully and no errors were returned
	// calculate the gas required to store the code. If the code could not
	// be stored due to not enough gas set an error and let it be handled
	// by the error checking condition below.
	if err == nil {
		createDataGas := uint64(len(ret)) * params.CreateDataGas
		if contract.UseGas(createDataGas) {
			evm.StateDB.SetCode(address, ret)
		} else {
			err = ErrCodeStoreOutOfGas
		}
	}

	// When an error was returned by the EVM or when setting the creation code
	// above we revert to the snapshot and consume any gas remaining. Additionally
	// when we're in homestead this also counts for code storage gas errors.
	if err != nil && (evm.chainRules.IsHomestead || err != ErrCodeStoreOutOfGas) {
		evm.StateDB.RevertToSnapshot(snapshot)
		if err != ErrExecutionReverted {
			contract.UseGas(contract.Gas)
		}
	}

	if evm.Config.Debug {
		if evm.depth == 0 {
			evm.Config.Tracer.CaptureEnd(ret, gas-contract.Gas, time.Since(start), err)
		} else {
			evm.Config.Tracer.CaptureExit(ret, gas-contract.Gas, err)
		}
	}
	return ret, address, contract.Gas, err
}
```


evm.create会接受部署合约的代码、地址、gas等信息

检查：
1. 合约创建的递归调用次数，params/protocol_params.go中记录了一个参数：CallCreateDepth uint64 = 1024，合约的递归调用次数不能大于这个值（这是因为以太坊区块大小是存在上限的，如果区块很大很大，矿工运行节点就会有比较高的门槛）
2. 检查地址余额大小，是不是有足够余额来支付gas fee
3. 获取到合约创建者的地址，检查是否存在随机数溢出，然后给随机数加1

创建合约账户 根据合约传入的chain config来生成合约的执行环境 
```go
// Create a new account on the state
	snapshot := evm.StateDB.Snapshot()
	evm.StateDB.CreateAccount(address)
	if evm.chainRules.IsEIP158 {
		evm.StateDB.SetNonce(address, 1)
	}
	evm.Context.Transfer(evm.StateDB, caller.Address(), address, value)

	// Initialise a new contract and set the code that is to be used by the EVM.
	// The contract is a scoped environment for this execution context only.
	contract := NewContract(caller, AccountRef(address), value, gas)
	contract.SetCodeOptionalHash(&address, codeAndHash)
```
   
用contract := NewContract初始化合约生成一个新的contract对象，给合约对象设置code。

然后调用evm.interpreter.Run执行一遍合约，把返回的结果记为ret。
evm.interpreter.Run这个函数是用来评估合约、接受input来初始化合约的。input应该是你在部署合约的时候传入的construct里的参数（比如nft合约的Max supply mint价格之类的东西？）

关于run函数下面仔细看。


对ret的值进行size检查，太大的话就拒绝写入抛出错误，然后就可以根据ret的size来计算需要的gas，如果合约创建者的余额足够支付，就把ret写入到statedb中。
如果在这个过程中报错了，就使用在创建合约之前生成的snapshot进行回滚。



```go
// Create creates a new contract using code as deployment code.
func (evm *EVM) Create(caller ContractRef, code []byte, gas uint64, value *big.Int) (ret []byte, contractAddr common.Address, leftOverGas uint64, err error) {
	contractAddr = crypto.CreateAddress(caller.Address(), evm.StateDB.GetNonce(caller.Address()))
	return evm.create(caller, &codeAndHash{code: code}, gas, value, contractAddr, CREATE)
}
```



## 调用合约
### 创建合约evm执行环境

将contractRef、value、gas传入NewContract，会返回一个contract对象：
&Contract{CallerAddress: caller.Address(), caller: caller, self: object}

生成contract对象之后，调用caller.(*Contract)，看看这个caller内有没有缓存JUMPDEST analysis，如果有就直接复用没有就来生成一个空的jumpdests。

contract对象的定义如下：
你会发现这里面有两个地址的定义：CallerAddress和caller
这其实和callcode有关 方便你把这个合约内的函数当作库一样复用
calleraddress中存储的是最初这个合约创建者的地址，这是一个不变量
ContractRef中存储的则是一个可变的地址，谁在调用这个合约这个地址就是谁的
```go
type ContractRef interface {
	Address() common.Address
}
```

其中看不太明白的是jumpdests，它是一个map，键是Hash，值是bitvec
analysis中存储的jumpdests的缓存（下面仔细研究

在core/vm/opcodes.go中的'storage' and execution下有JUMPDEST值的定义，JUMPDEST OpCode = 0x5b

在core/vm/jump_table.go中有关于JUMPDEST的定义：
原来jumptable是用来存放evm opcode的（存放你当前fork下支持的所有evm opcode）
```go
func newFrontierInstructionSet() JumpTable {
	tbl := JumpTable{
        ···
        ···
    }
}
```

其他的比如code codehash input这些都比较显而易见

```go
type Contract struct {
	// CallerAddress is the result of the caller which initialised this
	// contract. However when the "call method" is delegated this value
	// needs to be initialised to that of the caller's caller.
	CallerAddress common.Address
	caller        ContractRef
	self          ContractRef

	jumpdests map[common.Hash]bitvec // Aggregated result of JUMPDEST analysis.
	analysis  bitvec                 // Locally cached result of JUMPDEST analysis

	Code     []byte
	CodeHash common.Hash
	CodeAddr *common.Address
	Input    []byte

	Gas   uint64
	value *big.Int
}
```

## Run函数解析
最前面这一块挺简单的 就是调用一次栈的深度加一 还有清空上次的数据啥的
```go
	// Increment the call depth which is restricted to 1024
	in.evm.depth++
	defer func() { in.evm.depth-- }()

	// Make sure the readOnly is only set if we aren't in readOnly yet.
	// This also makes sure that the readOnly flag isn't removed for child calls.
	if readOnly && !in.readOnly {
		in.readOnly = true
		defer func() { in.readOnly = false }()
	}

	// Reset the previous call's return data. It's unimportant to preserve the old buffer
	// as every returning call will return new data anyway.
    // 执行前先清空上一次call的return data
	in.returnData = nil

	// Don't bother with the execution if there's no code.
    // 要是合约里都没代码就直接返回了
	if len(contract.Code) == 0 {
		return nil, nil
	}

```

## stack变量

stack       = newstack()  // local stack会生成一个栈的对象，newStack的定义如下：

stack就是一个数组，里面的元素是uint256.Int。

stack有一些操作方法：push pop len dup等。

这些都是一些栈的操作

比如push是把一个uint256.Int的指针写入栈的数组中。
所以栈里面保存的不是这个值，而是这个值的指针内存位置。
pop是把数组切块，把末尾的那个指针给删了.
peek是返回栈的末尾的一个数。
(我写了个程序实现了一下栈的操作 在go_learning_booking_app里)


```go
type Stack struct {
	data []uint256.Int
}
```


### Run里的for循环
比较复杂且重要的是下面这个for循环(我看的时候先忽略了debug模式下的处理：

```go
	for {
		if in.cfg.Debug {
			// Capture pre-execution values for tracing.
			logged, pcCopy, gasCopy = false, pc, contract.Gas
		}
		// Get the operation from the jump table and validate the stack to ensure there are
		// enough stack items available to perform the operation.
		op = contract.GetOp(pc)
		operation := in.cfg.JumpTable[op]
		cost = operation.constantGas // For tracing
		// Validate stack
		if sLen := stack.len(); sLen < operation.minStack {
			return nil, &ErrStackUnderflow{stackLen: sLen, required: operation.minStack}
		} else if sLen > operation.maxStack {
			return nil, &ErrStackOverflow{stackLen: sLen, limit: operation.maxStack}
		}
		if !contract.UseGas(cost) {
			return nil, ErrOutOfGas
		}
		if operation.dynamicGas != nil {
			// All ops with a dynamic memory usage also has a dynamic gas cost.
			var memorySize uint64
			// calculate the new memory size and expand the memory to fit
			// the operation
			// Memory check needs to be done prior to evaluating the dynamic gas portion,
			// to detect calculation overflows
			if operation.memorySize != nil {
				memSize, overflow := operation.memorySize(stack)
				if overflow {
					return nil, ErrGasUintOverflow
				}
				// memory is expanded in words of 32 bytes. Gas
				// is also calculated in words.
				if memorySize, overflow = math.SafeMul(toWordSize(memSize), 32); overflow {
					return nil, ErrGasUintOverflow
				}
			}
			// Consume the gas and return an error if not enough gas is available.
			// cost is explicitly set so that the capture state defer method can get the proper cost
			var dynamicCost uint64
			dynamicCost, err = operation.dynamicGas(in.evm, contract, stack, mem, memorySize)
			cost += dynamicCost // for tracing
			if err != nil || !contract.UseGas(dynamicCost) {
				return nil, ErrOutOfGas
			}
			if memorySize > 0 {
				mem.Resize(memorySize)
			}
		}
		if in.cfg.Debug {
			in.cfg.Tracer.CaptureState(pc, op, gasCopy, cost, callContext, in.returnData, in.evm.depth, err)
			logged = true
		}
		// execute the operation
		res, err = operation.execute(&pc, in, callContext)
		if err != nil {
			break
		}
		pc++
	}
```

pc的初始值是0
op = contract.GetOp(pc)
GetOp()这个函数是在core/vm/contract.go中定义的，作用是你传入一个uint，对contract对象调用c.GetOp()就会以OpCode（二进制）的形式返回code（code是一个数组）的第n位数据，长这样：
```go
// GetOp returns the n'th element in the contract's byte array
func (c *Contract) GetOp(n uint64) OpCode {
	if n < uint64(len(c.Code)) {
		return OpCode(c.Code[n])
	}

	return STOP
}
```

拿到op后把op作为一个索引，从jumptable中找到对应的operation

中间是一些不太重要的，比如对stack进行长度检查，计算gas消耗等等。

执行operation的代码：
res, err = operation.execute(&pc, in, callContext)
这里有点奇怪的是为什么这句语句直接写在了for循环之中，最终直接返回了这个res？
==》
应该是因为operation.execute函数本身就是对res进行了一个append的处理，而不是直接replace。
具体看一下operation.execute函数：
operation := in.cfg.JumpTable[op]，execute是它的一个interface功能 
core/vm/jump_table.go中有一句：
type JumpTable [256]*operation

### operation & jumptable

说明JumpTable是一个存放operation指针的数组，不一样的分支会有不一样的chainconfig，jumptable内的operation指针也就不一样。
operation的定义如下：
```go
type operation struct {
	// execute is the operation function
	execute     executionFunc
	constantGas uint64
	dynamicGas  gasFunc
	// minStack tells how many stack items are required
	minStack int
	// maxStack specifies the max length the stack can have for this operation
	// to not overflow the stack.
	maxStack int

	// memorySize returns the memory size required for the operation
	memorySize memorySizeFunc
}
```

我看了一下生成jumptable的代码，一开始没看懂，因为他定义了一连串的 instructionset生成的函数，最上面那个函数一连串地调用下面的函数。

看懂了其实很简单：

最下面定义的newFrontierInstructionSet函数只包含了最基础需要的opcode，其他的instructionset函数都是在这个基础上新增新的指令。这和以太坊升级的过程是一样的，每次升级都会在原有指令的基础上新增一些指令。


```go
// newLondonInstructionSet returns the frontier, homestead, byzantium,
// contantinople, istanbul, petersburg, berlin and london instructions.
func newLondonInstructionSet() JumpTable {
	instructionSet := newBerlinInstructionSet()
	enable3529(&instructionSet) // EIP-3529: Reduction in refunds https://eips.ethereum.org/EIPS/eip-3529
	enable3198(&instructionSet) // Base fee opcode https://eips.ethereum.org/EIPS/eip-3198
	return validate(instructionSet)
}
```

### operation.execut
继续看operation.execute函数。
for循环中每次都会调用这个operation.execute函数：res, err = operation.execute(&pc, in, callContext)。所以具体看下execute函数的内容就知道虚拟机指令是如何执行的了。

execute是operation type中的参数，如下：
另外的参数都很简单，关于dynamic gas在下面做出说明：

```go
type operation struct {
	// execute is the operation function
	execute     executionFunc
	constantGas uint64
	dynamicGas  gasFunc
	// minStack tells how many stack items are required
	minStack int
	// maxStack specifies the max length the stack can have for this operation
	// to not overflow the stack.
	maxStack int

	// memorySize returns the memory size required for the operation
	memorySize memorySizeFunc
}
```

拿个最简单的add指令来看下：

```go
ADD: {
    execute:     opAdd,
    constantGas: GasFastestStep,
    minStack:    minStack(2, 1),
    maxStack:    maxStack(2, 1),
}
```
GasFastestStep是constant gas，GasFastestStep uint64 = 3。执行一次add指令gas就加3。

minstack和maxstack是在core/vm/stack_table.go中被定义的。

```go
func maxStack(pop, push int) int {
	return int(params.StackLimit) + pop - push
}
func minStack(pops, push int) int {
	return pops
}
```
输入pop和push两个整数，minstack返回pop，maxstack返回int(params.StackLimit) + pop - push。
params.StackLimit被记录在params/protocol_params.go中，StackLimit   uint64 = 1024  // Maximum size of VM stack allowed.
它定义了vm最深栈的深度。

pop是把一个参数弹出栈，push是把一个参数压入栈，所以：
指令的最大栈深度 = 虚拟机栈的深度 + pop - push
指令的最小栈深度 = pops

重点看opAdd这个execute函数
```go
func opAdd(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
	x, y := scope.Stack.pop(), scope.Stack.peek()
	y.Add(&x, y)
	return nil, nil
}
```
Stack.pop()对把栈最末尾的数踢出去了，被踢出去的这个值赋值给了x。
被pop过的栈调用scope.Stack.peek()，返回现在栈的最末尾的值，赋值给了y。
然后



## 关于dynamic gas和constant gas

https://ethereum.org/en/developers/docs/gas/

在伦敦升级前，以太坊的区块大小是固定的，在需求很大的时候人们将交易写入区块的等待时间就会变得很长。在伦敦升级之后，区块的大小被改成了一个浮动的区间，区块大小有了一个target：15million的gas，也就是区块大小的均值为15，同时会随着区块的需求大小变化而变化（不过有一个30的上限）。



