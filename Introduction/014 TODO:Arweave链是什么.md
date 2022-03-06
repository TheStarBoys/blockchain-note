# Arweave 链是什么

## 简介

传统的区块链技术的数据存储成本是高昂的，往往需要第三方协议进行集成，以减少链上存储开销。因此一个低成本的分布式数据存储协议是必要的。Arweave 链，一个被称为 block-weave 的类区块链结构，被设计以提供一个可扩展的**低成本链上存储**，以及提供 REST API 让应用程序更容易在 Arweave 上层构建。采用 Proof of Access（PoA）共识算法以提高效率[1]。



目前的痛点

1. 中心化服务器能**随意修改/删除数据**、**撤销用户的访问权限**
2. 网站可能由于资金不足而关闭
3. 许多政府正**加强监管**并移除网络中的**政治敏感**的信息
4. 现在大多数区块链技术要求“全节点”必须维护**整个区块链的副本**以验证未来的交易

实现 Arweave 的动机

1. 让数据永存成为现实
2. 通过 REST API，为现有互联网提供其极度需要但缺失的数据永久存储
3. 让在其上构建的 app 的功能更广泛与不同
4. 提交数据到 Arweave 上需要小额的 token，以拒绝“垃圾信息”传播
5. 随着网络和文件增强 token 价值，维护 weave 的激励也会增加
6. 期望结合这些特征后，Arweave token 将成为信息时代的宝贵资产



## 技术

下面这四个核心技术使低成本、高吞吐量、永久存储在 Arweave 链上成为可能：

1. Blockweave
2. Proof of Access
3. Wildfire
4. Blockshadows

### Blockweave

Arweave 引入两个新概念使得节点在没有处理整个区块链的情况下实现核心的网络功能。

1. 区块哈希列表，所有前区块的哈希的列表。这意味着旧的区块能被验证，并且更高效的挖出潜在的新区块
2. 系统中活跃钱包的列表。这允许在没拥有前一笔交易所在区块时验证该交易。

### Proof of Access

Arweave 的共识机制基于 Proof of Acess （PoA）与 Proof of Work（PoW）。

`recall block` 将被包含进下一个区块，通过 **当前区块的哈希值 % 当前区块高度** 来选择 `recall block`。

当矿工找到了有效的哈希值（PoW）时，将挖出的新区块和 `recall block` 一起发给其他节点。这允许没有 `recall block` 的节点也能独自验证新区块是否有效。

### Wildfire

Wildfire 是一个，通过使网络数据请求的快速满足成为参与的必要部分，来解决分布式网络中的数据共享问题的系统。它拥有一个根据响应请求、接收数据的速度来排名的排名系统，节点按照排名顺序服务，表现差的节点将被网络列入黑名单，以鼓励矿工免费地分享数据。

### Blockshadows

Blockshadows 不仅将最大限度的减少数据浪费，还能加快区块共识速度与增加交易吞吐量。

### 智能合约

[Introducing SmartWeave: Building smart contracts with Arweave](https://arweave.medium.com/introducing-smartweave-building-smart-contracts-with-arweave-1fc85cb3b632)

[Arweave Smart Contracts Now Support Solidity, Rust And C: Everything We Know About 3em](https://arweave.news/3em-smart-contracts/)

## 参考

[1] [Arweave lightpaper](https://www.arweave.org/files/arweave-lightpaper.pdf)