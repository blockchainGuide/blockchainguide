> 死磕以太坊源码分析之EVM指令集.md
>
> 配合以下代码进行阅读：https://github.com/blockchainGuide/
>
> 写文不易，给个小关注，有什么问题可以指出，便于大家交流学习。
>
> 以下指令集持续更新，最新文章请参考上面

![f4a720afd869d70c3d1d2149980ba0e9](https://tva1.sinaimg.cn/large/008eGmZEgy1gn1z0rqeqaj31c00u0k20.jpg)

## EVM 指令集概念

**EVM执行的是字节码**。由于操作码被限制在一个字节以内，所以EVM指令集最多只能容纳**256**条指令。目前EVM已经定义了`100`多条指令，还有**100**多条指令可供以后扩展。**这100多条指令包括算术运算指令，比较操作指令，按位运算指令，密码学计算指令，栈、memory、storage操作指令，跳转指令，区块、智能合约相关指令等**。



## EVM指令集

### 算数运算指令集

> *0x0*

```
	STOP:       "STOP",
	ADD:        "ADD", //加法运算
	MUL:        "MUL", //乘法运算
	SUB:        "SUB", //减法运算
	DIV:        "DIV", //无符号整除运算
	SDIV:       "SDIV", //有符号整除运算
	MOD:        "MOD", //无符号取模运算
	SMOD:       "SMOD", //有符号取模运算
	EXP:        "EXP",  //指数运算
	NOT:        "NOT",
	
	//从栈顶弹出两个元素，进行比较，
	//然后把结果（1表示true，0表示false）推入栈顶。
	//其中LT和GT把弹出的元素解释为无符号整数进行比较，
	//SLT和SGT把弹出的元素解释为有符号数进行比较，EQ不关心符号
	LT:         "LT",  //无符号小于比较
	GT:         "GT", //无符号大于比较
	SLT:        "SLT", //有符号小于比较
	SGT:        "SGT", //有符号大于比较
	EQ:         "EQ",  // 等于比较
	
	//SZERO指令从栈顶弹出一个元素，判断它是否为0，如果是，则把1推入栈顶，否则把0推入栈顶
	ISZERO:     "ISZERO", //布尔取反
	
	//SIGNEXTEND指令从栈顶依次弹出k和x，并
	//把x解释为k+1（0 <= k <= 31）字节有符号整数，然
	//后把x符号扩展至32字节。比如x是二进制10000000，k是0，
	//则符号扩展之后，结果为二进制1111…10000000（共249个1）
	SIGNEXTEND: "SIGNEXTEND" //符号位扩展
```



### 位运算指令集

> *0x10*

```
	//AND、OR、XOR指令从栈顶弹出两个元素，进行按位运算，然后把结果推入栈顶
	AND:    "AND",
	OR:     "OR",
	XOR:    "XOR",
	
	//BYTE指令先后从栈顶弹出n和x，取x的第n个字节并推入栈顶。
	//由于EVM的字长是32个字节，所以n在[0, 31]区间内才有意义，
	//否则BYTE的运算结果就是0。另外，字节是从左到右数的，因此第0个字节占据字的最高位8个比特
	BYTE:   "BYTE", 
	
	//这三条指令都是先后从栈顶弹出两个数n和x，
	//其中x是要进行位移操作顶数，n是位移比特数，然后把结果推入栈顶
	SHL:    "SHL",
	//SHR和SAR的区别在于，前者执行逻辑右移（空缺补0），后者执行算术右移（空缺补符号位）
	SHR:    "SHR",
	SAR:    "SAR",
	
	ADDMOD: "ADDMOD",
	
	//MULMOD指令依次从栈顶弹出x、y、z三个数，
	//先计算x和y的乘积（不受溢出限制），再计算乘积和z的模，最后把结果推入栈顶
	//假定乘积不会溢出，那么MULMOD(x, y, z)等价于x * y % z
	MULMOD: "MULMOD",
```



### 加密指令集

> *0x20*

```
SHA3: "SHA3"
```



### 关闭状态指令集

> *0x30*

```
   ADDRESS:        "ADDRESS",
	BALANCE:        "BALANCE",
	ORIGIN:         "ORIGIN",
	CALLER:         "CALLER",
	CALLVALUE:      "CALLVALUE",
	CALLDATALOAD:   "CALLDATALOAD",
	CALLDATASIZE:   "CALLDATASIZE",
	CALLDATACOPY:   "CALLDATACOPY",
	CODESIZE:       "CODESIZE",
	CODECOPY:       "CODECOPY",
	GASPRICE:       "GASPRICE",
	EXTCODESIZE:    "EXTCODESIZE",
	EXTCODECOPY:    "EXTCODECOPY",
	RETURNDATASIZE: "RETURNDATASIZE",
	RETURNDATACOPY: "RETURNDATACOPY",
	EXTCODEHASH:    "EXTCODEHASH",
```



### 块操作指令集

>*0x40*

```
	BLOCKHASH:   "BLOCKHASH",
	COINBASE:    "COINBASE",
	TIMESTAMP:   "TIMESTAMP",
	NUMBER:      "NUMBER",
	DIFFICULTY:  "DIFFICULTY",
	GASLIMIT:    "GASLIMIT",
	CHAINID:     "CHAINID",
	SELFBALANCE: "SELFBALANCE"
```



### 存储和执行指令集

> *0x50*

```
	POP: "POP",  // 栈顶弹出元素
	MLOAD:    "MLOAD",
	MSTORE:   "MSTORE",
	MSTORE8:  "MSTORE8",
	SLOAD:    "SLOAD", //先取出栈顶元素x，然后在storage中取以x为键的值（storage[x]）存入栈顶
	SSTORE:   "SSTORE", //存储storage是一个键值存储，可将256位字映射到256位字
	JUMP:     "JUMP",
	JUMPI:    "JUMPI",
	PC:       "PC",
	MSIZE:    "MSIZE",
	GAS:      "GAS",
	JUMPDEST: "JUMPDEST"
```



### Push指令集

> *0x60*

```
	// PUSH系列指令把紧跟在指令后面的N（1 ～ 32）字节元素推入栈顶
	PUSH1:  "PUSH1",
	...
	PUSH32: "PUSH32",

    //DUP系列指令复制从栈顶开始数的第N（1 ～ 16）个元素，并把复制后的元素推入栈顶
	DUP1:  "DUP1",
	DUP2:  "DUP2",
	...
	DUP16: "DUP16",

	//SWAP系列指令把栈顶元素和从栈顶开始数的第N（1 ～ 16）+ 1 个元素进行交换。
	SWAP1:  "SWAP1",
	...
	SWAP16: "SWAP16",
	
	LOG0:   "LOG0",
	...
	LOG4:   "LOG4",
```



## 参考

> https://mindcarver.cn
>
> https://github.com/blockchainGuide
>



