# 写在前面···
interpreter.Run是call中最重要的部分了
分段看他的逻辑

# run

```go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
}
```

传入的参数是contract结构体，input数组，还有表示是只读还是读写。返回的是包含byte元素的数组和error信息。


```go
// Increment the call depth which is restricted to 1024
in.evm.depth++
defer func() { in.evm.depth-- }()
```

每调用一次就会增加栈的深度，但是这个栈的深度不能超过1024。
这里的栈深度具体是如何计算的？以evm interpreter为单位？以block为单位？以一次合约交易为单位？

==》
原来是在一个合约中可以使用new来创建另一个合约，实现递归调用，这个depth就是用来记录合约递归调用的次数的，合约递归调用的层数不能超过1024

```go
	var (
		op          OpCode        // current opcode
        // mem是新开的memory位置
		mem         = NewMemory() // bound memory
		stack       = newstack()  // local stack
		callContext = &ScopeContext{
			Memory:   mem,
			Stack:    stack,
			Contract: contract,
		}
		// For optimisation reason we're using uint64 as the program counter.
		// It's theoretically possible to go above 2^64. The YP defines the PC
		// to be uint256. Practically much less so feasible.
		pc   = uint64(0) // program counter
		cost uint64
		// copies used by tracer
		pcCopy  uint64 // needed for the deferred EVMLogger
		gasCopy uint64 // for EVMLogger to log gas remaining before execution
		logged  bool   // deferred EVMLogger should ignore already logged steps
		res     []byte // result of the opcode execution function
	)
```
# 重要的变量

## 1. op：    当前的opcode

## 2. mem：   新开的内存

实际上 memory就是一个指针 指向一个struct结构体

结构体中包含了一个byte数组用于记录store 一个uint64整数用来记录上次花费的gas

NewMemory()会返回一个空的结构体的地址 长这样： &{[] 0}

store是一个空的byte数组，可以将字符串转换为一个byte数组

例如： mem.store = []byte("hahaha up up and away")会得到：[104 97 104 97 104 97 32 117 112 32 117 112 32 97 110 100 32 97 119 97 121]


```go
// Memory implements a simple memory model for the ethereum virtual machine.
type Memory struct {
	store       []byte
	lastGasCost uint64
}

// NewMemory returns a new memory model.
func NewMemory() *Memory {
	return &Memory{}
}
```

## 3. stack： 栈

```go
type Stack struct {
	data []uint256.Int
}

func newstack() *Stack {
	return stackPool.Get().(*Stack)
}

```
栈的处理看起来和memory差不多 不过它多了一层处理

在用Newmemory新建memory的时候返回的是memory的指针：*Memory

但是用newstack新建stack的时候返回的是stackPool.Get().(*Stack)

为什么？

### what is pool？

要理解stack要先理解pool的概念

pool是go官方提供的并发相关的库 他是一个非常巧妙的库

查看go源码可以看到pool是一系列的temporary objects（缓存池）其中的对象可以被多个gorotines同时存储和读取 

所以pool就是用来存储被allocated的对象以便后面调用和释放，这样可以减轻garbage collector的压力，提高效率，并实现thread-safe

用pool的形式来存储栈的结构会带来更好的性能

```go
type Stack struct {
	data []uint256.Int
}

var stackPool = sync.Pool{
	New: func() interface{} {
		return &Stack{data: make([]uint256.Int, 0, 16)}
	},
}
```
stack是一个结构体，里面只有data这一个元素，data是一个数组，里面存放uint256

stack的池子里存放的是stacK的地址：&Stack{data: make([]uint256.Int, 0, 16)} data数组的最大长度被限制为16（初始化为0）

## callContext

```go
callContext = &ScopeContext{
			Memory:   mem,
			Stack:    stack,
			Contract: contract,
		}

type ScopeContext struct {
	Memory   *Memory
	Stack    *Stack
	Contract *Contract
}
```

callContext包含了pre-call的信息 比如stack和memory 还有前面生成的合约的环境

## pc

pc指的是program counter 不知道是做什么的？

==》 原来是这是个程序计数器 是一个正整数

op = contract.GetOp(pc)

合约的op指令就是以pc的值做为索引拿到的：

```go
// GetOp returns the n'th element in the contract's byte array
func (c *Contract) GetOp(n uint64) OpCode {
	if n < uint64(len(c.Code)) {
		return OpCode(c.Code[n])
	}

	return STOP
}
```
code是contract里的一个对象，是一个[]byte数组，里面应该是合约的bytecode。

contract.GetOp(pc)传入一个整数pc，如果pc的值小于code的长度就返回这个位置的opcode，如果pc的值等于or大于code的长度就返回stop指令。

opcode是一个十六进制的byte，每一个opcode代表着一个指令，拿到指令之后就可以让虚拟机执行相关操作。

比如：

STOP       OpCode = 0x0

ADD        OpCode = 0x1

MUL        OpCode = 0x2



## input

contract.Input = input

input是一个byte数组，也就是调用合约时传入的calldata

在使用remix中的单元测试的时候，你把参数填到函数的框框里，remix会根据你调用的函数和你传入的参数为你生成calldata然后传入。


## interpreter loop

run过程中最重要的loop

op = contract.GetOp(pc)

拿到了op之后从jumptable中获取operation

operation := in.cfg.JumpTable[op]

jumptable是什么？

jumptable是一个数组 里面的元素是*operation

operation是一个结构体，其中最重要的是execute执行函数。

执行函数会接收pc、interpreter、callcontext等参数，执行之后返回结果。

```go
type JumpTable [256]*operation

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
如果在执行过程中报错就向前移动pc，返回错误。正常执行的话，随着pc不断增加，evm会把合约中每一个operation都执行一遍，最终返回结果。

```go
res, err = operation.execute(&pc, in, callContext)
```




