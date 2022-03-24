# 以太坊架构概览

## 概览

![img](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/afWDt.jpg)

## 账户模型

## 以太坊的数据结构

## 挖矿过程

如果收到当前正在挖掘的区块已经被同伴打包，经验证为有效区块，则停止挖矿，自动开始挖下一个区块。

1. 矿工开始挖矿
2. 矿工搜索并包含有效的叔块（得到 Ommers 列表并计算出 OmmersHash）
   1. 包含最多两个叔块
   2. 优先添加本地的叔块
3. 矿工从 mempool 挑选要打包进区块的交易，并放入 Transaction Trie
   1. 从 TxPool 中取出所有 pending 交易，并将其分为本地的与远程的交易，优先包含本地的交易
   2. 根据 gas price 和 nonce 进行排序
4. 处理所有交易，计算出最终状态（得到状态根：World State Trie Root、Transaction Receipt Trie Root）
   1. 针对当前状态进行快照
   2. 根据顺序处理当前交易
      1. 检查
         1. nonce 必须等于当前状态期望的 nonce 值
         2. 交易发送者必须是外部拥有账户（EOA）
         3. 余额够转账
      2. EVM 处理交易
         1. 如果 msg.to 是零地址，则进行合约创建
         2. 否则
            1. 解释器解释交易 input 中的每个指令
            2. 根据指令在指令集中相应的操作进行处理
      3. EVM处理后
         1. 如果发生错误 ，则回滚到快照时的状态
         2. 将处理后的结果形成 Transaction Receipt
      4. 得到当前状态
5. 敲定并组装区块（consensusEngine.FinalizeAndAssemble，进行任何交易处理后的状态修改，例如出块奖励）
6. 封装区块（consensusEngine.Seal）
7. 将区块和状态写入本地区块链中
8. 广播区块并将其插入未确认的区块集合中等待确认

## 交易顺序

单个账户的交易顺序由 nonce 值决定，nonce 小的交易先执行。对于交易池中所有的 pending 交易，优先考虑矿工费给的最高的交易，如果矿工费相同，就考虑先看到的交易。这个是采用的堆实现的，core/types/transactions.go 中的 TxByPriceAndTime 结构实现了 go 标准库的 heap 接口。

## mempool（又称txpool）

mempool 中有 pending、queue 交易，pending 交易是当前可以执行的交易，queue 交易是当前不能执行的交易。

## EVM

### 概览

1. 如果 msg.to 是零地址，则进行合约创建
   1. 根据钱包地址和其 nonce 计算合约地址
   2. 创建合约
2. 否则
   1. 增加交易发送者的 nonce 值后，进行 evm.call
   2. 如果当前调用栈过深（超过1024）则报错
   3. 确保当前执行上下文的 caller 有足够余额转账
   4. 进行快照
   5. 进行转账
   6. 检查是否是预编译合约
      1. 是，执行预编译合约
      2. 否
         1. 获取合约的代码
         2. 解释器解释
            1. 增加调用栈深度
            2. 清空返回值数据，设置是否只读
            3. 交易 input 中的每个指令
            4. 根据指令在指令集中相应的操作进行处理
            5. 减少调用栈深度，只读标志设置为 false



### 解释器（Interpreter）工作原理

解释器主要有 pc（program counter，程序计数器）、stack（栈）、memory（内存）结构。

1. 根据 pc 从合约的字节数组中获取对应的字节（OpCode，操作码）
2. 根据 OpCode 在 JumpTable 中查询对应的操作（Operation，Op）
3. 检查当前栈的长度是否满足操作的要求，太大或太小都会报错
4. 如果当前模式为只读，但操作为写操作，则报错 “写保护”
5. 根据操作的固定gas成本，扣除合约剩余的 gas，如果不够，则“Out of Gas”
6. 检查内存大小
7. 根据操作的动态gas成本，扣除合约剩余的 gas，如果不够，则“Out of Gas”
8. 分配内存以适配操作
9. 执行操作
10. 如果该操作需要记录返回值，则记录到解释器中
11. 检查是否执行结果
    1. 如果操作需要回滚，则返回 ErrExecutionReverted
    2. 如果操作需要中止，则直接返回结果
    3. 如果操作不需要跳过 pc，则增加 pc 的值
12. 如果 gas 有剩余且没有其他需要退出执行的条件，则从第一步循环执行



### 操作码

在[这里](https://ethereum.stackexchange.com/questions/268/ethereum-block-architecture/6413#6413)可以看到全部操作码。我做出的重点讲解可以参考[此处](./evm/opcode.md)。



## P2P

## 参考

- [StackOverflow：Ethereum Block Architecture](https://ethereum.stackexchange.com/questions/268/ethereum-block-architecture/6413#6413)