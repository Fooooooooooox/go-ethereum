# DATA STRUCTURE DESIGN in ETHEREUM and BITCOIN
I appreciate independent and critical spirit when learning and I love asking questions like why things works like that. So this article will be written along my own thoughts and lines because I want it to be more like a thought-provoking essay instead of a reference book telling you how things work in Ethereum and blockchain.

# What is Blockchain? Blockchain V.S. Linked List

We start from a simple question: what is blockchain?

Blockchain is a data structure. In order to understand blockchain, you must learn about linked list first.

In a linked list, nodes are connected by links using a pointer.
There 2 elements in a node:
1. a pointer to the next node
2. a value part to store data
   
In a linked list, you can easily insert and delete new nodes by changing the pointer of the nodes without affecting other nodes.

Blockchain is a linked list that uses hash pointers to connect nodes. What do hash pointers mean?
Hash pointer includes both a pointer to the previous node and a hash value of the data stored in the previous node.

This hash value changes if the content changes. So it makes blockchain immutable with tamper-evident logs. For example, you change the value content of one node, the node next to it and so on will all changed. So you can keep the hash of the most recent block and prove that all the transactions are correct. That's where the light node coms from. Light node only keeps the recent blocks and querys additional data from full node when needed.

# Merkle Tree and Merkle Patricia Tree

Now we know how blocks are connected. Let's see how transactions and account data are stored in blocks.

In Bitcoin, transactions are stored in merkle tree.

Transactions are stored in merkle tree. 

