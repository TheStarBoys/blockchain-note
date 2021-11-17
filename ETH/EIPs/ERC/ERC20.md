# 一文弄懂什么是 ERC20

> 【本文只做技术探讨，谨防数字加密货币炒作风险。】



## Token

Token，即通证，是以数字形式存在的权益凭证，它代表的是一种权利，一种固有和内在的价值。货币、积分、股票等权益证明，都可以由通证来代表。它代表着数字资产。下图就是在 [opensea](https://opensea.io/) 上售卖的一些数字资产，这些资产也是通证。

![image-20211116220121379](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211116220121379.png)



试想，如果这样的一些通证没有标准化，就只能在自己的体系内流通。通证的标准协议是数字资产上链的关键，它定义了不同的通证资产接口，从而可以对不同类型的资产进行交易和交换。



## 什么是 ERC20？

ERC20 就是以太坊生态中的通证（token） 标准，允许实现该标准的任何通证从钱包到去中心化的交易所能够被复用。



举个例子，在以前，公司发行的积分，往往只能够在内部使用，仅代表公司生态内部的权益。而有了通证就不一样了。公司发行通证，对于公司来说，可以分配通证来进行融资，上交易所（类似于上市），激励用户使用公司产品等；对于持有人来说，根据通证持有占比分红，持有的通证可以任意交换，低买高卖赚取差价等。ERC20 为通证的发行、流通提供了统一的标准，以相同的方法发行、交易、交换通证，而不用关心这个通证的发行方将它用来做什么（这取决于发行方）以及怎么实现通证。



任何智能合约只要符合 ERC20 标准，就可以通过 ERC20 标准接口进行操作。这也意味着符合 ERC20 标准的合约 A，名字为 Token A，符号为 A，合约地址为 `0x000..0a`，合约 B，名字为 Token B，符号为 B，合约地址为 `0x000..0b`；A、B 都是 ERC20 通证，转移通证 A 与转移通证 B 在操作上对于用户来说没有任何区别，与下图的操作类似。

![image-20211117002826777](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211117002826777.png)



**值得注意的是，任何人都可以发行一个与合约 A，名字、符号相同的符合 ERC20 标准的合约 C，来冒充合约 A，但合约 C 与合约 A 相比，合约地址是不同的，因此建议在交易时通过合约地址来区分 A、B，而不是简单的通过名字、符号区分。**



下图是某交易所中的交易页面，其中所有的通证都是 ERC20 通证，符合 ERC20 标准，但都有各自的合约地址，并且可以类似股票一样交易。

![image-20211116222745925](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211116222745925.png)



类似的信息也可以在 [etherscan](https://etherscan.io/) 中查看

![image-20211116223750926](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211116223750926.png)



## EIPs 中的定义

EIPs（Ethereum Improvement Proposals），以太坊改进提案，ERC20 便是 EIPs 中的一个标准。

ERC20 标准允许在智能合约中实现通证的标准API。该标准提供了转移通证的基本功能，并允许通证被批准，以便其他链上第三方可以使用它们。



下面是智能合约的接口定义。

```solidity
pragma solidity ^0.8.0;

/**
 * @dev EIP中定义的ERC20标准接口.
 */
interface IERC20 {
    /**
     * @dev 返回存在的通证数量
     */
    function totalSupply() external view returns (uint256);

    /**
     * @dev 返回`account`拥有的通证数量
     */
    function balanceOf(address account) external view returns (uint256);

    /**
     * @dev  从调用者的账户向`recipient`转移`amount`数量的通证
     *
     * 返回布尔值来指出操作是否成功
     *
     * 发出一个 {Transfer} 事件.
     */
    function transfer(address recipient, uint256 amount) external returns (bool);

    /**
     * @dev 返回' spender '将被允许通过{transferFrom}代表' owner '
     * 消费的通证的剩余数量。默认为零。
     *
     * 当 {approve} 或者 {transferFrom} 被调用的时候，这个值会随之改变
     */
    function allowance(address owner, address spender) external view returns (uint256);

    /**
     * @dev 允许`spender`花费`amount`数量的调用者的通证
     *
     * 返回布尔值来指出操作是否成功
     * 发出一个 {Approval} 事件
     */
    function approve(address spender, uint256 amount) external returns (bool);

    /**
     * @dev 使用批准机制，从 `sender` 账户中转移 `amount` 数量的通证到 `recipient` 账户
     * 并从调用者被批准花费的数额中扣除 `amount` 数量
     *
     * 返回布尔值来指出操作是否成功
     *
     * 发出一个 {Transfer} 事件
     */
    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external returns (bool);

    /**
     * @dev 当通证从一个账户 `from` 转移至另一个账户 `to` 时，发出该事件。
     *
     * 注意 `value` 可能是 0
     */
    event Transfer(address indexed from, address indexed to, uint256 value);

    /**
     * @dev 当通过 {approve}，新的批准花费的值被设置的时候，发出该事件。
     * `value` 是新批准花费的值。
     */
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

interface IERC20Metadata is IERC20 {
    /**
     * @dev 返回通证的名字
     */
    function name() external view returns (string memory);

    /**
     * @dev 返回通证的符号
     */
    function symbol() external view returns (string memory);

    /**
     * @dev 返回通证的小数位
     */
    function decimals() external view returns (uint8);
}
```



注意，`IERC20Metadata` 中定义的接口是可选的，但在实践中往往都会实现。以 UNI 为例，它的名字是 Uniswap，符号是 UNI，小数位为 18。



一般而言，有两个 Transfer 事件比较特殊，需要注意一下。一个是在铸币（mint）时触发，由于是凭空产生，所以 `from` 被指定为 `0x0000000000000000000000000000000000000000` 地址；一个是在销毁时触发，`to` 被指定为 `0x0000000000000000000000000000000000000000` 地址，而销毁时的 `to` 地址采用 `0x00...0` 地址是为了便于统计销毁数据的约定做法，如果 `to` 地址是一个任何人都没有对应私钥的地址，仍然属于销毁，但这很难统计。



## 在 etherscan 中的 ERC20

以 BNB 为例，我们在 etherscan 中找到 BNB，并打开。

![image-20211116230240090](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211116230240090.png)



看一下它的概览页，其中 Max Total Supply 来自于接口 `totalSupply`，Decimals 来自于接口 `decimals`。

![image-20211116230336021](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211116230336021.png)



也可以点开 Transfers 查看转账记录，它就是通过 Transfer 事件查询得到的，假如你将你的 BNB 转移给某个人，你也能够通过交易哈希在这上面找到。

![image-20211116231924840](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211116231924840.png)



更可以方便的通过 contract 页面调用智能合约。

![image-20211116232225555](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211116232225555.png)



可以通过这个功能查看通证供应量，只需要轻轻点击 totalSupply 。

![image-20211116232304820](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211116232304820.png)



我们可以通过在右上角搜索框填入 `0x0000000000000000000000000000000000000000` 地址来作为过滤条件，过滤 Transfer 事件，以此来查看历史上的铸币/销毁事件。以 USDT 为例：

![image-20211116233512477](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211116233512477.png)



## 钱包中的 ERC20

以在 MetaMask 中使用 BNB 为例，上面的 etherscan 的 BNB 页面中可以获取到 BNB 合约的地址 `0xB8c77482e45F1F44dE1745F52C74426C631bDD52`，确认自己的 MetaMask 连接向 Ethereum Mainnet。

![image-20211117003927055](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211117003927055.png)



点击下方的 `import tokens`，将 BNB 合约地址粘贴进去，会自动获取 BNB 的相关信息，此时可以再次确认，确保没有倒入错误，点击 `Add Custom Token` 即可添加 BNB 到 MetaMask 中。

![image-20211117004057574](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211117004057574.png)



添加后，将出现在你的资产列表中。

![image-20211117004249967](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211117004249967.png)



点击资产列表中的 BNB，你就可以选择发送一些 BNB 给其他人了！

![image-20211117004347938](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20211117004347938.png)

## 引用

- [百度百科：通证](https://baike.baidu.com/item/%E9%80%9A%E8%AF%81/24281877?fr=aladdin)
- 《Token经济》
- [EIP-20](https://eips.ethereum.org/EIPS/eip-20)

