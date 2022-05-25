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

1. op：    当前的opcode

2. mem：   新开的内存

实际上 memory就是一个指针 指向一个struct结构体

结构体中包含了一个byte数组用于记录store 一个uint64整数用来记录上次花费的gas

NewMemory()会返回一个空的结构体的地址 长这样： &{[] 0}

store是一个空的byte数组，可以将一个字符串转换为一个byte数组（似乎只能一个字符串

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

3. stack： 栈

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

# what is pool？

这一块需要理解pool这个概念

pool是一系列的temporary objectsx


   





