# 操作码

[a, b, c]，其中 [] 表示栈，a，b，c 表示栈内元素，a 为栈顶元素，c 为栈底元素。

本文档不会列出全部 opcode，只是**列出合约开发需要掌握的重点**，如果对全部内容感兴趣，可以参考[此处](https://www.ethervm.io/)。

## Create

栈： [value, offset, size, ...]

操作：从 memory[offset, offset+size] 获得合约部署代码，转移 value 数值的 ETH 到子合约中，计算出子合约地址，将地址 push 到栈中。

返回值：[contract_addr, ...] （子合约地址）

作用：将 value 作为部署代码，创建合约。子合约地址由 caller 地址和其 nonce 计算得到。



## Create2

栈： [value, offset, size, salt, ...]

返回值：[contract_addr, ...] （子合约地址）

作用：与 Create 的区别是：子合约地址由 caller 地址、salt 以及部署代码的哈希值计算得到。



## Call

栈：[tmp(之前的版本是gas), addr, value, inOffset, inSize, retOffset, retSize, ...]

参数解析：addr 为目标合约地址。

操作：将 tmp 出栈，gas 在 interpreter.evm.callGasTemp 中拿到，依次将 addr, value, inOffset, inSize, retOffset, retSize 值出栈。从 memory[inOffset, inOffset + inSize] 拿到调用参数，调用 Call（这里可能存在递归），根据执行结果，成功则将 tmp 赋值为 1（true），否则为 0（false）。将 tmp 放入栈中。

返回值：[success, ...]

作用：调用其他合约的方法



## Callcode



## DelegateCall



## Static Call

