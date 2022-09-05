# Solidity 存储布局

## 简介

理解存储布局后，就能够知道 Solidity 状态变量是如何存储到 EVM 的 Storage 中的。Storage 在 EVM 中以 key-value 对的形式存储持久化的状态，key、value 都被约定为固定的 32 字节大小，整存整取。Solidity 利用其独特的编码方式，将 32 字节的 Storage 转变为复杂的数据类型。



了解存储布局的用处：

1. 在状态变量未设置为 `public` 时，仍能获取到其数据
2. 优化存储以减少 gas 开销
3. 使用内联汇编直接操作 storage
4. 有助于理解类似于 [0x protocol proxy](https://github.com/0xProject/protocol/blob/development/docs/architecture/proxy.rst) 这样的可升级代理的原理



## 工作原理

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

#### 映射

映射的 key `k` 对应存储的数据的位置计算方式： `keccak256(h(k) . p)`，其中 `.` 是连接符，`h` 是一个函数，它根据 `k` 的类型采用不同的手段：

- 对于值类型，`h` ，与在内存中存储值时相同的方式，把值拉长为 32 字节。
- 对于字符串和字节数组，`h` 计算未拉长数据的 `keccake256` hash。

如果映射的值不是值类型，计算出的存储槽标记为数据的开始。如果值为结构体类型，例如，你必须为每个结构体成员添加相应的偏移量（offset）。



## 例子

### 内联汇编操作 storage

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.0;

library Storage {
    function getStorageAt(uint slot) public view returns (bytes32 ret) {
        assembly {
            ret := sload(slot)
        }
    }

    function setStorageAt(uint slot, uint data) public returns (bytes32 ret) {
        assembly {
            sstore(slot, data)
            ret := sload(slot)
        }
    }
}

contract UseStorage {
    uint public a = 1; // slot 0
    uint public b = 2; // slot 1
    uint immutable public c; // not stored at storage
    uint constant public d = 4; // not stored at storage

    constructor() {
        c = 3;
    }

    function getStorageAt(uint slot) public view returns (bytes32 ret) {
        return Storage.getStorageAt(slot);
    }

    function setStorageAt(uint slot, uint data) public returns (bytes32 ret) {
        return Storage.setStorageAt(slot, data);
    }
}
```



### 合约继承

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.0;

import './UseStorage.sol';

contract A {
    uint a = 1; // slot 0 for contract D. slot 2 for contract E.
    uint128 b = 2; // slot 1 for contract D slot 3 for contract E.
    uint128 c = 3;
}

contract B {
    uint d = 4; // slot 2 for contract D. slot 0 for contract E.
    uint e = 5; // slot 3 for contract D. slot 1 for contract E.
}

contract C {
    function getStorageAt(uint slot) public view returns (bytes32 ret) {
        return Storage.getStorageAt(slot);
    }

    function setStorageAt(uint slot, uint data) public returns (bytes32 ret) {
        return Storage.setStorageAt(slot, data);
    }
}

contract D is A, B, C {
    uint f = 6; // slot 4
}

contract E is B, A , C{
    uint f = 6; // slot 4
}
```



### 理解布局



```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.0;

contract A {
    uint a = 1; // slot 0
    uint128 b = 2; // slot 1
    uint128 c = 3;
}

contract B {
    uint d = 4; // slot 2
    uint e = 5; // slot 3
}

contract C is A, B {
    struct S {
        uint128 a; // slot 9
        uint128 b;
        uint[2] staticArray; // slot 10
        uint[] dynArray; // slot 12
    }

    uint x; // slot 4
    uint8 y; // slot 5
    uint16 z;
    uint[2] staticArray; // slot 6
    uint128[] dynArray; // slot 8

    S public s; // slot 9
    address public addr; // slot 13
    mapping (uint => mapping (address => bool)) public map; // slot 14
    string public s1; // slot 15
    bytes public b1; // slot 16
    mapping(uint => bool) public map2; // slot 17
    mapping(bytes => bool) public map3; // slot 18
    bool public bo1; // slot 19
    bool public bo2;
    bool public bo3;
    mapping(bytes4 => address) public map4; // slot 20

    constructor(bytes memory b_) {
        x = 6;
        y = 7;
        z = 8;
        staticArray = [uint(9), uint(10)];
        dynArray = new uint128[](3);
        dynArray[0] = 11;
        dynArray[1] = 12;
        dynArray[2] = 13;

        s = S({
            a: 14,
            b: 15,
            staticArray: [uint(16), uint(17)],
            dynArray: new uint[](3)
        });

        s.dynArray[0] = 18;
        s.dynArray[1] = 19;
        s.dynArray[2] = 20;
        addr = msg.sender;
        map[0][msg.sender] = true;
        map[3][msg.sender] = true;
        
        s1 = "hello";
        b1 = b_;

        map2[0] = true;
        map2[4] = true;

        map3[b1] = true;
        // b1 = new bytes(3);
        // b1[0] = 0x12;
        // b1[1] = 0x34;
        // b1[2] = 0x56;
        // b1 = 0x1234567812345678123456781234567812345678123456781234567812345678
        bo1 = true;
        bo2 = false;
        bo3 = true;

        map4[bytes4(0x6af479b2)] = msg.sender;
    }

    function hashSlot(uint slot) public pure returns (bytes32) {
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

    function getMapLocation(uint slot, uint key) public pure returns (bytes32) {
        return bytes32(keccak256(abi.encode(key, slot)));
    }

    function getMapLocation2(uint slot, bytes memory key) public pure returns (bytes32) {
        return bytes32(keccak256(abi.encodePacked(keccak256(key), slot)));
    }

    function abiEncode(uint slot, bytes4 key)  public pure returns (bytes memory) {
        return abi.encode(key, slot);
    }

    function getMap4Location(uint slot, bytes4 key) public pure returns (bytes32) {
        return bytes32(keccak256(abi.encode(key, slot)));
    }
}
```



- [x] Execute evm bytecode
- [x] Decode transaction logs to events
- [x] Decode state variables from storage layout gengerated by solc
- [x] Decode function arguments of contract creation and contract interaction
- [x] Decode any state variable by given slot, offset, length and something else

### 通过 web3 获取 storage 数据

```bash
$ geth attach http://localhost:8545
# console
> web3.eth.getStorageAt('0x086A75A13699B75604eF57AC1e564B810eb7e955', 0)
> "0x01"
```



### 0x protocol 的 proxy 存储布局

proxy 合约地址 0xDef1C0ded9bec7F1a1670819833240f027b25EfF

它利用 StorageBucket 的设计模式，将全局唯一的 StorageId 映射到一个 slot 上。并且让 StorageId 的 slot 之间间隔 2^128，这意味着一个 Bucket 有连续的长度为 2^128 的 storage 可用，以此来避免所有的 feature 合约使用同一个 proxy 合约的存储上下文带来的 storage 容易冲突的情况。

**LibStorage.sol**:

```solidity
pragma solidity ^0.6.5;
pragma experimental ABIEncoderV2;


/// @dev Common storage helpers
library LibStorage {

    /// @dev What to bit-shift a storage ID by to get its slot.
    ///      This gives us a maximum of 2**128 inline fields in each bucket.
    uint256 private constant STORAGE_SLOT_EXP = 128;

    /// @dev Storage IDs for feature storage buckets.
    ///      WARNING: APPEND-ONLY.
    enum StorageId {
        Proxy,
        SimpleFunctionRegistry,
        Ownable,
        TokenSpender,
        TransformERC20
    }

    /// @dev Get the storage slot given a storage ID. We assign unique, well-spaced
    ///     slots to storage bucket variables to ensure they do not overlap.
    ///     See: https://solidity.readthedocs.io/en/v0.6.6/assembly.html#access-to-external-variables-functions-and-libraries
    /// @param storageId An entry in `StorageId`
    /// @return slot The storage slot.
    function getStorageSlot(StorageId storageId)
        internal
        pure
        returns (uint256 slot)
    {
        // This should never overflow with a reasonable `STORAGE_SLOT_EXP`
        // because Solidity will do a range check on `storageId` during the cast.
        return (uint256(storageId) + 1) << STORAGE_SLOT_EXP;
    }
}
```

**LibOwnableStorage.sol**:

```solidity
/*

  Copyright 2020 ZeroEx Intl.

  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.

*/

pragma solidity ^0.6.5;
pragma experimental ABIEncoderV2;

import "./LibStorage.sol";


/// @dev Storage helpers for the `Ownable` feature.
library LibOwnableStorage {

    /// @dev Storage bucket for this feature.
    struct Storage {
        // The owner of this contract.
        address owner; // slot 1020847100762815390390123822295304634368
    }

    /// @dev Get the storage bucket for this contract.
    function getStorage() internal pure returns (Storage storage stor) {
        uint256 storageSlot = LibStorage.getStorageSlot(
            LibStorage.StorageId.Ownable
        );
        // Dip into assembly to change the slot pointed to by the local
        // variable `stor`.
        // See https://solidity.readthedocs.io/en/v0.6.8/assembly.html?highlight=slot#access-to-external-variables-functions-and-libraries
        assembly { stor_slot := storageSlot }
    }
}
```



例如对于 Ownable feature 来说，它的 storage 开始位置为 (2+1) << 128：

```python
$ python3
>>> (2+1)<<128
1020847100762815390390123822295304634368
```

其 owner 变量的值为：

```bash
# console
> web3.eth.getStorageAt('0xDef1C0ded9bec7F1a1670819833240f027b25EfF', '1020847100762815390390123822295304634368')
> "0x000000000000000000000000618f9c67ce7bf1a50afa1e7e0238422601b0ff6e"
```

