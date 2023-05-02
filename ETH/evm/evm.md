# EVM

> 部分内容直接摘抄自 Ethereum 官方文档

EVM的物理实例不能用云或海浪的方式来描述，但它确实作为一个单一实体存在，由数千台运行以太坊客户端的连接计算机维护。

以太坊协议本身的存在仅仅是为了保持这种特殊状态机的连续、不间断和不可变的运行。这是所有以太坊账户和智能合约所处的环境。在链中的任何给定块中，以太坊都有且只有一个“规范”状态，EVM定义了从块到块计算新有效状态的规则。

## From Ledger to  State Machine

“分布式账本”的比喻经常被用来描述比特币等区块链，它使用基本的密码学工具来实现分散的货币。分类帐维护着活动的记录，这些活动必须遵守一套规则，这些规则规定了某人可以做什么，不可以做什么来修改分类帐。例如，一个比特币地址不能花费比之前收到的更多的比特币。这些规则支撑着比特币和许多其他区块链上的所有交易。

虽然以太坊拥有自己的原生加密货币(Ether)，遵循几乎完全相同的直观规则，但它还支持更强大的功能:智能合约。对于这个更复杂的特性，需要一个更复杂的类比。以太坊不是分布式账本，而是分布式状态机。以太坊的状态是一个大型数据结构，不仅包含所有账户和余额，还包含机器状态，可以根据预定义的规则集从块更改到块，并且可以执行任意机器代码。EVM定义了在区块之间改变状态的具体规则。

![A diagram showing the make up of the EVM](https://ethereum.org/static/e8aca8381c7b3b40c44bf8882d4ab930/302a4/evm.png)

## THE ETHEREUM STATE TRANSITION FUNCTION

EVM的行为就像一个数学函数:给定一个输入，它产生一个确定性的输出。因此，更正式地将以太坊描述为具有状态转换函数是很有帮助的:

```
Y(S, T)= S'
```

给定一个旧的有效状态(S)和一组新的有效交易(T)，以太坊状态转换函数Y(S, T)产生一个新的有效输出状态S'。

## State

在以太坊的背景下，状态是一个巨大的数据结构，称为修改的Merkle Patricia Trie，它将所有帐户通过哈希连接起来，并可简化为存储在区块链上的单个根哈希。

## Trasactions

交易是来自账户的经过加密签名的指令。有两种类型的交易:导致消息调用的交易和导致合同创建的交易。合约创建导致创建一个包含编译后的智能合约字节码的新合约账户。每当另一个帐户对该合约进行消息调用时，它都会执行其字节码。

## EVM INSTRUCTIONS

EVM作为堆栈机执行，深度为1024项。每个项是一个256位字，选择它是为了方便256位加密(例如Keccak-256哈希或secp256k1签名)。

在执行期间，EVM维护一个临时内存(作为字寻址字节数组，word-addressed byte array)，它不会在事务之间持久存在。

然而，合约确实包含一个Merkle Patricia存储树(作为一个单词可寻址的单词数组，word-addressable word array)，它与所讨论的帐户和全局状态的一部分相关联。

编译后的智能合约字节码作为多个EVM操作码执行，这些操作码执行标准的堆栈操作，如XOR, AND, ADD, SUB等。EVM还实现了一些区块链特定的堆栈操作，如ADDRESS、BALANCE、BLOCKHASH等。

![A diagram showing where gas is needed for EVM operations](https://ethereum.org/static/9628ab90bfd02f64cf873446cbdc6c70/302a4/gas.png)

## EVM IMPLEMENTATIONS

EVM的所有实现都必须遵守以太坊黄文件中描述的规范。在以太坊9年的历史中，EVM经历了几次修订，EVM在各种编程语言中有几种实现。以太坊执行客户端（[Ethereum execution clients](https://ethereum.org/en/developers/docs/nodes-and-clients/#execution-clients)）包括一个EVM实现。此外，还有多个独立实现，包括:

- [Py-EVM↗](https://github.com/ethereum/py-evm) - *Python*
- [evmone↗](https://github.com/ethereum/evmone) - *C++*
- [ethereumjs-vm↗](https://github.com/ethereumjs/ethereumjs-vm) - *JavaScript*
- [eEVM↗](https://github.com/microsoft/eevm) - *C++*



## evm 的方法

**call 函数簇在智能合约上，体现为操作码**。call 函数簇的执行上下文、caller（msg.Sender）等信息以表格形式体现：

定义如下：

U 为外部拥有账户（ External Owned Account，EOA）

C 为合约（Contract），Cx 为合约 x

调用链为 U -> C0 -> C1 -> ... -> Cn

[...] 这里面的内容不是主要场景，但为了理解，需要补充出来的部分。

| Op               | Scenario                                             | Context that the code runs | Storage | msgSender                        | code |
| ---------------- | ---------------------------------------------------- | -------------------------- | ------- | -------------------------------- | ---- |
| evm.Call         | C0 does CALL on C1                                   | C1                         | C1      | C0                               | C1   |
| evm.CallCode     | C0 does CALL_CODE on C1                              | C0                         | C0      | C0                               | C1   |
| evm.StaticCall   |                                                      |                            |         |                                  |      |
| evm.DelegateCall | [U does CALL on C0, and] C0 does DELEGATE_CALL on C1 | C0                         | C0      | U is the `msg.sender`  inside C1 | C1   |



### evm.Call

作用：将给定的 input 作为函数入参，执行指定 addr 的合约。它还处理所需的任何必要的值转移（ ETH 的转移），并采取必要的步骤来创建帐户，并在执行错误或值转移失败时 revert。如果目标地址是预编译合约，则执行预编译合约；如果没有合约代码，则什么也不执行；否则，由[解释器解析](##解释器)执行合约代码。

### evm.CallCode

作用：与 evm.Call 的不同处在于，合约执行上下文和caller ，见上表。



## evm.DelegateCall

作用：与 evm.Call 的不同处在于，合约执行上下文和caller ，见上表。



## evm.StaticCall

作用：与 evm.Call 的不同处在于，它只读，任何的写操作会导致异常。



## 解释器

解释器会根据 pc（program counter，程序计数器）从合约代码中获取当前的操作码（OpCode），再从 JumpTable（跳转表，或者叫指令表）中获得操作码对应的操作（Operation）。

JumpTable 支持 256 种操作码，每个操作码是 0～255 的整型。

| OpCode | Number | Operation                                          |
| ------ | ------ | -------------------------------------------------- |
| STOP   | 0      | 停止合约执行                                       |
| ADD    | 1      | [a, b, ...]，a 与 b 出栈后相加，将结果 push 到栈顶 |
| MUL    | 2      | [a, b, ...]，a 与 b 出栈后相乘，将结果 push 到栈顶 |
| ...    |        |                                                    |



## FURTHER READING

- [Ethereum Yellowpaper(opens in a new tab)↗](https://ethereum.github.io/yellowpaper/paper.pdf)
- [Jellopaper aka KEVM: Semantics of EVM in K(opens in a new tab)↗](https://jellopaper.org/)
- [The Beigepaper(opens in a new tab)↗](https://github.com/chronaeon/beigepaper)
- [Ethereum Virtual Machine Opcodes(opens in a new tab)↗](https://www.ethervm.io/)
- [A short introduction in Solidity's documentation(opens in a new tab)↗](https://docs.soliditylang.org/en/latest/introduction-to-smart-contracts.html#index-6)
