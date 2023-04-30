# 常见DeFi合约介绍

## 简介

一般来说常见的合约有以下几种：

- Token（代币，通证）
  - ERC-20
  - ERC-721(NFT，非同质化代币，数字藏品)
- Design Pattern（设计模式）
  - Implementation Contract
  - Factory
  - Stateless Contract
- DeFi Contracts（DeFi合约）
  - Liquidity Pool(流动池)
  - Farming(Liquidity Mining, Yield Farming，流动性挖矿)
  - Flashloan(Flash Swap，闪电贷)
  - Oracle
- Governance（治理）
  - Governance Contract
  - Off-chain Governance

本文会根据它的用例来展开讲解，并且讲解一些注意事项。



## Token

### ERC-20

ERC-20 是最常见的代币合约标准，目的是为了让 DeFi 项目有一个统一的代币标准，以获得更好的互操作性，例如，在协议（项目）A中获得的代币，能够在协议B中进行使用（stake、swap、farming）。其最重要的接口如下：

```solidity
interface IERC20 {
		// 该接口属于扩展接口，并不是 ERC-20 强制要求实现的一部分。
		function symbol() external view returns (string memory);
		
    function transfer(address recipient, uint256 amount) external returns (bool);

    function approve(address spender, uint256 amount) external returns (bool);

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external returns (bool);
}
```

ERC-20 是一个合约标准（接口），实现了该标准的合约都被认为是 ERC-20。例如，合约 A 实现了 ERC-20 标准，那么合约 A 本身就是 ERC-20 代币，它的 `symbol` 接口返回一个人类可读形式的代币名称，例如 USDT。

`transfer` 接口作用是给 `recipient` 地址转 `amount` 数额的代币。

`approve` 接口作用是授权给 `spender` 能够花费对应 `amount` 数额的代币。

`transferFrom` 接口作用是从 `sender` 处转移一笔 `amount` 数额的代币到 `recipient` 处，调用者必须拥有足够的通过 `approve` 接口授权的数额的代币，转移成功后，会扣除相应的数额。



常见的流程：

1. 用户 A 给用户 B 转账：直接调用 `transfer` 接口进行转账即可。
2. 用户 A 希望质押代币到合约 B 中，此时需要用户 A 先 `approve` 授权给合约 B，随后合约 B 调用 `transferFrom` 进行转账。这样做的目的是为合约提供统一使用 ERC-20 代币的方式；`approve` 足够数额的代币后，后续的质押只需要调用一次合约接口。

应用：

在 etherscan 中查看 ERC-20：

![image-20230312154958674](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20230312154958674.png)



在 Metamask 中转 ERC-20 代币：

![image-20230312155218223](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20230312155218223.png)

### ERC-721

ERC-721 是非同质化代币标准，简称 NFT。与 ERC-20 不同的是，它使用唯一的 `tokenId` 标识唯一的一个 NFT 资产，不可分割，一般用于表征数字藏品的稀缺性，并进行确权，例如，OpenSea上拍卖的艺术作品和 CryptoCat 里每只独一无二的猫。

其最重要的接口如下：

```solidity
interface IERC721 {
    function ownerOf(uint256 tokenId) external view returns (address owner);

    function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) external;

    function approve(address to, uint256 tokenId) external;

    function setApprovalForAll(address operator, bool _approved) external;
}
```



`ownerOf` 接口用于进行确权，确认这个唯一的 NFT 的所有者。

`transferFrom` 接口用于转移 `tokenId` NFT 从 `from` 到 `to`。如果调用者不是 `from`， 那么它必须被授权以转移对应 NFT，使用 `approve` 或 `setApprovalForAll` 接口均可授权。

`approve` 接口用于给 `to` 地址进行授权，使其能够转移调用者的 `tokenId` 对应的 NFT。

`setApprovalForAll` 接口用于给 `operator` 地址授权，使其能够转移调用者的**所有**NFT。



常见流程：

1. 用户 A 给用户 B 转一个 NFT：使用 `transferFrom` 方法，将 `from` 填成用户 A 的地址。
2. 用户 A 在合约 B 中进行质押：首先使用 `approve` 或 `setApproveForAll` 方法给合约 B 授权，然后合约 B 使用 `transferFrom` 方法转移用户 A 的 NFT。



应用：

OpenSea 上的某一个 NFT：

![image-20221122152926730](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/upload_1669102170144image-20221122152926730.png)

![image-20221123140915841](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/upload_1669183757366image-20221123140915841.png)

从 etherscan 上能查到对应 NFT 的所有者：

![image-20221122153204368](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/upload_1669102325842image-20221122153204368.png)

![image-20221123140858278](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/upload_1669183739379image-20221123140858278.png)

## Design Pattern

### Implementation Contract

Implementation 合约（又称 logic 合约，逻辑合约），顾名思义，它仅作为代码被部署到区块链网络，并不会直接使用其存储。一般用于 Proxy 合约和 Factory 合约中。

在 Proxy 合约中，Implementation 合约充当代码具体的执行逻辑，不会使用其存储，Proxy 合约是对外的入口，将所有的代码逻辑转发到 Implementation 合约中进行处理，并使用 Proxy 自身的存储，从而方便通过更新 Implementation 合约进行合约升级，以及减少部署成本。

例如，在部署逻辑合约后，想要部署多个相同功能的合约，仅需要部署多次 Proxy 合约（其均指向逻辑合约）即可，它只会在后续花费部署 Proxy 合约的 gas 成本。



### Factory

在上面讲 Proxy 合约提到了多次部署，为了方便管理，一般会有一个专门的 Factory 合约来部署。一般有两种部署的方式：

- 每次创建新合约时，都是将 implementation 合约的代码拷贝到新合约中，gas 成本高。
- 每次创建新合约时仅创建 Proxy 合约，其指向 Implementation 合约，以节省 gas 开销。



### Stateless Contract

它仅提供合约交互的逻辑，并不存储任何状态。其底层的合约负责核心的算法运算、以及状态存储，无状态合约来提供对外访问的接口，从而在升级时，仅需替换无状态合约即可。



## DeFi Contracts

### Liquidity Pool

Liquidity Pool（又称流动池），一般是用于为 swap 或者跨链 swap 提供流动性，为 swap 添加流动性需要一个代币对。如果某一个代币对相应的流动池不存在，需要先创建流动池。

一般有以下三种操作：

- Add Liquidity（添加流动性）
- Remove Liquidity（移除流动性）
- Get Reward（获取提供流动性的分红）



#### Add Liquidity

为流动池添加流动性的用户（又称 Liquidity Provider，流动性提供者）将能够得到对应的 LP Token，并能从流动池中的兑换产生的手续费中进行分红（根据持有 LP Token 的占比）。但其缺点是有 Impermanent Loss（无常损失）。

为 swap 提供流动性的流动池，一般要求提供两种及以上的代币，例如 ETH/USDT。

为跨链 swap 提供流动性的流动池，一般是在不同链上提供相同的代币，例如在链 A 和链 B 提供 ETH（不用同时在不同链上添加流动性）。



![image-20221122164541972](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/upload_1669106744841image-20221122164541972.png)

#### Remove Liquidity

移除流动性将需要调用者提供相应的 LP Token，流动池将按照其比例释放相应数量的质押代币（ETH/USDT）。

#### Get Reward

无常损失是由于代币对（ETH/USDT）之间的汇率波动导致的提供流动性的收益不如单纯持有对应代币，因此需要给流动性提供者进行分红作为提供流动性的补偿。



#### UniswapV2 Router

UniswapV2 Router 也是流动池的实现之一，但它还提供了 swap 接口。它是典型的无状态合约，从它的合约代码中可以看出这一点，其中并没有状态变量（factory和WETH均为常量，在构造函数里初始化后无法修改）。

![image-20221118142732051](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/upload_1668752854015image-20221118142732051.png)

其中一个 swap 函数如下：

```solidity
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint[] memory amounts);
```

其参数分别为：卖出代币的数量 `amountIn` 、期望最少获得的目标代币数量 `amountOut` 、兑换的路径 `path`、代币接受地址 `to` ，以及过期时间 `deadline` 。

假设说你需要用 WETH 兑换 DAI，此时没有相应的 WETH/DAI 流动池，但有两个池子，WETH/USDT 和 USDT/DAI，你可以尝试构造一个兑换路径 WETH -> USDT -> DAI，达成兑换目的。



### Farming

![image-20221118155246555](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/upload_1668757968789image-20221118155246555.png)

为了额外的激励用户为流动池提供流动性，一些项目方（协议）会提供流动性挖矿功能。主要有以下几个操作：

- Stake/Deposit（质押）
- Unstake/Withdraw（取消质押、取款）
- Get Reward

流动性挖矿要求你质押（stake）对应的 LP Token，从而在接下来每过一个区块时间都能累积流动性挖矿奖励。

调用 Unstake 可以将质押的 LP Token 取回。



### FlashLoan

FlashLoan（闪电贷）利用了交易的原子性，在一个交易里借出资产，并要求在交易结束前归还资产以及利息。它实现了在不需要抵押任何资产的情况下，让实现了相应接口的第三方合约借入资产，并在交易结束前还回，只需要支付 gas fee。

其用途一般是用于套利交易，例如在 Uniswap 中，购买 1 ETH 的成本是 200 DAI，而在 Oasis（或任何其他交易场所）上，1 ETH 购买 220 DAI。对于任何拥有 200 DAI 的人来说，这种情况代表了 20 DAI 的无风险利润。不幸的是，你身边可能没有 200 DAI。然而，通过闪电贷，任何人都可以获得这种无风险的利润，只要他们能够支付 gas 费。



### Oracle

oracle 从区块链(链下)数据源中获取数据，并将其放在区块链(链上)上，以供智能合约使用。因为在以太坊上运行的智能合约不能访问存储在区块链网络之外的信息。去中心化预测市场依靠 oracle 提供有关结果的信息，他们可以用这些信息来验证用户的预测。比如，预测球赛的获胜方，因此需要有 oracle 将链下的球赛胜负信息推送到链上，链上的合约才能根据此判定本次预测的胜利方是谁。

![image-20221122111144274](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/upload_1669086705674image-20221122111144274.png)

总的来说，oracle 分为链上合约部分和链下服务部分，链上合约负责收集链下服务推送的数据，并将相应数据反馈给其他合约；链下服务负责收集链下的相关信息，在达成共识后，将数据推送到链上合约。

根据链下共识的不同，又可以分为去中心化的 oracle 和中心化的 oracle。去中心化的 oracle 有多个数据源，并在链下服务中对某一个结果达成共识后推送到链上合约。中心化的 oracle 往往是项目方控制，其数据容易受到项目方的操控。

比较常见的 oracle 的应用场景是将链下的加密货币兑换率推送到链上，例如 ETH/USD 的价格。提供这一服务的常见协议是 ChainLink。

![image-20221122153343085](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/upload_1669102424995image-20221122153343085.png)

## Governance

治理可以分为链上治理和链下治理两个大类，有些协议会只采用其中一种，有些协议会两者相结合，比如链下治理去收集热度，链上治理会最终转为可执行的链上代码，**去调整协议参数或执行其他代码**。

根据投票权的表现形式不同，链上治理又分为：

- 基于 Token 投票的治理
- 基于股份投票的治理

基于 Token 投票的治理没有准入门槛，可以由多种途径获得治理 Token，例如从一个 DEX（去中心化交易所）中兑换。

基于股份投票的治理需要获得更多许可，但仍然相当开放。任何潜在成员都可以提交加入治理的提案，通常以代币或工作的形式提供一定价值的贡品。股份代表直接投票权和所有权。所有股份都是不可转让的，并且是治理合约独有的。成员可以以其在金库中的比例份额退出。一般是基于 ERC-20 合约，移除其中 `transfer` 相关的接口。

### Governance Contract

治理合约中一个提案的状态切换流程如下：

![](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/gov_diagram-1-9bc9f7797121de9e8c8210d39b1c0dc3.png)

其中投票阶段，一般 1 个 Token 代表 1 投票权。投票权的多少取决于在该提案创建时，你拥有多少 Token。例如，创建提案的时刻，你拥有 100 Token，在投票阶段，你拥有 200 Token，但你的投票权仍然只是 100。

类似的，你可以委托给另一个地址替你投票。在投票阶段结束后，达到法定人数的提案会被认为是成功的提案，进入 Timelock 的等待队列中等待执行。Timelock 合约主要是为了防止一些恶意的提案没有被人注意到，从而导致协议被攻击，因此给一定缓冲时间有利于减弱此影响。

### Off-chain Governance

Snapshot 是链下治理中最热门的服务，可以让你在不需要支付 gas 的情况下，参与治理，一般被用于提案初期的热度收集阶段。Uniswap的链下治理页面看起来像这样：

![image-20221118152117367](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/upload_1668756079599image-20221118152117367.png)



点开提案的详情，可以看到其相应的策略，一般和链上治理类似，以创建提案的时间点的代币数量作为你的投票权：

![image-20221118152159600](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/upload_1668756123180image-20221118152159600.png)

