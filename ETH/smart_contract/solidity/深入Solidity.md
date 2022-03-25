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

### 映射和动态数组

由于它们的大小无法预测，映射和动态数组不能存储在它们上下的状态变量中间。相反，它们根据上述规则占据 32 字节，它们存储的元素从一个由 `Keccak-256` 计算出的新的存储槽开始。

假设，在使用存储布局规则后，映射或动态数组的存储位置最终是存储槽 `p`。对于动态数组，这个存储槽存储数组中的元素个数（byte 数组和字符串除外）。对于映射，这个存储槽一直是空的，但它仍然需要确保，即便有两个相邻的映射，它们的内容最终在不同的存储槽位置。

数组数据在 `keccak245(p)` 存储槽开始，并且以与定长数组相同的方式排列：一个元素挨一个元素，如果元素长度不超过 16 字节，可能共享同一个存储槽。动态数组的动态数组递归地应用此规则。元素 `x[i][j]` 的位置（`x`的类型为 `uint24[][]` ）由计算方法如下（假定 `x` 本身存储在槽 `p`）：

Slot = `keccak256(keccak256(p) + i) + floor(j / floor(256 / 24))`

the data of element = `(v >> ((j % floor(256 / 24)) * 24)) & type(uint24).max` 其中 `v` 是存储槽中的数据。

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

## 清除变量

## 合约元数据

## 合约 ABI 详述



## 引用

- https://docs.soliditylang.org/en/v0.8.11/