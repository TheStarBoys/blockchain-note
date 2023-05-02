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

以 OpenZeppelin 的 [Merkle Tree 实现](https://github.com/OpenZeppelin/merkle-tree/blob/HEAD/src/core.ts)，深度解析 Merkle Tree 及其证明的生成的**关键点**。

通过上文讲到的，双重散列每个叶子结点来避免二次预像攻击。

其生成的 Merkle Tree 是一颗完全二叉树，而不要求是满二叉树。目的是当叶子结点个数为奇数时，不用为了将叶子结点个数凑成偶数，复制其中一个叶子结点。还是以之前的图为例，考虑只有 L1、L2、L3 时，为了这个算法能正常工作，我们将 L3 复制一份放到 L4 的位置，就可以仍然按照一颗满二叉树来处理了，并且我们进行了 6 次散列运算（我们忽略叶子结点是由双重散列数据块得到的。从数据块到叶子结点共3次，hash(L4)是复制hash(L3)得到的）：

![merkle tree](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/hash_tree-svg.png)

同样是 3 个叶子结点，现在我们来考虑按照完全二叉树来处理的情况。其中，数字表示每个结点的编号且 2、3、4 为叶子结点。并且只进行了 5 次散列运算。

```
[0 1 2 3 4]
		 0
	 /	 \
  1		  2
 / \
3   4
```

因此 OpenZeppelin 这样实现的好处是：

- 减少一次数据拷贝
- 减少一次散列运算
- 可以利用完全二叉树的性质，用数组来实现（而不是链表）以减少空间复杂度



Merkle Tree 生成的关键代码：

```typescript
const hashPair = (a: Bytes, b: Bytes) => keccak256(concatBytes(...[a, b].sort(compareBytes)));
const leftChildIndex  = (i: number) => 2 * i + 1;
const rightChildIndex = (i: number) => 2 * i + 2;

export function makeMerkleTree(leaves: Bytes[]): Bytes[] {
  leaves.forEach(checkValidMerkleNode);

  if (leaves.length === 0) {
    throw new Error('Expected non-zero number of leaves');
  }

  const tree = new Array<Bytes>(2 * leaves.length - 1);

  for (const [i, leaf] of leaves.entries()) {
    tree[tree.length - 1 - i] = leaf;
  }
  for (let i = tree.length - 1 - leaves.length; i >= 0; i--) {
    tree[i] = hashPair(
      tree[leftChildIndex(i)]!,
      tree[rightChildIndex(i)]!,
    );
  }

  return tree;
}
```

其中，`hashPair` 函数并不是简单的拼接两个散列值，而是先排序后拼接。这表明，对于有相同父节点的两个孩子节点，它们的相对顺序是不重要的。这一点很重要，这能够方便实现后面的证明生成与验证。

它的证明是必要结点的序列，假设我们提供数据块 L3，那么我们还需要节点 Hash1-1、Hash0 才能计算得到根散列值。而其证明在这个特例中就是 `[Hash1-1, Hash0]`。

```typescript
const parentIndex     = (i: number) => i > 0 ? Math.floor((i - 1) / 2) : throwError('Root has no parent');
const siblingIndex    = (i: number) => i > 0 ? i - (-1) ** (i % 2)     : throwError('Root has no siblings');

export function getProof(tree: Bytes[], index: number): Bytes[] {
  checkLeafNode(tree, index);

  const proof = [];
  while (index > 0) {
    proof.push(tree[siblingIndex(index)]!);
    index = parentIndex(index);
  }
  return proof;
}
```

这段代码就是生成证明，从叶子结点开始，向上遍历，拿到对应的兄弟节点后，加入 `proof` 数组。以生成 L3 的证明为例，其证明为  `[Hash1-1, Hash0]` 。



下面这段代码，是传入叶子结点以及证明，计算其根散列值：

```typescript
export function processProof(leaf: Bytes, proof: Bytes[]): Bytes {
  checkValidMerkleNode(leaf);
  proof.forEach(checkValidMerkleNode);

  return proof.reduce(hashPair, leaf);
}
```

可以看到，计算根散列值很简单，`proof.reduce(hashPair, leaf)` 这一行代码就解决了。我们以 L3 的证明为例，拆解一下这个过程：

- leaf 在这里为 `Hash1-0`，即 `hash(L3)`。证明为 `[Hash1-1, Hash0]` 
- 第一步：`hashPair(leaf, Hash1-1)` => `hashPair(Hash1-0, Hash1-1)` => `Hash1`
- 第二步：`hashPair(Hash1, Hash0)` => `Top Hash`



这个例子看不出 `hashPair` 这个函数的妙处，我们再以 L4 的证明为例拆解：

- leaf 在这里为 `Hash1-1`，即 `hash(L4)`。证明为 `[Hash1-0, Hash0]` 
- 第一步：`hashPair(leaf, Hash1-0)` => `hashPair(Hash1-1, Hash1-0)` => `Hash1`
- 第二步：`hashPair(Hash1, Hash0)` => `Top Hash`

很明显，在第一步出现了差异，`hashPair` 的两个参数位置互换了，但最终结果没变。因此 `proof` 只需要是数组即可，不需要关心两个兄弟节点之间的相对位置。



## Open-Source Implementations

- [OpenZeppelin/merkle-tree](https://github.com/OpenZeppelin/merkle-tree)



## Referrences

[1] [attacking-merkle-trees-with-a-second-preimage-attack/](https://flawed.net.nz/2018/02/21/attacking-merkle-trees-with-a-second-preimage-attack/)

[2] [Wikipedia: Preimage Attack](https://en.wikipedia.org/wiki/Preimage_attack)

[3] [OpenZeppelin/merkle-tree](https://github.com/OpenZeppelin/merkle-tree)