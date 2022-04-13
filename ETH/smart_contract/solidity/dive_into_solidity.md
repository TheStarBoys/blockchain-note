# 深入 Solidity

> 文中大部分是从 solidity 官方文档中直接翻译的，如果有不准确的地方，可以直接看下方给出的原文链接。

## 存储中状态变量的布局

### 状态变量

合约的状态变量以紧凑的方式存储在存储（storage）中，它们有时会共享同一个存储槽（storage slot），动态数组和映射除外（详情见下文）。对于每个变量，根据它的类型，确定它需要占用多少字节。少于 32 字节的多个、连续的项，如果可能的话，会打包进同一个存储槽。规则如下：

- 存储槽的第一个项以低顺序对齐（lower-order aligned）存储
- 值类型只使用存储它们必要的字节数
- 如果存储槽的剩余部分无法放入该值类型，就把它存储到下一个存储槽中
- 结构体和数组数据总是以新的存储槽开始，并且它们严格按照这些规则打包
- 紧跟结构体或数组后面的项总是以新的存储槽开始



对于使用继承的合约，状态变量的顺序由没有任何其他合约依赖的合约开始的 C3 线性顺序（C3-linearized order）决定。如果上述规则允许的话，不同合约的状态变量共享同一个存储槽。

结构体和数组的元素依次存储，就好像它们是独立的值一样。

> 注意：
>
> 常量不是状态变量，无法对其使用 `.slot` 和 `.offset` 。

### 映射和动态数组

由于它们的大小无法预测，映射和动态数组不能存储在它们上下的状态变量中间。相反，它们根据上述规则占据 32 字节，它们存储的元素从一个由 `Keccak-256` 计算出的新的存储槽开始。

假设，在使用存储布局规则后，映射或动态数组的存储位置最终是存储槽 `p`。对于动态数组，这个存储槽存储数组中的元素个数（byte 数组和字符串除外）。对于映射，这个存储槽一直是空的，但它仍然需要确保，即便有两个相邻的映射，它们的内容最终在不同的存储槽位置。

#### 数组

数组数据在 `keccak256(p)` 存储槽开始，并且以与定长数组相同的方式排列：一个元素挨一个元素，如果元素长度不超过 16 字节，可能共享同一个存储槽。动态数组的动态数组递归地应用此规则。

元素 `x[i]` 的位置（`x` 的类型为 `uint256[]`），计算方法如下（假定 `x` 本身存储在槽 `p`）：

Slot = `keccak256(p)`

the data of elemt = `keccak256(p) + i`



元素 `x[i][j]` 的位置（`x`的类型为 `uint24[][]` ）计算方法如下（假定 `x` 本身存储在槽 `p`）：

Slot = `keccak256(keccak256(p) + i) + floor(j / floor(256 / 24))`

the data of element = `(v >> ((j % floor(256 / 24)) * 24)) & type(uint24).max` 其中 `v` 是存储槽中的数据。



#### 映射

映射的 key `k` 对应存储的数据的位置计算方式： `keccak256(h(k) . p)`，其中 `.` 是连接符，`h` 是一个函数，它根据 `k` 的类型采用不同的手段：

- 对于值类型，`h` ，与在内存中存储值时相同的方式，把值拉长为 32 字节。
- 对于字符串和字节数组，`h` 计算未拉长数据的 `keccake256` hash。

如果映射的值不是值类型，计算出的存储槽标记为数据的开始。如果值为结构体类型，例如，你必须为每个结构体成员添加相应的偏移量（offset）。

考虑如下合约代码：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.0 <0.9.0;


contract C {
    struct S { uint16 a; uint16 b; uint256 c; }
    uint x;
    mapping(uint => mapping(uint => S)) data;
}
```

我们来计算 `data[4][9].c` 的存储位置。映射自己的存储槽位置为 `1` （因为 `x` 占用了完整的 32 字节）。这意味着 `data[4]` 存储在 `keccak256(uint256(4) . uint256(1))` 。`data[4]` 的类型还是一个映射，并且它的数据 `data[4][9]` 存储在 `keccak256(uint256(9) . keccak256(uint256(4) . uint256(1)))`。结构体 `S` 里的成员 `c` 的存储槽偏移量为 `1`，因为 `a` 和 `b` 打包进了同一个存储槽。这意味着 `data[4][9].c` 的存储槽是 `keccak256(uint256(9) . keccak256(uint256(4) . uint256(1))) + 1` 。其值的类型是 `uint256`，所以它使用一个完整的存储槽。



### bytes 和字符串

`bytes`、 `string` 的编码方式是相同的。通常，这个编码与 `bytes1[]` 类似，感觉上会有一个数组自己的存储槽，并且数据区域由它们位置的 `keccak256` hash 计算得到。然而，对于比 32 字节短的值，数组元素会和长度一起存储在同一个存储槽中。

特别地，如果数据长度不超过 31 字节，它的元素存储在高阶字节（higher-order）中（左对齐，left aligned），并且低阶字节存储 `length * 2` 。对于大于等于 32 字节长度的字节数组，存储槽 `p` 存储 `length * 2 + 1` ，并且与往常一样根据 `keccak256(p)` 存储数据。这意味着你能区分短的数组和长的数组，通过检查它的最低 bit 是否被设置：短数组不会设置，长数组会。



### JSON 输出

合约的存储布局能够通过标准 [JSON 接口](https://docs.soliditylang.org/en/v0.8.11/using-the-compiler.html#compiler-api)获取。这个输出是一个包含两个 key （`storage` 和 `types`）的 JSON 对象。其中 `storage` 对象是一个数组，它的元素格式如下：

```json
{
    "astId": 2,
    "contract": "fileA:A",
    "label": "x",
    "offset": 0,
    "slot": "0",
    "type": "t_uint256"
}
```

上面的示例是来自文件 `fileA` 的 `contract A { uint x; }` 的存储布局，其中：

- `astId` 是状态变量声明的 AST 节点 id
- `contract` 是包含它的路径作为前缀的合约的名字
- `label` 状态变量的名字
- `offset` 是根据编码在存储槽中的字节形式的偏移量
- `slot` 是变量存储和开始的存储槽。这个值可能很大，所以它的 JSON 值表示为一个字符串
- `type` 是一个标识符，用于该变量类型信息的一个 key

给定 `type` ，`t_uint256` 表示为 `types` 中的一个元素，它的格式为：

```json
{
    "encoding": "inplace",
    "label": "uint256",
    "numberOfBytes": "32",
}
```

其中：

- `encoding` 是数据在存储中编码的方式，其可能的值为:
  - `inplace`: 数据在存储中连续排列（见[上文](##状态变量)）。
  - `mapping`: 基于 `Keccak-256` hash  (见[上文](##映射和动态数组))。
  - `dynamic_array`: 基于 `Keccak-256` hash  (见[上文](##映射和动态数组))。
  - `bytes`: 单个存储槽或者基于 `Keccak-256` hash  (见[上文](##bytes 和字符串)).
- `label` 是规范的类型名。
- `numberOfBytes` 是使用的字节数量（以十进制字符串表示）。注意，如果 `numberOfBytes > 32` 意味着它使用了一个以上的存储槽。

有些类型除了上面四种之外还有额外的信息。映射包含它的 `key` 和 `value` 类型（再次引用类型映射中的条目），数组有它的 `base` 类型，结构体会列出它的字段。

> 合同存储布局的JSON输出格式仍然被认为是实验性的，在solidality的非破坏版本中会有所改变。



下面的示例显示合约及其存储布局，其中包含值和引用类型、编码打包的类型和嵌套类型。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.0 <0.9.0;
contract A {
    struct S {
        uint128 a;
        uint128 b;
        uint[2] staticArray;
        uint[] dynArray;
    }

    uint x;
    uint y;
    S s;
    address addr;
    mapping (uint => mapping (address => bool)) map;
    uint[] array;
    string s1;
    bytes b1;
}
```





```json
{
  "storage": [
    {
      "astId": 15,
      "contract": "fileA:A",
      "label": "x",
      "offset": 0,
      "slot": "0",
      "type": "t_uint256"
    },
    {
      "astId": 17,
      "contract": "fileA:A",
      "label": "y",
      "offset": 0,
      "slot": "1",
      "type": "t_uint256"
    },
    {
      "astId": 20,
      "contract": "fileA:A",
      "label": "s",
      "offset": 0,
      "slot": "2",
      "type": "t_struct(S)13_storage"
    },
    {
      "astId": 22,
      "contract": "fileA:A",
      "label": "addr",
      "offset": 0,
      "slot": "6",
      "type": "t_address"
    },
    {
      "astId": 28,
      "contract": "fileA:A",
      "label": "map",
      "offset": 0,
      "slot": "7",
      "type": "t_mapping(t_uint256,t_mapping(t_address,t_bool))"
    },
    {
      "astId": 31,
      "contract": "fileA:A",
      "label": "array",
      "offset": 0,
      "slot": "8",
      "type": "t_array(t_uint256)dyn_storage"
    },
    {
      "astId": 33,
      "contract": "fileA:A",
      "label": "s1",
      "offset": 0,
      "slot": "9",
      "type": "t_string_storage"
    },
    {
      "astId": 35,
      "contract": "fileA:A",
      "label": "b1",
      "offset": 0,
      "slot": "10",
      "type": "t_bytes_storage"
    }
  ],
  "types": {
    "t_address": {
      "encoding": "inplace",
      "label": "address",
      "numberOfBytes": "20"
    },
    "t_array(t_uint256)2_storage": {
      "base": "t_uint256",
      "encoding": "inplace",
      "label": "uint256[2]",
      "numberOfBytes": "64"
    },
    "t_array(t_uint256)dyn_storage": {
      "base": "t_uint256",
      "encoding": "dynamic_array",
      "label": "uint256[]",
      "numberOfBytes": "32"
    },
    "t_bool": {
      "encoding": "inplace",
      "label": "bool",
      "numberOfBytes": "1"
    },
    "t_bytes_storage": {
      "encoding": "bytes",
      "label": "bytes",
      "numberOfBytes": "32"
    },
    "t_mapping(t_address,t_bool)": {
      "encoding": "mapping",
      "key": "t_address",
      "label": "mapping(address => bool)",
      "numberOfBytes": "32",
      "value": "t_bool"
    },
    "t_mapping(t_uint256,t_mapping(t_address,t_bool))": {
      "encoding": "mapping",
      "key": "t_uint256",
      "label": "mapping(uint256 => mapping(address => bool))",
      "numberOfBytes": "32",
      "value": "t_mapping(t_address,t_bool)"
    },
    "t_string_storage": {
      "encoding": "bytes",
      "label": "string",
      "numberOfBytes": "32"
    },
    "t_struct(S)13_storage": {
      "encoding": "inplace",
      "label": "struct A.S",
      "members": [
        {
          "astId": 3,
          "contract": "fileA:A",
          "label": "a",
          "offset": 0,
          "slot": "0",
          "type": "t_uint128"
        },
        {
          "astId": 5,
          "contract": "fileA:A",
          "label": "b",
          "offset": 16,
          "slot": "0",
          "type": "t_uint128"
        },
        {
          "astId": 9,
          "contract": "fileA:A",
          "label": "staticArray",
          "offset": 0,
          "slot": "1",
          "type": "t_array(t_uint256)2_storage"
        },
        {
          "astId": 12,
          "contract": "fileA:A",
          "label": "dynArray",
          "offset": 0,
          "slot": "3",
          "type": "t_array(t_uint256)dyn_storage"
        }
      ],
      "numberOfBytes": "128"
    },
    "t_uint128": {
      "encoding": "inplace",
      "label": "uint128",
      "numberOfBytes": "16"
    },
    "t_uint256": {
      "encoding": "inplace",
      "label": "uint256",
      "numberOfBytes": "32"
    }
  }
}
```

## 内存的布局

Solidity保留四个32字节的槽，特定的字节范围(包括端点)被使用如下：

- `0x00` - `0x3f` (64 bytes): 为 hash 函数留出暂存空间（scratch spece）
- `0x40` - `0x5f` (32 bytes): 当前分配的内存大小(又称，空闲内存指针，free memory pointer)
- `0x60` - `0x7f` (32 bytes): 零槽（zero slot）

暂存空间能在语句间使用（例如，在内联汇编中）。零槽用作动态内存数组的初始值，并且不应该被写入（空闲内存指针最初指向 `0x80`）。

Solidity 总是将新对象放在空闲内存指针处，内存永远不会被释放(这在将来可能会改变)。

Solidity 中，内存数组的元素总是占用 32 字节的倍数（对于 `bytes1[]` 是这样，但对 `bytes` 和 `string` 不是）。多维内存数组是指向内存数组的指针。动态数组的长度存储在数组的第一个槽中，后面跟着数组元素。

> WARNING
>
> 在Solidity中有一些操作需要大于64字节的临时内存区域，因此无法装入暂存空间。它们将被放置在空闲内存所指向的位置，但是由于它们的生存期很短，指针不会被更新。内存可能被置零，也可能不被置零。因此，我们不应该期望空闲内存指向被置零的内存。
>
> 虽然使用 `msize` 来到达一个绝对为零的内存区域似乎是一个好主意，但是使用这样一个非临时的指针而不更新空闲内存指针可能会产生意想不到的结果。

### 与存储布局的差异

如上所述，内存中的布局不同于存储中的布局。下面是一些例子：

#### 在数组上的差异示例

下面的数组在存储中占用 32 字节（1个存储槽），而在内存中占用 128 字节（4个元素，每个都占 32 字节）。

```solidity
uint8[4] a;
```

#### 在结构体布局的差异示例

下面的结构体在存储中占用 96 字节（3个 32字节槽），但在内存中占用 128 字节（4个字段，每个占 32 字节）。

```solidity
struct S {
    uint a;
    uint b;
    uint8 c;
    uint8 d;
}
```



## Calldata 的布局

假定函数调用的输入数据采用[ABI 规范](https://docs.soliditylang.org/en/v0.8.11/abi-spec.html#abi)定义的格式。其中，ABI 规范要求将参数填充为 32 字节的倍数。内部函数调用使用不同的约定。

合约构造函数的参数直接附加在合约代码的末尾，同样采用 ABI 编码。构造函数将通过硬编码的偏移量访问它们，而不是使用 `codesize` 操作码，因为在将数据附加到代码时这当然会改变。



## 有助于理解布局的示例

[从私有数据中读取信息](https://mp.weixin.qq.com/s/_DV6UaRdA_6pUFXt-EnTtA)：

```solidity
contract Vault {
    uint public count = 123; // slot 0
    // owner、isTrue、u16 都被编码进同一个 slot，slot 1
    // 因为地址 20 字节 + bool 1字节 + uint16 2 字节 = 23 字节
    // 编码后的数据为：0x000000000000000000001f015b38da6a701c568545dcfcb03fcb875f56beddc4
    address public owner = msg.sender; // 0x5b38da6a701c568545dcfcb03fcb875f56beddc4
    bool public isTrue = true; // 0x01
    uint16 public u16 = 31; // 0x1f

    // 000000000000000000001f
    bytes32 private password; // slot 2
    uint public constant someConst = 123; // 常量不占用 slot
    bytes32[3] public data; // slot 3, slot4, slot5

    struct User {
        uint id;
        bytes32 password;
    }
    User[] private users; // slot 6
    mapping(uint => User) private idToUser; // slot 7

    constructor(uint _password) {
        password = bytes32(_password);
    }

    function addUser(bytes32 _password) public {
        User memory user = User({id: users.length, password: _password});

        users.push(user);
        idToUser[user.id] = user;
    }
    
    function getStorageKey(uint slot) public pure returns (bytes32) {
        return bytes32(keccak256(abi.encodePacked(slot)));
    }

    function getStorageAt(uint slot) public view returns (bytes32 ret) {
        assembly {
            ret := sload(slot)
        }
    }

    function getArrayDataSlotGivenIndex(
        uint slot,
        uint index
    ) public pure returns (bytes32) {
        return bytes32(uint(keccak256(abi.encodePacked(slot))) + index);
    }

    function getMapLocation(uint slot, uint key) public pure returns (uint) {
        return uint(keccak256(abi.encodePacked(key, slot)));
    }
}
```



## 清除变量

当一个值小于 256 位，这个情况下剩余比特必须清空。在任何可能由于剩余比特的脏数据受到不利影响的操作之前，Solidity 编译器会去清理这些剩余比特。例如，在写一个值到内存之前，剩余比特需要清空，因为内存的内容可以用于计算哈希值或作为一个消息调用的数据发送。相似地，在存储一个值到存储之前，剩余比特需要清空，否则会干扰数据。

注意，通过内联汇编访问不被认为是这样的操作：如果你使用内联汇编访问小于 256 位的solididity变量，编译器不能保证该值被正确清理。

此外，如果紧随其后的操作不受影响，我们不会清除比特。例如，由于 `JUMPI` 指令认为任何非零值都为 `true`，所以在将布尔值用作 `JUMPI` 的条件之前，我们不会清除它们。

除了上面的设计原则外，Solidity 编译器在将输入数据加载到堆栈时也会清理输入数据。 不同的类型有不同的清理无效值的规则：

| Type              | Valid Values       | Invalid Values Mean                                          |
| ----------------- | ------------------ | ------------------------------------------------------------ |
| enum of n members | 0 until n - 1      | exception                                                    |
| bool              | 0 or 1             | 1                                                            |
| signed integers   | sign-extended word | currently silently wraps; in the future exceptions will be thrown |
| unsigned integers | higher bits zeroed | currently silently wraps; in the future exceptions will be thrown |



## 合约元数据

## 合约 ABI 规范

合约应用二进制接口 (Application Binary Interface，ABI) 是在以太坊生态系统中与合约交互的标准方式，既可以从区块链外部进行，也可以用于合约间交互。数据根据其类型进行编码，如本规范中所述。编码不是自我描述的，因此需要一个模式才能解码。

我们假设合约的接口函数是强类型的，在编译时已知并且是静态的。我们假设所有合约都具有它们在编译时调用的任何合约的接口定义。

此规范不处理接口是动态的或仅在运行时已知的合约。

### 函数选择器（Function Selector）

一个函数调用的调用数据（call data）的前四个字节指出要调用哪一个函数。它是函数签名的kecak -256哈希值的第一个(以大端顺序的左侧/高阶)四个字节。签名被定义为基本原型的规范表达式，没有数据位置说明符，即带有圆括号中的参数类型列表的函数名。参数类型由一个逗号分隔——不使用空格。

> 注意
>
> 函数的返回类型不是该签名的一部分。在Solidity的函数中，不考虑重载返回类型。这样做的原因是为了保持函数调用解析与上下文无关。然而，ABI的JSON描述同时包含输入和输出。



### 参数编码

从第五个字节开始，后面是编码的参数。这种编码也用在其他地方，例如返回值和事件参数以相同的方式编码，没有指定函数的四个字节。

### 类型

存在以下基本类型：

- `uint<M>`: `M` 位的无符号整型， `0 < M <= 256`, `M % 8 == 0`。例如 `uint32`, `uint8`, `uint256`。
- `int<M>`: `M` 位的二进制补码有符号整型, `0 < M <= 256`, `M % 8 == 0`.
- `address`: 除了假定的解释和语言输入，等同于 `uint160`。 为了计算函数选择器，使用 `address`。
- `uint`, `int`: 分别是 `uint256`, `int256` 的别名。为了计算函数选择器，必须使用 `uint256` 和 `int256`。
- `bool`: 相当于`uint8`限制为值 0 和 1。为了计算函数选择器，使用 `bool` 。
- `fixed<M>x<N>`： `M` 位的有符号定点十进制数 ， `8 <= M <= 256`, `M % 8 == 0`, 并且 `0 < N <= 80`, 值 `v` 标记为 `v / (10 ** N)` 。
- `ufixed<M>x<N>`:  `fixed<M>x<N>` 的无符号变体。
- `fixed`, `ufixed`: 等同于 `fixed128x18` 和 `ufixed128x18` 。 为了计算函数选择器，必须使用 `fixed128x18` 和 `ufixed128x18` 。
- `bytes<M>`: `M` 字节的二进制类型, `0 < M <= 32`。
- `function`: 一个地址（20 个字节），后跟一个函数选择器（4 个字节）。编码等同于 `bytes24`。

存在以下（固定大小）数组类型：

- `<type>[M]`: 给定类型的 `M` 个元素的固定长度数组， `M >= 0`。

> 注意
>
> 虽然这个ABI规范可以用零元素表示固定长度的数组，但编译器不支持这种数组。

存在以下非固定大小类型：

- `bytes`：动态大小的字节序列。
- `string`：假定为 UTF-8 编码的动态大小的 unicode 字符串。
- `<type>[]`：由给定类型的元素组成的变长数组。

类型可以通过将它们括在用逗号分隔的圆括号中组合成一个元组：

- `(T1,T2,...,Tn)`：由类型 `T1`, …, `Tn`, `n >= 0` 组成的元祖。

可以形成元组的元组、元组的数组等等。也可以形成零元组(其中n == 0)。

### 将Solidity映射到ABI类型

除了元组之外，Solidity 支持上面提到的所有具有相同名称的类型。另一方面，ABI 不支持某些 Solidity 类型。下表在左列显示不属于 ABI 的 Solidity 类型，在右列显示代表它们的 ABI 类型。

| Solidity                                                     | ABI            |
| ------------------------------------------------------------ | -------------- |
| [address payable](https://docs.soliditylang.org/en/v0.8.11/types.html#address) | `address`      |
| [contract](https://docs.soliditylang.org/en/v0.8.11/contracts.html#contracts) | `address`      |
| [enum](https://docs.soliditylang.org/en/v0.8.11/types.html#enums) | `uint8`        |
| [user defined value types](https://docs.soliditylang.org/en/v0.8.11/types.html#user-defined-value-types) | 它的底层值类型 |
| [struct](https://docs.soliditylang.org/en/v0.8.11/types.html#structs) | `tuple`        |

> 警告
>
> 在0.8.0版本之前，枚举可以有超过256个成员，并且由最小的整数类型表示，其大小刚好可以容纳任何成员的值。

### 编码的设计标准

编码被设计为具有以下属性，如果某些参数是嵌套数组，这些属性特别有用：

1. 访问一个值所需要的读取次数最多是该值在参数数组结构中的深度，即检索 `a_i[k][l][r]` 需要读取 4 次。在ABI的前一个版本中，在最坏的情况下，读取的次数与动态参数的总数线性缩放。
2. 变量或数组元素的数据不与其他数据交叉，它是可重定位的，也就是说，它只使用相对“地址”。

### 编码的正式规范

我们区分静态和动态类型。静态类型在原地编码，动态类型在当前块之后的单独分配位置编码。

**定义：**以下类型称为“动态”：

- `bytes`
- `string`
- `T[] `对于任何 `T`
- `T[k] `对于任何动态 `T` 和任何 `k >= 0`
- `(T1,...,Tk)` 如果 `Ti` 对某些人来说是动态的 `1 <= i <= k`

所有其他类型都称为“静态”。

**定义**：`len(a)` 是一个二进制字符串 `a` 的字节数。`len(a)` 的类型是 `uint256` 。

我们定义 `enc` ，是实际的编码，作为 ABI 类型的值到二进制字符串的一个映射。当且仅当 `X` 的类型是动态的时候，`len(enc(X))` 取决于 `X` 的值。

**定义**：对于任意 ABI 值 `X`，我们递归地定义 `enc(X)` ，这取决于 `X` 的类型是：

- `T[k]`对于任何 `T` 和 `k` ：

  `enc(X) = enc((X[0], ..., X[k-1]))`

  即它被编码为就好像它是一个具有 `k` 相同类型元素的元组一样。

- `T[]` 其中 `X` 有 `k` 元素（ `k` 假定为类型 `uint256` ）：

  `enc(X) = enc(k) enc([X[0], ..., X[k-1]])`

  即，它被编码为一个静态大小的数组 `k`，前缀为元素数。

- `bytes`，长度 `k`（假定为类型 `uint256`）：

  `enc(X) = enc(k) pad_right(X)`，即字节数被编码为一个 `uint256` 后跟作为字节序列 `X` 的实际值，然后是最小的零字节数，以至 `len(enc(X))` 是 32 的倍数。

- `string`:

  `enc(X) = enc(enc_utf8(X))`，即 `X` 是 UTF-8 编码的，这个值被解释为 `bytes` 类型并进一步编码。请注意，此后续编码中使用的长度是 UTF-8 编码字符串的字节数，而不是其字符数。

- `uint<M>`: `enc(X)` 是 `X` 的大端编码，在高阶（左侧）用零字节填充，使得长度为 32 字节。

- `address`: 等同于 `uint160` 的情况。

- `int<M>`: `enc(X)` 是 `X` 的大端二进制补码编码， 为负数  `X`在高阶（左）侧填充`0xff` 字节，并且为非负数 `X` 填充零字节，以至编码后长度为 32 个字节。

- `bool`: 和`uint8`情况一样，其中 `1` 为 `true`，`0` 为 `false`。

- `fixed<M>x<N>`: `enc(X)` 是 `enc(X * 10**N)` 。其中 `X * 10**N` 被解释为 `int256`。

- `fixed`: 与 `fixed128x18` 的情况一样。

- `ufixed<M>x<N>`: `enc(X)` 是 `enc(X * 10**N)` 。其中 `X * 10**N` 被解释为 `uint256`。

- `ufixed`: 与 `ufixed128x18` 的情况一样

- `bytes<M>`: `enc(X)` 用零字节填充在后面使其长度为 32 字节的  `X` 的字节序列。

请注意，对于任何 `X`，`len(enc(X))` 是 32 的倍数。

### 函数选择器和参数编码

总而言之，一个对函数 `f` 参数为 `a_1, ..., a_n` 的函数调用编码为：

`function_selector(f) enc((a_1, ..., a_n))`

并且 `f` 返回值 `v_1, ..., v_k` 的编码为：

`enc((v_1, ..., v_k))`

也就是说，值被组合成一个元组并进行编码。

### 示例

给定如下合约：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.16 <0.9.0;

contract Foo {
    function bar(bytes3[2] memory) public pure {}
    function baz(uint32 x, bool y) public pure returns (bool r) { r = x > 32 || y; }
    function sam(bytes memory, bool, uint[] memory) public pure {}
}
```



因此，对于我们的 `Foo` 示例，如果我们想要以参数 `69` 和 `true` 调用函数 `baz`，我们会总共传 `68` 字节，解释如下：

- `0xcdcd77c0`：Method ID（函数选择器）。由函数签名 `baz(uint32,bool)` 的 ASCII 格式的 `Keccak` hash 的前 4 字节得到。
- `0x0000000000000000000000000000000000000000000000000000000000000045`: 第一个参数，一个 uint32 值，`69`填充到 32 个字节
- `0x0000000000000000000000000000000000000000000000000000000000000001`: 第二个参数 - boolean `true`, 填充到 32 字节

总共：

```
0xcdcd77c000000000000000000000000000000000000000000000000000000000000000450000000000000000000000000000000000000000000000000000000000000001
```

它返回一个`bool`。例如，如果它要返回 `false`，它的输出将是单个字节数组`0x0000000000000000000000000000000000000000000000000000000000000000`，一个布尔值。

如果我们想要以参数 `["abc", "def"]` 调用函数 `bar` ，我们一共会传 68 字节，解释如下：

- `0xfce353f6`: Method ID。这是从签名 `bar(bytes3[2])` 中得出的。
- `0x6162630000000000000000000000000000000000000000000000000000000000`: 第一个参数的第一部分，一个 `bytes3` 值 `"abc"`（左对齐）。
- `0x6465660000000000000000000000000000000000000000000000000000000000`：第一个参数的第二部分，一个 `bytes3` 值 `"def"`（左对齐）。

总共：

```
0xfce353f661626300000000000000000000000000000000000000000000000000000000006465660000000000000000000000000000000000000000000000000000000000
```

如果我们想用参数`"dave"`，`true`和 `[1,2,3]` 调用 `sam`，我们将传总共 292 个字节，解释如下：

- `0xa5643bf2`: Method ID。这是从签名 `sam(bytes,bool,uint256[])` 中得出的。请注意，`uint` 替换为它的规范表示 `uint256`。
- `0x0000000000000000000000000000000000000000000000000000000000000060`：第一个参数(动态类型)的数据部分的位置，从参数块开始以字节为单位测量。在这种情况下为 `0x60`。
- `0x0000000000000000000000000000000000000000000000000000000000000001`：第二个参数：布尔值 true。
- `0x00000000000000000000000000000000000000000000000000000000000000a0`：第三个参数（动态类型）的数据部分的位置，以字节为单位测量。在这种情况下为 `0xa0`。
- `0x0000000000000000000000000000000000000000000000000000000000000004`: 第一个参数的数据部分，它以字节数组的元素长度开始，在本例中为 4。
- `0x6461766500000000000000000000000000000000000000000000000000000000`: 第一个参数的内容： `"dave"` 的 UTF-8（在这种情况下等于 ASCII）编码，在右侧填充到 32 个字节。
- `0x0000000000000000000000000000000000000000000000000000000000000003`：第三个参数的数据部分，它以元素数组的长度开始，在本例中为 3。
- `0x0000000000000000000000000000000000000000000000000000000000000001`：第三个参数的第一个元素。
- `0x0000000000000000000000000000000000000000000000000000000000000002`: 第三个参数的第二个元素。
- `0x0000000000000000000000000000000000000000000000000000000000000003`: 第三个参数的第三个元素。

总共：

```
0xa5643bf20000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000000000000000000000000000464617665000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000003
```



### 使用动态类型

用参数 `(0x123, [0x456, 0x789], "1234567890", "Hello, world!")` 调用一个签名为 `f(uint,uint32[],bytes10,bytes)` 的函数，其编码方式如下：

我们取 `sha3("f(uint256,uint32[],bytes10,bytes)")` 的前四个字节，即 `0x8be65246`。然后我们对所有四个参数的头部部分进行编码。对于静态类型 `uint256` 和 `bytes10`，这些直接是我们想要传递的值，而对于动态类型 `uint32[]` 和 `bytes`，我们使用字节偏移量到它们的数据区域的开头，从值编码的开头测量（即不计算包含函数签名哈希的前四个字节）。这些是：

- `0x0000000000000000000000000000000000000000000000000000000000000123`（`0x123`填充到 32 个字节）
- `0x0000000000000000000000000000000000000000000000000000000000000080`（到第二个参数数据部分开始的偏移量，4*32 字节（128，hex'80'），正好是头部的大小）
- `0x3132333435363738393000000000000000000000000000000000000000000000`（`"1234567890" `在右边填充到 32 个字节）
- `0x00000000000000000000000000000000000000000000000000000000000000e0`（第四个参数数据部分开始的偏移量 = 第一个动态参数数据部分开始的偏移量 + 第一个动态参数数据部分的大小 = 4\*32 + 3\*32（见下文））

在此之后，第一个动态参数的数据部分如下：`[0x456, 0x789]`

- `0x0000000000000000000000000000000000000000000000000000000000000002`（数组元素个数，2）
- `0x0000000000000000000000000000000000000000000000000000000000000456`（第一个元素）
- `0x0000000000000000000000000000000000000000000000000000000000000789`（第二个元素）

最后，我们对第二个动态参数的数据部分进行编码：`"Hello, world!"`

- `0x000000000000000000000000000000000000000000000000000000000000000d`（元素数量（在这种情况下为字节）：13）
- `0x48656c6c6f2c20776f726c642100000000000000000000000000000000000000`（在右边填充到 32 个字节）`"Hello, world!"`

总之，编码是（为清楚起见，函数选择器和每个 32 字节后加入换行符）：

```
0x8be65246
  0000000000000000000000000000000000000000000000000000000000000123
  0000000000000000000000000000000000000000000000000000000000000080
  3132333435363738393000000000000000000000000000000000000000000000
  00000000000000000000000000000000000000000000000000000000000000e0
  0000000000000000000000000000000000000000000000000000000000000002
  0000000000000000000000000000000000000000000000000000000000000456
  0000000000000000000000000000000000000000000000000000000000000789
  000000000000000000000000000000000000000000000000000000000000000d
  48656c6c6f2c20776f726c642100000000000000000000000000000000000000
```

让我们应用相同的原则，用参数 `([[1, 2], [3]], ["one", "two", "three"])` 对具有 `g(uint[][],string[])` 签名的函数进行编码，但从编码的最原子部分开始：

首先我们对第一个根数组  `[[1, 2], [3]]`  的第一个嵌入动态数组 `[1, 2]` 的长度和数据进行编码：

- `0x0000000000000000000000000000000000000000000000000000000000000002`（第一个数组中的元素数，2；元素本身是`1`和`2`）
- `0x0000000000000000000000000000000000000000000000000000000000000001`（第一个元素）
- `0x0000000000000000000000000000000000000000000000000000000000000002`（第二个元素）

然后我们对第一个根数组 `[[1, 2], [3]]` 的第二个嵌入动态数组 `[3]` 的长度和数据进行编码：

- `0x0000000000000000000000000000000000000000000000000000000000000001`（第二个数组的元素个数，1；元素是`3`）
- `0x0000000000000000000000000000000000000000000000000000000000000003`（第一个元素）

然后我们需要找到动态数组`[1, 2]` 和 `[3]` 各自的偏移量 `a` 和`b` 。要计算偏移量，我们可以查看第一个根数组 `[[1, 2], [3]]` 的编码数据， 枚举编码中的每一行：

```
0 - a                                                                - offset of [1, 2]
1 - b                                                                - offset of [3]
2 - 0000000000000000000000000000000000000000000000000000000000000002 - count for [1, 2]
3 - 0000000000000000000000000000000000000000000000000000000000000001 - encoding of 1
4 - 0000000000000000000000000000000000000000000000000000000000000002 - encoding of 2
5 - 0000000000000000000000000000000000000000000000000000000000000001 - count for [3]
6 - 0000000000000000000000000000000000000000000000000000000000000003 - encoding of 3
```

偏移量 `a` 指向数组内容 `[1, 2]` 的开始，即第 2 行（64 字节）；因，.`a = 0x0000000000000000000000000000000000000000000000000000000000000040`

偏移量 `b` 指向数组内容 `[3]` 的开始，即第 5 行（160 字节）；因此，`b = 0x00000000000000000000000000000000000000000000000000000000000000a0`

然后我们对第二个根数组的嵌入字符串进行编码：

- `0x0000000000000000000000000000000000000000000000000000000000000003`（单词中的字符数`"one"`）
- `0x6f6e650000000000000000000000000000000000000000000000000000000000`（单词的utf8表示`"one"`）
- `0x0000000000000000000000000000000000000000000000000000000000000003`（单词中的字符数`"two"`）
- `0x74776f0000000000000000000000000000000000000000000000000000000000`（单词的utf8表示`"two"`）
- `0x0000000000000000000000000000000000000000000000000000000000000005`（单词中的字符数`"three"`）
- `0x7468726565000000000000000000000000000000000000000000000000000000`（单词的utf8表示`"three"`）



与第一个根数组一样，因为字符串是动态元素，我们需要找到它们的偏移量 `c`，`d` 和`e`：

```
0 - c                                                                - offset for "one"
1 - d                                                                - offset for "two"
2 - e                                                                - offset for "three"
3 - 0000000000000000000000000000000000000000000000000000000000000003 - count for "one"
4 - 6f6e650000000000000000000000000000000000000000000000000000000000 - encoding of "one"
5 - 0000000000000000000000000000000000000000000000000000000000000003 - count for "two"
6 - 74776f0000000000000000000000000000000000000000000000000000000000 - encoding of "two"
7 - 0000000000000000000000000000000000000000000000000000000000000005 - count for "three"
8 - 7468726565000000000000000000000000000000000000000000000000000000 - encoding of "three"
```

偏移量 `c` 指向字符串内容的开头，`"one" ` 即第 3 行（96 字节）；因此，`c = 0x0000000000000000000000000000000000000000000000000000000000000060`

偏移量 `d` 指向字符串内容的开头，`"two" `即第 5 行（160 字节）；因此，`d = 0x00000000000000000000000000000000000000000000000000000000000000a0`

偏移量 `e` 指向字符串内容的开头，`"three" `即第 7 行（224 字节）；因此，`e = 0x00000000000000000000000000000000000000000000000000000000000000e0`

请注意，根数组的嵌入元素的编码彼此不依赖，并且对于具有签名 `g(string[],uint[][])` 的函数具有相同的编码。

然后我们对第一个根数组的长度进行编码：

- `0x0000000000000000000000000000000000000000000000000000000000000002`（第一个根数组中的元素数，2；元素本身是 `[1, 2]` 和`[3]`）

然后我们对第二个根数组的长度进行编码：

- `0x0000000000000000000000000000000000000000000000000000000000000003`（第二个根数组中的字符串数，3；字符串本身是 `"one"`，`"two"` 和 `"three"`）

最后，我们找到根动态数组 `[[1, 2], [3]]` 和 `["one", "two", "three"]`各自的偏移量 `f` 和 `g`，并以正确的顺序组装这些部分：

```
0x2289b18c                                                            - function signature
 0 - f                                                                - offset of [[1, 2], [3]]
 1 - g                                                                - offset of ["one", "two", "three"]
 2 - 0000000000000000000000000000000000000000000000000000000000000002 - count for [[1, 2], [3]]
 3 - 0000000000000000000000000000000000000000000000000000000000000040 - offset of [1, 2]
 4 - 00000000000000000000000000000000000000000000000000000000000000a0 - offset of [3]
 5 - 0000000000000000000000000000000000000000000000000000000000000002 - count for [1, 2]
 6 - 0000000000000000000000000000000000000000000000000000000000000001 - encoding of 1
 7 - 0000000000000000000000000000000000000000000000000000000000000002 - encoding of 2
 8 - 0000000000000000000000000000000000000000000000000000000000000001 - count for [3]
 9 - 0000000000000000000000000000000000000000000000000000000000000003 - encoding of 3
10 - 0000000000000000000000000000000000000000000000000000000000000003 - count for ["one", "two", "three"]
11 - 0000000000000000000000000000000000000000000000000000000000000060 - offset for "one"
12 - 00000000000000000000000000000000000000000000000000000000000000a0 - offset for "two"
13 - 00000000000000000000000000000000000000000000000000000000000000e0 - offset for "three"
14 - 0000000000000000000000000000000000000000000000000000000000000003 - count for "one"
15 - 6f6e650000000000000000000000000000000000000000000000000000000000 - encoding of "one"
16 - 0000000000000000000000000000000000000000000000000000000000000003 - count for "two"
17 - 74776f0000000000000000000000000000000000000000000000000000000000 - encoding of "two"
18 - 0000000000000000000000000000000000000000000000000000000000000005 - count for "three"
19 - 7468726565000000000000000000000000000000000000000000000000000000 - encoding of "three"
```

偏移量 `f `指向数组内容 `[[1, 2], [3]]` 的开始，即第 2 行（64 字节）；因此，`f = 0x0000000000000000000000000000000000000000000000000000000000000040`

偏移量 `g` 指向数组内容 `["one", "two", "three"]` 的开始，即第 10 行（320 字节）；因此，`g = 0x0000000000000000000000000000000000000000000000000000000000000140`

### 事件

事件是以太坊日志/事件监视协议的抽象。日志条目提供合约的地址、一系列最多四个主题（topic）和一些任意长度的二进制数据。事件利用现有的函数 ABI 来将其（连同接口规范）解释为正确类型的结构。

给定一个事件名称和一系列事件参数，我们将它们分成两个子系列：那些被索引的和那些没有被索引的。那些被索引的，可能最多 3 个（对于非匿名事件）或 4 个（对于匿名事件），与事件签名的 Keccak Hash 一起使用以形成日志条目的主题。那些没有被索引的构成事件的字节数组。

实际上，使用此 ABI 的日志条目描述为：

- `address`：合约地址（本质上由以太坊提供）；
- `topics[0]`：`keccak(EVENT_NAME+"("+EVENT_ARGS.map(canonical_type_of).join(",")+")")` ( `canonical_type_of` 是一个简单地返回给定参数的规范类型的函数，例如对于 `uint indexed foo`，它会返回 `uint256`)。`topics[0]` 仅在事件未声明为 `anonymous` 时出现；
- `topics[n]`：
  - 定义：被索引的 `EVENT_ARGS` 的列表是 `EVENT_INDEXED_ARGS`
  - `abi_encode(EVENT_INDEXED_ARGS[n - 1])`，如果事件没有声明为 `anonymous` 
  - `abi_encode(EVENT_INDEXED_ARGS[n])`，如果事件声明为 `anonymous`
- `data`：`EVENT_NON_INDEXED_ARGS` 的 ABI 编码(`EVENT_NON_INDEXED_ARGS` 是未索引 `EVENT_ARGS` 的列表，`abi_encode` 是用于从函数返回一系列类型值的 ABI 编码函数，如上所述)。



对于最多 32 字节的所有类型的长度，`EVENT_INDEXED_ARGS` 数组直接包含值，填充或符号扩展（对于有符号整数）到 32 字节，就像常规 ABI 编码一样。但是，对于所有“复杂”类型或动态长度类型，包括所有数组、`string` 和 `bytes` 结构， `EVENT_INDEXED_ARGS `将包含特殊原地编码值的 Keccak 哈希（请参阅[索引事件参数的编码](https://docs.soliditylang.org/en/v0.8.11/abi-spec.html#indexed-event-encoding))，而不是直接编码值。这允许应用程序有效地查询动态长度类型的值（通过将编码值的哈希设置为主题），但使应用程序无法解码它们未查询的索引值。对于动态长度类型，应用程序开发人员面临在快速搜索预定值（如果参数被索引）和任意值的易读性（要求参数不被索引）之间的权衡。开发人员可以通过定义具有两个参数的事件来克服这种折衷并实现高效搜索和任意易读性 - 一个有索引，一个没有 - 旨在保持相同的值。



### 错误

如果合约内部发生故障，合约可以使用特殊的操作码中止执行并恢复所有状态更改。除了这些效果之外，还可以将描述性数据返回给调用者。此描述性数据是错误及其参数的编码，其方式与函数调用的数据相同。

例如，让我们考虑以下合约，其 `transfer` 功能总是以“余额不足”的自定义错误恢复：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;

contract TestToken {
    error InsufficientBalance(uint256 available, uint256 required);
    function transfer(address /*to*/, uint amount) public pure {
        revert InsufficientBalance(0, amount);
    }
}
```



返回数据的编码方式与对函数 `InsufficientBalance(uint256,uint256)` 的函数调用 `InsufficientBalance(0, amount)` 相同。即，`0xcf479181`，`uint256(0)`，`uint256(amount)`。

错误选择器`0x00000000`和`0xffffffff`保留以供将来使用。

> 警告
>
> 永远不要相信错误数据。默认情况下，错误数据会通过外部调用链向上冒泡，这意味着合约可能会收到未在其直接调用的任何合约中定义的错误。此外，任何合约都可以通过返回与错误签名匹配的数据来伪造任何错误，即使错误没有在任何地方定义。



### JSON

合约接口的 JSON 格式由一组函数、事件和错误描述给出。函数描述是具有以下字段的 JSON 对象：

- `type`: `"function"`, `"constructor"`, `"receive"`（[“接收以太币”函数](https://docs.soliditylang.org/en/v0.8.11/contracts.html#receive-ether-function)）或 `"fallback"`（[“默认”函数](https://docs.soliditylang.org/en/v0.8.11/contracts.html#fallback-function)）；
- `name`: 函数名；
- `inputs`: 一个对象数组，每个对象包含：
  - `name`：参数的名称。
  - `type`：参数的规范类型（更多下文）。
  - `components`: 用于元组类型（更多下文）。
- `outputs`: 类似于 的对象数组`inputs`。
- `stateMutability`：具有以下值之一的字符串：（`pure`指定[不读取区块链状态](https://docs.soliditylang.org/en/v0.8.11/contracts.html#pure-functions)），`view`（[指定不修改区块链状态](https://docs.soliditylang.org/en/v0.8.11/contracts.html#view-functions)），`nonpayable`（函数不接受 Ether - 默认值）和`payable`（函数接受 Ether）。

构造函数（constructor）和默认函数（fallback function）永远不会有 `name` or `outputs`。默认函数也没有 `inputs`。

> 注意
>
> 向非支付函数发送非零以太币将恢复交易。
>
> 状态可变性 `nonpayable` 通过根本不指定状态可变性修饰符在 Solidity 中反映。



事件描述是一个 JSON 对象，具有非常相似的字段：

- `type`： 总是 `"event"`
- `name`：事件的名称。
- `inputs`: 一个对象数组，每个对象包含：
  - `name`：参数的名称。
  - `type`：参数的规范类型（更多下文）。
  - `components`: 用于元组类型（更多下文）。
  - `indexed`：`true` 如果该字段是日志主题的一部分，`false` 如果它是日志的数据段之一。
- `anonymous`: `true` 如果事件被声明为 `anonymous`.

错误如下所示：

- `type`： 总是 `"error"`
- `name`: 错误的名称。
- `inputs`: 一个对象数组，每个对象包含：
  - `name`：参数的名称。
  - `type`：参数的规范类型（更多下文）。
  - `components`: 用于元组类型（更多下文）。

> 注意
>
> 在 JSON 数组中可能存在多个具有相同名称甚至具有相同签名的错误，例如，如果错误源自智能合约中的不同文件或被另一个智能合约引用。对于 ABI，只有错误名称本身是相关的，而不是它的定义位置。

例如：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;


contract Test {
    constructor() { b = hex"12345678901234567890123456789012"; }
    event Event(uint indexed a, bytes32 b);
    event Event2(uint indexed a, bytes32 b);
    error InsufficientBalance(uint256 available, uint256 required);
    function foo(uint a) public { emit Event(a, b); }
    bytes32 b;
}
```

将导致 JSON：

```json
[{
"type":"error",
"inputs": [{"name":"available","type":"uint256"},{"name":"required","type":"uint256"}],
"name":"InsufficientBalance"
}, {
"type":"event",
"inputs": [{"name":"a","type":"uint256","indexed":true},{"name":"b","type":"bytes32","indexed":false}],
"name":"Event"
}, {
"type":"event",
"inputs": [{"name":"a","type":"uint256","indexed":true},{"name":"b","type":"bytes32","indexed":false}],
"name":"Event2"
}, {
"type":"function",
"inputs": [{"name":"a","type":"uint256"}],
"name":"foo",
"outputs": []
}]
```



### 处理元组类型

尽管名称故意不属于 ABI 编码的一部分，但将它们包含在 JSON 中以将其显示给最终用户确实很有意义。结构嵌套如下：

一个有成员 `name` ，`type` 和可能的 `components` 的对象描述了一个类型变量。在达到一个元组类型前，确认它的规范类型。元组的 component 存储在成员 `components` 中，`components` 是一个数组，并且与顶层对象有相同的结构（`indexed` 除外）。

示例代码：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.5 <0.9.0;
pragma abicoder v2;

contract Test {
    struct S { uint a; uint[] b; T[] c; }
    struct T { uint x; uint y; }
    function f(S memory, T memory, uint) public pure {}
    function g() public pure returns (S memory, T memory, uint) {}
}
```



JSON 中的结果是：

```json
[
  {
    "name": "f",
    "type": "function",
    "inputs": [
      {
        "name": "s",
        "type": "tuple",
        "components": [
          {
            "name": "a",
            "type": "uint256"
          },
          {
            "name": "b",
            "type": "uint256[]"
          },
          {
            "name": "c",
            "type": "tuple[]",
            "components": [
              {
                "name": "x",
                "type": "uint256"
              },
              {
                "name": "y",
                "type": "uint256"
              }
            ]
          }
        ]
      },
      {
        "name": "t",
        "type": "tuple",
        "components": [
          {
            "name": "x",
            "type": "uint256"
          },
          {
            "name": "y",
            "type": "uint256"
          }
        ]
      },
      {
        "name": "a",
        "type": "uint256"
      }
    ],
    "outputs": []
  }
]
```



### 严格编码模式

严格编码模式是导致与上述正式规范中定义的编码完全相同的模式。这意味着偏移量必须尽可能小，同时仍然不会在数据区域中产生重叠，因此不允许有间隙。

通常，ABI 解码器是在偏移指针之后以直接的方式编写的，但一些解码器可能会强制执行严格模式。Solidity ABI 解码器目前不强制执行严格模式，但编码器始终以严格模式创建数据。

### 非标准打包模式

通过 `abi.encodePacked()`，Solidity 支持非标准打包模式，其中：

- 小于 32 字节的类型直接连接，没有填充或符号扩展
- 动态类型在原地编码并且没有长度。
- 数组元素被填充，但仍原地编码

此外，不支持结构和嵌套数组。

例如，`int16(-1), bytes1(0x42), uint16(0x03), string("Hello, world!")` 的编码为：

```
0xffff42000348656c6c6f2c20776f726c6421
  ^^^^                                 int16(-1)
      ^^                               bytes1(0x42)
        ^^^^                           uint16(0x03)
            ^^^^^^^^^^^^^^^^^^^^^^^^^^ string("Hello, world!") without a length field
```

进一步来说：

- 在编码期间，所有内容都在原地编码。这意味着没有头尾之分，就像在 ABI 编码中一样，并且数组的长度没有被编码。
- `abi.encodePacked` 的直接参数在没有填充的情况下进行编码，只要它们不是数组， `string` 或 `bytes`。
- 数组的编码是其元素的编码与填充的连接。
- 动态大小的类型，如 `string` ，`bytes` 或 `uint[]` 在没有长度字段的情况下进行编码。
- `string` 或 `bytes` 的编码在末尾不应用填充，除非它是数组或结构的一部分（然后它被填充为 32 字节的倍数）。

通常，由于缺少长度字段，只要有两个动态大小的元素，编码就会不明确。

如果需要填充，可以使用显式类型转换：`abi.encodePacked(uint16(0x12)) == hex"0012" ` 。

由于调用函数时不使用压缩编码，因此没有特别支持预先添加函数选择器。由于编码不明确，因此没有解码功能。

> 警告
>
> 如果你使用 `keccak256(abi.encodePacked(a, b))` 并且 `a` 和 `b` 都是动态类型，则很容易通过移动 `a` 的部分到 `b` 中去制造哈希碰撞，反之亦然。特别是 `abi.encodePacked("a", "bc") == abi.encodePacked("ab", "c")` 。如果你为了签名、认证或数据完整性使用 `abi.encodePacked` ，请确保总是使用相同的类型，并检查他们最多只能有一个是动态类型。除非有令人信服的原因，不然优先使用 `abi.encode` 。



### 索引事件参数的编码

引用类型的索引事件参数，即数组和结构体不直接存储，而是存储编码的 keccak256-hash。这种编码定义如下：

- `bytes` 和 `string` 的值的编码只是没有任何填充或长度前缀的字符串内容。
- 结构体的编码是其成员编码的串联，总是填充到 32 字节的倍数（甚至 `bytes` 和 `string` 也是）。
- 数组的编码（动态和静态大小）是其元素编码的串联，总是填充到 32 个字节的倍数（`bytes`和`string` 也是）并且没有任何长度前缀

在上面，像往常一样，负数由符号扩展填充，而不是零填充。 `bytesNN` 类型填充在右侧，而 `uintNN`/ `intNN`填充在左侧。

> 警告
>
> 如果结构体包含多个动态大小的数组，则结构体的编码是不明确的。因此，请始终重新检查事件数据，不要仅依赖基于索引参数的搜索结果。



## 内联汇编

### 内联汇编

内联汇编由 `assembly {}` 标记，其中，大括号内的代码是 Yul 语言的代码。

以下示例提供库代码来访问另一个合约的代码并将其加载到 `bytes` 变量中。这也可以简单地使用 solidity 代码  `<address>.code` 实现。但重点是，可复用的汇编库可以在不更改编译器的情况下，增强 solidity 语言。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.16 <0.9.0;

library GetCode {
    function at(address _addr) public view returns (bytes memory o_code) {
        assembly {
            // retrieve the size of the code, this needs assembly
            let size := extcodesize(_addr)
            // allocate output byte array - this could also be done without assembly
            // by using o_code = new bytes(size)
            o_code := mload(0x40)
            // new "memory end" including padding
            mstore(0x40, add(o_code, and(add(add(size, 0x20), 0x1f), not(0x1f))))
            // store length in memory
            mstore(o_code, size)
            // actually retrieve the code, this needs assembly
            extcodecopy(_addr, add(o_code, 0x20), 0, size)
        }
    }
}
```

在优化器无法生成高效代码的情况下，内联汇编也很有用，例如：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.16 <0.9.0;


library VectorSum {
    // This function is less efficient because the optimizer currently fails to
    // remove the bounds checks in array access.
    function sumSolidity(uint[] memory _data) public pure returns (uint sum) {
        for (uint i = 0; i < _data.length; ++i)
            sum += _data[i];
    }

    // We know that we only access the array in bounds, so we can avoid the check.
    // 0x20 needs to be added to an array because the first slot contains the
    // array length.
    function sumAsm(uint[] memory _data) public pure returns (uint sum) {
        for (uint i = 0; i < _data.length; ++i) {
            assembly {
                sum := add(sum, mload(add(add(_data, 0x20), mul(i, 0x20))))
            }
        }
    }

    // Same as above, but accomplish the entire code within inline assembly.
    function sumPureAsm(uint[] memory _data) public pure returns (uint sum) {
        assembly {
            // Load the length (first 32 bytes)
            let len := mload(_data)

            // Skip over the length field.
            //
            // Keep temporary variable so it can be incremented in place.
            //
            // NOTE: incrementing _data would result in an unusable
            //       _data variable after this assembly block
            let data := add(_data, 0x20)

            // Iterate until the bound is not met.
            for
                { let end := add(data, mul(len, 0x20)) }
                lt(data, end)
                { data := add(data, 0x20) }
            {
                sum := add(sum, mload(data))
            }
        }
    }
}
```

#### 访问外部变量、函数和库

您可以使用名称访问 Solidity 变量和其他标识符。

值类型的局部变量可直接在内联汇编中使用。它们都可以被读取和分配。

引用内存的局部变量计算为内存中变量的地址，而不是值本身。这些变量也可以被赋值，但请注意，赋值只会改变指针而不是数据，尊重 Solidity 的内存管理是您的责任。请参阅[Solidity](https://docs.soliditylang.org/en/v0.8.11/assembly.html#conventions-in-solidity)中的约定。

类似地，引用静态大小的 calldata 数组或 calldata 结构的局部变量计算为 calldata 中变量的地址，而不是值本身。也可以为变量分配一个新的偏移量，但请注意，不会执行任何验证以确保变量不会指向超出 `calldatasize()`。

对于外部函数指针，可以使用 `x.address` 和 `x.selector` 访问地址和函数选择器。选择器由四个右对齐字节组成，这两个值都被可以分配。例如：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.10 <0.9.0;

contract C {
    // Assigns a new selector and address to the return variable @fun
    function combineToFunctionPointer(address newAddress, uint newSelector) public pure returns (function() external fun) {
        assembly {
            fun.selector := newSelector
            fun.address  := newAddress
        }
    }
}
```

对于动态的 calldata 数组，你可以通过 `x.offset` 和 `x.length` 访问他们的 calldata 偏移量（bytes为单位）和长度（元素的个数）。这两个表达式也可以赋值，但对于静态情况，不会执行任何验证来确保产生的数据区域在calldatasize()的范围内。

对于本地存储变量或状态变量，单个 Yul 标识符是不够的，因为它们不一定占用单个完整的存储槽。因此，它们的“地址”由一个槽和该槽内的字节偏移量组成。要检索变量指向的槽 `x`，请使用 `x.slot`，并检索您使用的字节偏移量 `x.offset`。使用 `x` 自身会导致错误。

你也可以为本地存储变量指针的 `.slot` 成员赋值。对于 struct、array、mapping，它们的 `offset` 永远为 0。虽然为状态变量的 `.slot `和 `.offset` 赋值是不可能的。

局部变量可以被赋值：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0 <0.9.0;

contract C {
    uint b;
    function f(uint x) public view returns (uint r) {
        assembly {
            // We ignore the storage slot offset, we know it is zero
            // in this special case.
            r := mul(x, sload(b.slot))
        }
    }
}
```

> 警告
>
> 如果您访问跨度小于 256 位的类型的变量（例如`uint64`、`address`或`bytes16`），则不能对不属于该类型编码的位做出任何假设。特别是，不要假设它们为零。为了安全起见，在重要的上下文中使用数据之前，请务必正确清除数据： 要清除签名类型，您可以使用操作码： `uint32 x = f(); assembly { x := and(x, 0xffffffff) /* now use x */ }``signextend``assembly { signextend(<num_bytes_of_x_minus_one>, x) }`

从 Solidity 0.6.0 开始，内联汇编变量的名称可能不会影响内联汇编块范围内可见的任何声明（包括变量、合同和函数声明）。

从 Solidity 0.7.0 开始，在内联汇编块内声明的变量和函数可能不包含`.`，但 using`.`从内联汇编块外部访问 Solidity 变量是有效的。

#### 要避免的事情

内联汇编可能有一个相当高级的外观，但它实际上是非常低级的。函数调用、循环、if 和switch通过简单的重写规则进行转换，之后，汇编器为您做的唯一一件事就是重新排列函数式操作码、计算变量访问的堆栈高度以及删除汇编局部变量的堆栈槽当到达他们的块的末尾时。



#### Solidity 中的约定

与 EVM 汇编相比，Solidity 具有比 256 位更窄的类型，例如 `uint24`. 为提高效率，大多数算术运算忽略了类型可以短于 256 位这一事实，并且在必要时清除高阶位，即在将它们写入内存或执行比较之前不久。这意味着如果您从内联汇编中访问这样的变量，您可能必须先手动清除高阶位。

Solidity 通过以下方式管理内存。在内存中的位置有一个“空闲内存指针” `0x40`。如果要分配内存，请使用从该指针指向的位置开始的内存并对其进行更新。不能保证之前没有使用过内存，因此你不能假设它的内容是零字节。没有释放或释放分配的内存的内置机制。这是一个汇编片段，可用于按照上述过程分配内存

```solidity
function allocate(length) -> pos {
  pos := mload(0x40)
  mstore(0x40, add(pos, length))
}
```

内存的前 64 字节可用作“暂存空间”，用于短期分配。空闲内存指针之后的 32 个字节（即，从 开始`0x60`）意味着永久为零，并用作空动态内存数组的初始值。这意味着可分配内存从 开始`0x80`，这是空闲内存指针的初始值。

Solidity 中的内存数组中的元素总是占据 32 字节的倍数（这对于 甚至是正确的`bytes1[]`，但对于`bytes`and则不是`string`）。多维内存数组是指向内存数组的指针。动态数组的长度存储在数组的第一个槽中，然后是数组元素。

> 警告
>
> 静态大小的内存数组没有长度字段，但以后可能会添加它以允许在静态和动态大小的数组之间更好地转换，所以不要依赖这个。

### 示例

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.16 <0.9.0;

contract AssemblyExample {
    uint public a;

    function mul(uint b) public view returns (uint c) {
        assembly {
            c := mul(b, sload(a.slot))
        }
    }

    function setA(uint a_) public {
        assembly {
            sstore(a.slot, a_)
        }
    }

    function power(uint base, uint exponent) public pure returns (uint ret) {
        assembly {
            function power(base_, exponent_) -> result
            {
                switch exponent_
                case 0 { result := 1 }
                case 1 { result := base_ }
                default
                {
                    result := power(mul(base_, base_), div(exponent_, 2))
                    switch mod(exponent_, 2)
                    case 1 { result := mul(base_, result) }
                }
            }
            
            ret := power(base, exponent)
        }
    }

    /**
     * @dev logical shift left y by x bits (y << x).
     * if x is 3, y is 1, ret will be 8.
     */
    function shiftLeft(uint x, uint y) public pure returns (uint ret) {
        assembly {
            ret := shl(x, y)
        }
    }

    function getCode(address _addr) public view returns (bytes memory o_code) {
        assembly {
            // retrieve the size of the code, this needs assembly
            let size := extcodesize(_addr)
            // allocate output byte array - this could also be done without assembly
            // by using o_code = new bytes(size)
            o_code := mload(0x40)
            // new "memory end" including padding
            mstore(0x40, add(o_code, and(add(add(size, 0x20), 0x1f), not(0x1f))))
            // store length in memory
            mstore(o_code, size)
            // actually retrieve the code, this needs assembly
            extcodecopy(_addr, add(o_code, 0x20), 0, size)
        }
    }

    function getStorageKey(uint slot) public pure returns (bytes32) {
        return bytes32(keccak256(abi.encodePacked(slot)));
    }

    function getStorageAt(uint slot) public view returns (bytes32 ret) {
        assembly {
            ret := sload(slot)
        }
    }

    function getArrayDataSlotGivenIndex(
        uint slot,
        uint index
    ) public pure returns (bytes32) {
        return bytes32(uint(keccak256(abi.encodePacked(slot))) + index);
    }

    function getMapLocation(uint slot, uint key) public pure returns (uint) {
        return uint(keccak256(abi.encodePacked(key, slot)));
    }

    function removeArrayLastElement(uint[] memory array) public pure returns (uint[] memory) {
        assembly {
            let length := mload(array)
            mstore(array, sub(length, 1))
        }
        return array;
    }

    function removeArrayFisrtElement(uint[] memory array) public pure returns (uint[] memory) {
        assembly {
            let length := mload(array)
            array := add(array, 0x20)
            mstore(array, sub(length, 1))
        }
        return array;
    }

    // We know that we only access the array in bounds, so we can avoid the check.
    // 0x20 needs to be added to an array because the first slot contains the
    // array length.
    function sumAsm(uint[] memory _data) public pure returns (uint sum) {
        for (uint i = 0; i < _data.length; ++i) {
            assembly {
                sum := add(sum, mload(add(add(_data, 0x20), mul(i, 0x20))))
            }
        }
    }

    /**
     * Intput: "dave", true, [1,2,3]
     * Output:
     * 0x1e8896de (这是由 samData(string,bool,uint[]) 得出的)
     * 0000000000000000000000000000000000000000000000000000000000000060 （第一个参数(动态类型)的数据部分的位置，从参数块开始以字节为单位测量。在这种情况下为 `0x60`。）
     * 0000000000000000000000000000000000000000000000000000000000000001 （第二个参数：布尔值 true）
     * 00000000000000000000000000000000000000000000000000000000000000a0 （第三个参数（动态类型）的数据部分的位置，以字节为单位测量。在这种情况下为 `0xa0`）
     * 0000000000000000000000000000000000000000000000000000000000000004 （第一个参数的数据部分，它以字节数组的元素长度开始，在本例中为 4）
     * 6461766500000000000000000000000000000000000000000000000000000000 （第一个参数的内容： `"dave"` 的 UTF-8（在这种情况下等于 ASCII）编码，在右侧填充到 32 个字节）
     * 0000000000000000000000000000000000000000000000000000000000000003 （第三个参数的数据部分，它以元素数组的长度开始，在本例中为 3）
     * 0000000000000000000000000000000000000000000000000000000000000001 （第三个参数的第一个元素）
     * 0000000000000000000000000000000000000000000000000000000000000002 （第三个参数的第二个元素）
     * 0000000000000000000000000000000000000000000000000000000000000003 （第三个参数的第三个元素）
     */
    function samData(string memory, bool, uint[] memory) public pure returns (bytes memory) {
        return msg.data;
    }
}
```



### Yul

Yul（以前也称为 JULIA 或 IULIA）是一种中间语言，可以编译为不同后端的字节码。它已经可以在独立模式下使用，也可以用于 Solidity 内部的“内联汇编”。

汇编器执行的唯一非局部操作是用户定义标识符（函数、变量等）的名称查找和从对战中清除局部变量。

为了避免值和引用等概念之间的混淆，Yul 是静态类型的。同时，还有一个默认类型（通常是目标机器的整数字），总是可以省略以提高可读性。

Yul 目前只定义了类型 `u256` ，这是 EVM 的原生 256 位类型。因此，我们不会在下面的示例中提供类型。

#### 简单示例

以下示例程序用 EVM 方言编写并计算幂。

```solidity
{
    function power(base, exponent) -> result
    {
        switch exponent
        case 0 { result := 1 }
        case 1 { result := base }
        default
        {
            result := power(mul(base, base), div(exponent, 2))
            switch mod(exponent, 2)
                case 1 { result := mul(base, result) }
        }
    }
}
```



也可以使用 for 循环而不是递归来实现相同的功能。在这里，计算是否小于。



```solidity
{
    function power(base, exponent) -> result
    {
        result := 1
        for { let i := 0 } lt(i, exponent) { i := add(i, 1) }
        {
            result := mul(result, base)
        }
    }
}
```



#### 句法

Yul 以与 Solidity 相同的方式解析注释、文字和标识符，因此您可以例如使用 `//` 和 `/* */` 来表示注释。例外是 Yul 的标识符中可以包含点 `.` 。

此代码部分始终由花括号分隔的块组成。

在代码块中，可以使用以下元素（有关详细信息，请参阅后面的部分）：

- 字面量，即`0x123`，`42`或`"abc"`（最多 32 个字符的字符串）
- 调用内置函数，例如`add(1, mload(0))`
- 变量声明，例如，`let x := 7` ，`let x := add(y, 3)` 或`let x`（初始值为 0）
- 标识符（变量），例如`add(3, x)`
- 赋值，例如`x := add(y, 3)`
- 在块内定义的局部变量，例如`{ let x := 3 { let y := add(x, 1) } }`
- if 语句，例如`if lt(a, b) { sstore(0, 1) }`
- switch 语句，例如`switch mload(0) case 0 { revert() } default { mstore(0, 1) }`
- for 循环，例如`for { let i := 0} lt(i, 10) { i := add(i, 1) } { mstore(i, 7) }`
- 函数定义，例如`function f(a, b) -> c { c := add(a, b) }`

多个句法元素可以相互跟随，只需用空格隔开，即不需要终止符 `;` 或换行符。



## 优化器



## 引用

- https://docs.soliditylang.org/en/v0.8.11/