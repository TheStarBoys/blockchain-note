# Merkle Tree

## What Merkle Tree is



用例：

- 防篡改
- 用于区块链中的空投(Airdrop)
- 零知识证明（Zero-knowledge Proof）



## How it works

![merkle tree](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/hash_tree-svg.png)



## Potential Attacks

在加密散列函数中，有三种常见的攻击，我们将他们放在一起有助于理解[2]。

预像攻击（Preimage Attack）是给定一个散列值，你去找到散列后能够得到该值的任何原始数据。即，给定一个散列值 y，你要找到原始数据 x 使得 `hash(x) == y` 成立。

第二预像攻击（Second Preimage attacks）是给定一个输入，找到另一个输入，它们散列后具有相同的散列值。即，给定原始数据 x，你要找到原始数据 x' 并且 x' != x 使得 `hash(x) == hash(x')` 成立。

碰撞攻击（Collision Attack）是找到两个不同的输入，它们散列后的散列值相同。即，找到 x 和 x' 使得 `hash(x) == hash(x')`。

### 在 Merkle Tree 中的第二预像攻击

看到上面的示例图，我们可以把这颗树的中间层中的每一对相邻节点拼接后，作为另一个颗Merkle树的输入，从而得到相同的根散列值。即，我们将 `hash(0-0) + hash(0-1)` 和 `hash(1-0) + hash(1-1)` 作为另一颗树的输入，所以得到另一颗树的叶子结点为  `hash(hash(0-0) + hash(0-1))` （等同于 Hash 0）和 `hash(hash(1-0) + hash(1-1))` （等同于 Hash 1），最终得到相同的根散列值。也就是说，两组不同的输入数据集合，得到了相同的根散列值，其中，第一组输入数据集合是你已知的，第二组数据集合是你构造出的，完全符合第二预像攻击的定义。



#### 如何修复

这里介绍三种方案，均通过区分叶子结点和中间结点，来避免攻击者直接提供中间值来构造输入[1]：

- 通过在前面添加一些字节，例如在前面添加0x00用于叶子结点，0x01中间结点。下面给出了具体的例子：

```
hash(0x01 + hash(0x00+L1) + hash(0x00+L2))	hash(0x01 + hash(0x00+L3) + hash(0x00+L4))

hash(0x00+L1)		hash(0x00+L2)								hash(0x00+L3)		hash(0x00+L4)

L1							L2													L3							L4
```



- 将树深度和节点深度等信息放入每个节点中
- 双重散列每个叶子结点[3]

```
hash(hash(hash(L1)) + hash(hash(L2))) hash(hash(hash(L3)) + hash(hash(L4)))
hash(hash(L1))		hash(hash(L2))			hash(hash(L3)		hash(hash(L4)

L1								L2									L3							L4
```



## Core Algorithms



## Open-Source Implementations

- [OpenZeppelin/merkle-tree](https://github.com/OpenZeppelin/merkle-tree)



## Referrences

[1] [attacking-merkle-trees-with-a-second-preimage-attack/](https://flawed.net.nz/2018/02/21/attacking-merkle-trees-with-a-second-preimage-attack/)

[2] [Wikipedia: Preimage Attack](https://en.wikipedia.org/wiki/Preimage_attack)

[3] [OpenZeppelin/merkle-tree](https://github.com/OpenZeppelin/merkle-tree)