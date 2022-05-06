


## what is callcode and delegatecall？

callcode和call的区别：
call一个合约A的时候你执行合约的地址是A合约的地址，你执行的是A合约的函数作用在A合约上，被调用的合约地址仍是A，改变的storage是A合约的storage。
callcode一个合约实现了库的功能，你在callcode一个合约A的时候，你是拿了A合约的code作用在你自己的合约上，所以被调用的合约实际上是你自己的合约，只是拿来A合约的方法来改变你自己合约的storage。

下面是callcode的evm相关代码：
address作为一个参数传入，
```go
// CallCode executes the contract associated with the addr with the given input
// as parameters. It also handles any necessary value transfer required and takes
// the necessary steps to create accounts and reverses the state in case of an
// execution error or failed value transfer.
//
// CallCode differs from Call in the sense that it executes the given address'
// code with the caller as context.
func (evm *EVM) CallCode(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
    ···
	// It is allowed to call precompiles, even via delegatecall
	if p, isPrecompile := evm.precompile(addr); isPrecompile {
		ret, gas, err = RunPrecompiledContract(p, input, gas)
	} else {
		addrCopy := addr
		// Initialise a new contract and set the code that is to be used by the EVM.
		// The contract is a scoped environment for this execution context only.
		contract := NewContract(caller, AccountRef(caller.Address()), value, gas)
		contract.SetCallCode(&addrCopy, evm.StateDB.GetCodeHash(addrCopy), evm.StateDB.GetCode(addrCopy))
		ret, err = evm.interpreter.Run(contract, input, false)
		gas = contract.Gas
	}
    ···
	return ret, gas, err
}
```
传入address hash code