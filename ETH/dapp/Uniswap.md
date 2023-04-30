# Uniswap

## V2

### 概念

#### 工作原理

![img](https://docs.uniswap.org/assets/images/anatomy-82d82239e5417e36ca9da17d14961434.jpg)

任何人都可以 deposit 相同价值的基础代币来换取池代币，从而成为流动性提供者（LP）。用于追踪你占整个池的资金储备量的比例，并且可以随时赎回资产。

货币对充当自动做市商，只要保留“恒定乘积”公式，就准备好接受一个代币换另一个代币。该公式最简单地表示为`x * y = k`，表明交易不得改变`k`一对储备余额 (`x`和`y`) 的乘积。因为`k`与交易的参考框架保持不变，它通常被称为不变量。该公式具有理想的特性，即较大的交易（相对于储备）以比较小的交易更差的指数执行。

在实践中，Uniswap 对交易收取 0.30% 的费用，该费用被添加到准备金中。结果，每笔交易实际上都在增加`k`。这起到了向 LP 支付的作用，这是在他们烧掉他们的池代币以提取他们在总储备中的一部分时实现的。将来，此费用可能会降低到 0.25%，剩余的 0.05% 将作为协议范围内的费用扣留。

![img](https://docs.uniswap.org/assets/images/trade-2027cdc01fe7c448f60a5e7da34af9b9.jpg)

由于两对资产的相对价格只能通过交易来改变，Uniswap 价格与外部价格的背离创造了套利机会。这种机制确保 Uniswap 价格始终趋向于市场出清价格。

#### 生态参与者

![img](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/participants-a3e150f3c98a0b402c2063de3e160f2e.jpg)

**流动性提供者**

流动性提供者或 LP 不是同质群体：

- 被动 LP 是希望被动投资资产以积累交易费用的代币持有者。
- 专业的 LP 专注于做市作为他们的主要策略。他们通常开发定制工具和方法来跟踪他们在不同 DeFi 项目中的流动性头寸。
- 代币项目有时会选择成为 LP 为其代币创建一个流动的市场。这使得代币买卖更容易，并通过 Uniswap 解锁与其他 DeFi 项目的互操作性。
- 最后，一些 DeFi 先驱正在探索复杂的流动性提供互动，例如激励流动性、流动性作为抵押品以及其他实验性策略。Uniswap 是项目尝试这些想法的完美协议。

**交易员**

协议生态系统中有几类交易者：

- 投机者使用各种社区构建的工具和产品，利用从 Uniswap 协议中提取的流动性来交换代币。
- 套利机器人通过比较不同平台的价格以寻找优势来寻求利润。（虽然看起来很榨取，但这些机器人实际上有助于平衡更广泛的以太坊市场的价格并保持公平。）
- DAPP 用户在 Uniswap 上购买代币，用于以太坊上的其他应用程序。
- 通过实现交换功能（从 DEX 聚合器等产品到自定义 Solidity 脚本）在协议上执行交易的智能合约。

在所有情况下，根据协议进行的交易均需支付相同的固定费用。每一项对于提高价格的准确性和激励流动性都很重要。

**开发商/项目**

在更广泛的以太坊生态系统中使用 Uniswap 的方式太多了，但一些例子包括：

- Uniswap 的开源、可访问性意味着有无数的 UX 实验和前端旨在提供对 Uniswap 功能的访问。您可以在大多数主要的 DeFi 仪表板项目中找到 Uniswap 功能。社区还构建了许多[特定于 Uniswap 的工具](https://github.com/Uniswap/universe)。
- 钱包通常将交换和流动性提供功能作为其产品的核心产品。
- DEX（去中心化交易所）聚合器从许多流动性协议中提取流动性，通过拆分交易为交易者提供最优惠的价格。Uniswap 是这些项目最大的单一去中心化流动性来源。
- 智能合约开发人员使用可用的功能套件来发明新的 DeFi 工具和其他各种实验性想法。查看[Unisocks](https://unisocks.exchange/)或[Zora](https://ourzora.com/)等项目，其中包括许多其他项目。

**Uniswap 的团队和社区**

Uniswap 团队与更广泛的 Uniswap 社区一起推动协议和生态系统的发展。

### Swap

Uniswap 中的代币交换是一种将一个 ERC20 代币换成另一个代币的简单方法。

对于最终用户来说，交换是直观的：用户选择一个输入令牌和一个输出令牌。他们指定输入金额，协议计算他们将收到多少输出令牌。然后他们一键执行交换，立即在钱包中收到输出令牌。

在本指南中，我们将在协议级别查看交换期间发生的情况，以便更深入地了解 Uniswap 的工作原理。

Uniswap 中的掉期交易不同于传统平台上的交易。Uniswap 不使用订单簿来代表流动性或确定价格。Uniswap 使用自动做市商机制提供有关利率和滑点的即时反馈。

正如我们在[协议概述](https://docs.uniswap.org/protocol/V2/concepts/protocol-overview/how-uniswap-works)中了解到的，Uniswap 上的每一对实际上都由一个流动资金池支撑。流动资金池是智能合约，持有两种独特代币的余额，并执行有关存取它们的规则。

这个规则就是[常数乘积公式](https://docs.uniswap.org/protocol/V2/concepts/protocol-overview/glossary#constant-product-formula)。当任一代币被提取（购买）时，必须按比例存入（出售）另一个代币，以保持不变。

Uniswap V2 中的所有交换都发生在一个函数中，恰当地命名为`swap`：

```solidity
function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data);
```

#### 接收代币

从函数签名中可能很清楚，Uniswap 要求`swap`调用者*指定他们希望*通过`amount{0,1}Out`参数接收多少个输出令牌，这些参数对应于所需的`token{0,1}`.

#### 发送令牌

尚不清楚的是 Uniswap 如何接收代币作为交换的付款。通常，需要代币来执行某些功能的智能合约要求调用者首先对代币合约进行 **Approve**，然后调用一个函数，该函数依次调用代币合约上的 transferFrom。这不是V2 对接受令牌的方式。相反，Pair 在每次交互**结束**时检查他们的代币余额。然后，**在下一次**交互开始时，当前余额与存储的值不同，以确定当前交互者发送的代币数量。请参阅[白皮书](https://docs.uniswap.org/whitepaper.pdf)了解为什么会这样。

关键点是**在调用 swap 之前必须将令牌转移到 Pair**（该规则的一个例外是[Flash Swaps](https://docs.uniswap.org/protocol/V2/concepts/core-concepts/flash-swaps)）。这意味着要安全地使用该`swap`函数，必须从**另一个智能合约**调用它。另一种选择（将代币转移到该货币对然后调用`swap`）以非原子方式执行是不安全的，因为发送的代币很容易受到套利。

### Pool



### Flash Swap

Uniswap Flash Swap 允许您提取 Uniswap 上任何 ERC20 代币的全部储备金，并免费执行任意逻辑，前提是在交易结束时您：

- 用相应的代币对支付提取的 ERC20 代币
- 退还提取的 ERC20 代币以及少量费用

Flash Swap 非常有用，因为它们避免了涉及 Uniswap 的多步骤交易的前期资本要求和不必要的操作顺序限制。

#### 例子

##### 零资本套利

Flash Swap的一个特别有趣的用例是无资本套利。众所周知，Uniswap 设计的一个组成部分是激励套利者将 Uniswap 价格交易为“公平”的市场价格。虽然从博弈论上讲是合理的，但这种策略只有那些有足够资金来利用套利机会的人才能使用。闪电互换完全消除了这一障碍，有效地使套利民主化。

想象一个场景，在 Uniswap 上购买 1 ETH 的成本是 200 DAI（通过调用`getAmountIn`指定 1 ETH 作为精确输出来计算），而在 Oasis（或任何其他交易场所）上，1 ETH 购买 220 DAI。对于任何拥有 200 DAI 的人来说，这种情况代表了 20 DAI 的无风险利润。不幸的是，你周围可能没有 200 DAI。然而，通过 Flash Swap，只要有能力支付gas 费，任何人都可以获得这种无风险的利润。

##### 从 Uniswap 取出 ETH

第一步是通过 Flash Swap 从 Uniswap 中*乐观地提取 1 ETH。*这将作为我们用来执行套利的资金。请注意，在这种情况下，我们假设：

- 1 ETH 是预先计算的利润最大化交易
- 自我们计算以来，Uniswap 或 Oasis 的价格没有变化

可能是我们希望在执行时计算利润最大化的链上交易，这对价格变动是稳健的。这可能有点复杂，具体取决于正在执行的策略。*然而，一种常见的策略是针对固定的外部价格进行*尽可能有利可图的交易。（这个价格可能是 Oasis 上一个或多个订单的平均执行价格。）如果 Uniswap 市场价格远远高于或低于这个外部价格，以下示例包含计算通过 Uniswap 交易的最大金额的代码利润：[`ExampleSwapToPrice.sol`](https://github.com/Uniswap/uniswap-v2-periphery/blob/master/contracts/examples/ExampleSwapToPrice.sol)。

##### 在外面交易

一旦我们从 Uniswap 获得了 1 ETH 的临时资金，我们现在可以在 Oasis 上用 220 DAI 进行交易。收到 DAI 后，我们需要偿还 Uniswap。我们已经提到，支付 1 ETH 所需的金额是 200 DAI（通过 `getAmountIn` 得到）。因此，在将 200 个 DAI 发送回 Uniswap 对后，您将获得 20 个 DAI 的利润！

##### 即时杠杆（Instant Leverage）

Flash Swap 可用于提高使用借贷协议和 Uniswap 的杠杆效率。

以最简单的形式考虑 Maker：一个接受 ETH 作为抵押品并允许用它铸造 DAI 的系统，同时确保 ETH 的价值永远不会低于 DAI 价值的 150%。

假设我们使用该系统存入 3 ETH 的本金，并铸造最大数量的 DAI。以 1 ETH / 200 DAI 的价格，我们收到 400 DAI。从理论上讲，我们可以通过出售 DAI 以获得更多 ETH、存入该 ETH、铸造最大数量的 DAI（这一次会更少）并重复直到达到我们想要的杠杆水平来提高这一头寸。

使用 Uniswap 作为该过程中 DAI 到 ETH 组件的流动性来源非常简单。但是，以这种方式循环协议并不是特别优雅，而且可能会消耗大量 gas。

幸运的是，Flash Swap 使我们能够预先提取*全部*ETH 金额。如果我们想要对我们的 3 ETH 本金进行 2 倍杠杆，我们可以简单地在 Flash Swap 中请求 3 ETH 并将 6 ETH 存入 Maker。这使我们能够铸造 800 DAI。如果我们铸造了我们需要的数量来覆盖我们的 Flash Swap （比如 605），那么剩余的部分可以作为对抗价格变动的安全边际。

### 公式

**乘积常量 K**：
$$
K = reserve0 * reserve1
$$
**当 swap 发生时，给定 amountIn，计算 amountOut**：
$$
(reserveIn + amountIn) * (reserveOut - amountOut) = K
$$
将此等式与上面的等式结合，最终得到（不考虑任何服务费的情况）：
$$
amountOut = \frac{reserveOut}{\frac{reserveIn}{amountIn} + 1}
$$
Uniswap 协议将收取 0.3% 作为 LPs 的服务费（简化写法，由于是整型，1*1000/997 仍然是 1）：
$$
amountOut = \frac{reserveOut}{\frac{reserveIn}{amountIn} * \frac{1000}{997} + 1}
$$
mintFee，如果费用开关打开，当 pair 发生 mint 或 burn 时，会产生的费用：
$$
mintFee = \frac{totalLiquidity * (K - KLast)}{5 * K + KLast}
$$


## DAO 投票

### 合约信息

| Contract Name          | Address                                                      | Functionality  |
| ---------------------- | ------------------------------------------------------------ | -------------- |
| UNI                    | [0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984](https://goerli.etherscan.io/address/0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984#code) | DAO 治理 Token |
| Timelock               | [0x1a9C8182C09F50C8318d769245beA52c32BE35BC](https://etherscan.io/address/0x1a9C8182C09F50C8318d769245beA52c32BE35BC) | 延迟执行交易   |
| GovernorBravoDelegator | [0x408ED6354d4973f66138C91495F2f2FCbd8724C3](https://etherscan.io/address/0x408ED6354d4973f66138C91495F2f2FCbd8724C3#code) |                |
| GovernorBravoDelegate  | [0x53a328F4086d7C0F1Fa19e594c9b842125263026](https://etherscan.io/address/0x53a328f4086d7c0f1fa19e594c9b842125263026#code) |                |

