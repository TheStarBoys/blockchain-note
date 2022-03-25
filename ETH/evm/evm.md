# EVM

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

