# mpt
Recently I've been watching on some zk projects like zksync and scroll.

There is a concept called mpt circuit, so I tried to understand what mpt is in geth itself and how will it be like when zkp is applied.

There are 4 kinds of nodes in mpt tree.

1. value node
   ```go
   valueNode []byte
   ```
2. short node
   ```go
   	shortNode struct {
		Key   []byte
		Val   node
		flags nodeFlag
	}
   ```
3. full node
   ```go
   	fullNode struct {
		Children [17]node // Actual trie node data to encode/decode (needs custom encoder)
		flags    nodeFlag
	}
   ```
4. hash node
   ```go
   hashNode  []byte
   ```
   

"MPT circuit checks that the modification of the trie state happened correctly.“

The circuit checks the transition from val1 to val2 at key1 that led to the change of trie root from root1 to root2 (the chaining of such proofs is yet to be added)"

how does zkp proves mpt correct?

