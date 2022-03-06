# 优化你的 Solidity 代码

## 优化技巧

### 打包你的变量

在以太坊中，你将为你使用的每个 storage slot（存储槽）支付 gas。一个 slot 是 256 位，你可以将许多变量打包到同一个 slot 中。打包由 Solidity 编译器与优化器自动完成，你只需要连续声明可打包的变量。

下面糟糕的示例代码将花费 3 个 slot：

```solidity
uint8 numberOne;
uint256 bigNumber;
uint8 numberTwo;
```

更好的方式是：

```solidity
uint8 numberOne;
uint8 numberTwo;
uint256 bigNumber;
```

 这个小的改动将为你节省大量的 gas，它只需要 2 个 slot 存储变量。另外需要注意，`struct`、`mapping`、`array` 总是会从一个新的 slot 开始。

### uint8 不是总比 uint256 便宜

EVM 一次只对 32 字节 / 256 位操作。这意味着如果你使用 `uint8`，EVM 会先把它转换为 `uint256`，转换操作会花费额外的 gas。如果你要定义单个变量并且不能打包，最好的办法就是使用 `uint256` 而不是 `uint8`。

### Mapping 往往比 Array 更便宜

由于 EVM 的工作方式，`Array` 并不是连续的存储在内存中，而是跟 `Mapping` 一样。你可以打包 `Array`，而 `Mapping` 不能打包。因此，如果你使用像 `uint8` 一样能打包的较小的元素，使用 `Array` 会更便宜。你不能够获取 `Mapping` 的长度或获取它的所有元素。所以，取决于你的用例，你可能只能使用 `Array`，即便它会花费更多 gas。

### 不是所有的元素都可以打包

关键字 `memory` 或 `calldata` 标记的变量不能打包。在函数调用与内存中使用较小的变量不会节省 gas。

### 使用 bytes32 而不是 string/bytes

如果你能把你的数据放进 32 字节，那么你应该使用 `bytes32` 数据类型而不是 `string` 或 `bytes`。因为它在 Solidity 中更便宜。通常，任何固定大小的变量都比可变大小的变量更便宜。

### 做更少的合约调用

每个合约调用会花费大量的 gas。为了优化，可以将你需要的**所有数据**通过**一次合约调用**获取到，而不是分多个合约调用。这跟许多其他语言的优化不太一样，Solidity 是特例。

### 使用 external 函数修饰符

对于所有 `public` 修饰的函数，输入参数会自动的拷贝到内存中，这会消耗 gas。如果你的函数只会在外部调用，那么你应该应该把它标记为 `external`。`external` 函数不会拷贝入参到内存，能从 `calldata` 直接读取。当入参较大的时候，做这个修改会节省很多 gas。

### 删除你不需要的变量

在以太坊中释放存储空间会退回一部分的 gas。如果你不再需要使用这个变量，那么你应该用 `delete` 关键字或将其值置为默认值来删除它。这将帮助区块链的大小更小。

### 充分利用“短路规则”

当使用逻辑或（||）和逻辑与（&&）的时候，如果逻辑或的第一个函数被解析为 true，或逻辑与的第一个函数被解析为 false，那么就不会执行第二个函数，这就会为你节省第二个函数的执行成本。你需要根据函数执行的概率，进行排序，以减少函数执行。

### 减少更改存储数据

**改变存储数据的成本比修改内存或栈数据的成本更高**。所以你应该在计算完后，再更新存储数据，而不是实时更新。下面的示例代码演示了这个过程：

```solidity
contract Demo
{
    uint internal counter;
 
    // The below function updates the storage counter every time
    // This is a bad coding practice and should be avoided
    // as updating a storage variable is expensive
    function badFunction(){
        for (uint i = 0; i < 100; i++){
            counter++;
        }
    }
	
    // This function uses a stack variable, j for calculations
    // and updates the storage variable at the last.
    // it's cheaper as updating a stack variable is almost free
    function betterfunction(){
        uint j;
        for (uint i = 0; i < 100; i++){
            j++;
        }
        counter = j;
    }
    // I know, you don't need a loop for this :/
}
```

### 最小化链上数据

**当你设计一个 Dapp 时，你需要决定将哪些代码/数据放在链上和链下。你在链上投入的越少，你的 gas 成本就越少。**

那么如何决定将什么上链呢？对你的 Dapp 至关重要的代码和数据通常必须上链：游戏的经济性、学校在区块链成绩单系统中的签名等……通常，这将是你所有代码/数据的一小部分. 可以脱链的是元数据、文件、设置和所有非关键部分。

### 开启 Solidity 编译优化

当你编译 Solidity 智能合约时，你可以指定一个优化标志来告诉 Solidity 编译器生成高度优化的字节码。

优化器试图简化复杂的表达式，从而减少代码大小和执行成本，即，它可以减少合约部署以及对合约的外部调用所需的 gas。

为什么我们需要开启这个优化？不能只是 Solidity 的正常运行模式吗？嗯，它存在的原因是因为这种优化比正常编译需要更长的时间来运行。在开发中，这种较长的编译时间可能会很烦人，因此默认情况下它是关闭的。

如果您使用 Truffle，您可以在 `truffle-config.js` 文件中使用此设置启用优化：

```javascript
module.exports = {
...
  solc: {
    optimizer: {
      enabled: true,
      runs: 200
    }
  }
}
```

运行次数 ( `runs`) 大致指定了部署代码的每个操作码在合约生命周期内执行的频率。这意味着它是代码大小（部署成本）和代码执行成本（部署后的成本）之间的权衡参数。`runs` 为 1将产生简短但昂贵的代码。相反，较大的 `runs` 参数将生成更长但更省 gas 的代码。`runs` 的最大值为 `2**32-1` 。

### 写字面值而不是计算值

如果你知道在编译时已经知道一些数据的值，直接写这个值。不要在编译时使用 Solidity 函数来导出数据的值。这样做可能不太方便，但可以避免浪费气体。

```solidity
//Good
bytes32 constant hash = 'uiHk78Uidaf....';

//Bad
bytes32 constant hash = keccack256(abi.encodePacked('MyDataToHash'));
```

### 使用 Proxy 合约对同一个合约进行多次部署

合约部署的成本是很高的。如果你需要对同一份代码进行多次部署的话，这个开销非常庞大。通过使用 Proxy 合约，这个合约实际使用 `delegate_call` 去调用实现合约，执行的上下文环境在 proxy 合约中，但使用的是实现合约的代码。因此将实现合约部署一次，proxy 合约多次部署，即可达到一样的效果。但由于 proxy 合约部署成本比实现合约更低，从而节省了 gas。

### 使用汇编代码

当你编译一个 Solidity 智能合约时，它会被转换成一系列 EVM（以太坊虚拟机）操作码。我们称之为字节码。Solidity 通常在编写优化的字节码方面做得很好。但有时，您可以通过自己编写字节码来比 Solidity 做得更好。

实际上，没有人直接编写 EVM 字节码。但是你可以做的是使用 [Yul](https://docs.soliditylang.org/en/latest/assembly.html)，一种低级智能合约语言。使用汇编，您可以编写非常接近操作码级别的代码。写这么低级别的代码不是很容易，但好处是可以手动优化操作码，在某些情况下优于 Solidity 字节码。

## 参考

[1] [Solidity gas optimization tips](https://mudit.blog/solidity-gas-optimization-tips/)

[2] [How to optimize gas cost in a Solidity smart contract? 6 tips](https://eattheblocks.com/how-to-optimize-gas-cost-in-a-solidity-smart-contract-6-tips/)

[3] [Solidity: The Optimizer](https://docs.soliditylang.org/en/v0.8.11/internals/optimizer.html)