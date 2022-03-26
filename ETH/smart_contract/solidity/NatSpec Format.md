# NatSpec 注释

> 本文直接翻译自 [solidity 官方文档](https://docs.soliditylang.org/en/latest/natspec-format.html)，如有不恰当出欢迎指出。

Solidity 合约可以使用一种特殊形式的注释来为函数、返回变量等提供丰富的文档。这种特殊形式被命名为以太坊自然语言规范格式（NatSpec）。

> 注意：
>
> NatSpec 的灵感来自[Doxygen](https://en.wikipedia.org/wiki/Doxygen)。虽然它使用 Doxygen 样式的注释和标签，但无意保持与 Doxygen 的严格兼容性。请仔细检查下面列出的支持标签。

本文档分为以开发人员为中心的消息和面向最终用户的消息。这些消息可能会在最终用户（人）与合同交互（即签署交易）时显示给他们。

建议对所有公共接口（ABI 中的所有内容）使用 NatSpec 对 Solidity 合约进行完全注释。

NatSpec 包括智能合约作者将使用的注释格式，Solidity 编译器可以理解这些格式。下面还详细介绍了 Solidity 编译器的输出，它将这些注释提取为机器可读的格式。

NatSpec 还可能包含第三方工具使用的注释。这些很可能是通过`@custom:<name>`标签完成的，一个很好的用例是分析和验证工具。

## 文档示例

在 `contract` ， `interface` ， `function` ，和 `event` 上面，使用 Doxygen 注释格式插入文档。对 NatSpec 而言，一个 `pulic` 的状态变量等同于一个函数 `function` 。

- 对于 Solidity 来说，你可以将 `///` 用于单行或多行注释，或者使用 `/** */` 用于多行注释。
- 对于 Vyper 来说，使用 `"""` 进行注释。

以下示例显示了使用所有可用标签的合约和函数。

> 注意：
>
> Solidity 编译器只解释外部或公共的标签。欢迎您为您的内部和私有函数使用类似的注释，但这些注释不会被解析。
>
> 这在未来可能会改变。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.2 < 0.9.0;

/// @title A simulator for trees
/// @author Larry A. Gardner
/// @notice You can use this contract for only the most basic simulation
/// @dev All function calls are currently implemented without side effects
/// @custom:experimental This is an experimental contract.
contract Tree {
    /// @notice Calculate tree age in years, rounded up, for live trees
    /// @dev The Alexandr N. Tetearing algorithm could increase precision
    /// @param rings The number of rings from dendrochronological sample
    /// @return Age in years, rounded up for partial years
    function age(uint256 rings) external virtual pure returns (uint256) {
        return rings + 1;
    }

    /// @notice Returns the amount of leaves the tree has.
    /// @dev Returns only a fixed number.
    function leaves() external virtual pure returns(uint256) {
        return 2;
    }
}

contract Plant {
    function leaves() external virtual pure returns(uint256) {
        return 3;
    }
}

contract KumquatTree is Tree, Plant {
    function age(uint256 rings) external override pure returns (uint256) {
        return rings + 2;
    }

    /// Return the amount of leaves that this specific kind of tree has
    /// @inheritdoc Tree
    function leaves() external override(Tree, Plant) pure returns(uint256) {
        return 3;
    }
}
```



## 标签

所有标签都是可选的。下表解释了每个 NatSpec 标签的用途以及可以使用的位置。作为一种特殊情况，如果没有使用标签，那么 Solidity 编译器将解释 `///`或 `/**` 注释，就像它被 `@notice` 标记了一样。

| 标签          |                                                     | 语境                                     |
| ------------- | --------------------------------------------------- | ---------------------------------------- |
| `@title`      | 应描述合约/接口的标题                               | 合约、库、接口                           |
| `@author`     | 作者姓名                                            | 合约、库、接口                           |
| `@notice`     | 向最终用户解释这是做什么的                          | 合约、库、接口、函数、公共状态变量、事件 |
| `@dev`        | 向开发人员解释任何额外的细节                        | 合约、库、接口、函数、状态变量、事件     |
| `@param`      | 像在 Doxygen 中一样记录一个参数（必须后跟参数名称） | 函数，事件                               |
| `@return`     | 记录合约函数的返回变量                              | 函数，公共状态变量                       |
| `@inheritdoc` | 从基本函数中复制所有缺失的标签（必须后跟合约名称）  | 函数，公共状态变量                       |
| `@custom:...` | 自定义标签，语义由应用程序定义                      | 任意位置                                 |

如果您的函数返回多个值 `(int quotient, int remainder)`， 则使用与 `@param` 语句格式相同的多个 `@return` 语句。

自定义标签以 `@custom:` 开头，并且必须后跟一个或多个小写字母或连字符（\-）。但是，它不能以连字符开头。它们可以在任何地方使用，并且是开发人员文档的一部分。

## 动态表达式

如本指南中所述，Solidity 编译器将通过 NatSpec 文档从您的 Solidity 源代码传递到 JSON 输出。此 JSON 输出的消费者，例如最终用户客户端软件，可能会直接将其呈现给最终用户，或者它可能会应用一些预处理。

例如，一些客户端软件会呈现：

```
/// @notice This function will multiply `a` by 7
```

如果一个函数被调用并且输入 `a` 被赋值为 10，最终用户将看到：

```
This function will multiply 10 by 7
```



指定这些动态表达式超出了 Solidity 文档的范围，您可以在 [radspec 项目](https://github.com/aragon/radspec)中阅读更多内容。

## 继承说明

没有 NatSpec 的函数将自动继承其基本函数的文档。例外情况是：

- 当参数名称不同时。
- 当有多个基本功能时。
- 当有一个明确的`@inheritdoc`标签指定应该使用哪个合约来继承时。



## 文档输出

当被编译器解析时，上面示例中的文档将生成两个不同的 JSON 文件。一个是供最终用户在执行功能时作为通知使用的，另一个是供开发人员使用的。

如果将上述合同另存为 `ex1.sol`，则您可以使用以下命令生成文档：

```
solc --userdoc --devdoc ex1.sol
```

输出如下。



> 注意
>
> 从 Solidity 版本 0.6.11 开始，NatSpec 输出还包含一个`version`和一个`kind`字段。当前`version`设置为`1`并且`kind`必须是`user`或之一`dev`。将来可能会引入新版本，弃用旧版本。



### 用户文档

上述文档将生成以下用户文档 JSON 文件作为输出：



```json
{
  "version" : 1,
  "kind" : "user",
  "methods" :
  {
    "age(uint256)" :
    {
      "notice" : "Calculate tree age in years, rounded up, for live trees"
    }
  },
  "notice" : "You can use this contract for only the most basic simulation"
}
```

请注意，查找方法的关键是[合约 ABI](https://docs.soliditylang.org/en/latest/abi-spec.html#abi-function-selector)中定义的函数的规范签名，而不仅仅是函数的名称。

### 开发者文档

除了用户文档文件外，还应生成开发人员文档 JSON 文件，该文件应如下所示：



```json
{
  "version" : 1,
  "kind" : "dev",
  "author" : "Larry A. Gardner",
  "details" : "All function calls are currently implemented without side effects",
  "custom:experimental" : "This is an experimental contract.",
  "methods" :
  {
    "age(uint256)" :
    {
      "details" : "The Alexandr N. Tetearing algorithm could increase precision",
      "params" :
      {
        "rings" : "The number of rings from dendrochronological sample"
      },
      "return" : "age in years, rounded up for partial years"
    }
  },
  "title" : "A simulator for trees"
}
```