# what is endian byte order

here's the code:

```go
func isLittleEndian() bool {
	n := uint32(0x01020304)
	// 判断返回值是不是0x04 来确定的
	return *(*byte)(unsafe.Pointer(&n)) == 0x04
}
```

Computers store data in memory in binary. One thing that is often overlooked is the formatting at the byte level of this data. This is called endianness and it refers to the ordering of the bytes.

Specifically, little-endian is when the least significant bytes are stored before the more significant bytes, and big-endian is when the most significant bytes are stored before the less significant bytes.

所以endian是关于一个数如何在计算机内存中存储 不同的处理器会有不同
比如 x86_64 processors (Intel/AMD 使用的是little-endian
IP/TCP 使用的是 big-endian

上面这段代码 是传入了一个数 0x0102030 使用unsafe.Pointer拿到内存位置 

# how difficulties are adjusted?

以太坊的不同版本升级有不同的难度值调整方式

以太坊在未来会把poa转换为pos 所以他们计划出一个难度炸弹 将区块的难度值调整到非常高的水平 从而给之前投入大量矿机的矿工一个过渡期（让他们知道挖矿是个死胡同

在最开始的frontier版本中 难度值的调整是：
```python
step = parent_diff // 2048
direction = 1 if block_timestamp - parent_timestamp < 13 else -1
expAdjust = 2**((block.number // 100000) - 2) if block.number >= 100000 else 0

Header.Difficulty = parent_diff + step * direction + expAdjust
```
出新块的时间小于13 方向为正 下一个区块的难度值就会变大 
出新块的时间大于13 方向为负 下一个区块的难度值减小
expadjust就是埋下的难度值炸弹 他表示当区块高度达到100000的时候 会给难度值增加一个expadjust的项 这个项是随着区块高度的上升呈现指数级上升的 这也是为什么你看以太坊区块难度的时间序列 会发现有一段时间难度值疯狂上升

再后来难度值又下降了 为什么 ==》 因为以太坊官方估计pos转型太乐观了 实际上的开发进度太慢了 ==》所以可以看到以太坊后面多次调整难度值的计算方式 不断换着法子把炸弹延期

# 关于dag（hash源

官方文档：

https://github.com/ethereum/wiki/wiki/Dagger-Hashimoto

asic芯片本身的性能是最适合挖矿的 以太坊人为设计算法阻止asic的使用

本来可以直接用区块头做为hash源的 以太坊专门搞了个数据库占你内存 数据库里放dataset 存放矿工挖矿的hash种子源 
