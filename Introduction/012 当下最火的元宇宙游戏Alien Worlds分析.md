# 当下最火的元宇宙游戏Alien Worlds分析

## 简介

Alien Worlds 是一个集挖矿、NFT、DeFi、DAO为一体的元宇宙游戏。其中的元宇宙通用货币 Trillium（简写为TLM）用于激励用户探索 Alien Worlds 世界。用户可以通过在行星的一块土地上挖矿获得TLM奖励；成为土地所有者收取佣金；质押 TLM 参与 DAO 治理，影响行星的 TLM 分配；租用航天器做任务获得额外TLM收益与NFT等。随时间推移，会推出更多玩法，比如行星提供了自己的游戏和NFT [1]。

在下文，我会讲到自动挖矿的方案，收益的简单计算等内容。

**我将尽可能的客观，引导你独自思考。请自行判断 Alien Worlds 的潜力、是否游玩等。因为，只要你读懂了，那么心里一定会有答案，都没读懂，建议不要轻易入场。**

## 游戏玩法一览

- 游戏本身玩法

  - Mining，挖矿

  - Shining，升级 NFT 游戏卡

  - Fighting（还未实现），战斗

  - Mission，任务

  - DAO，去中心化自治组织

- 二级市场玩法

  - 游戏卡（NFT） 低买高卖

  - 买土地，收佣金

  - 为 TLM/X 流动池提供流动性

- 官方活动

  - 答题，抢 NFT

  - 推广，有几率中 NFT

  - 参与 Twitch 游戏，有几率中 NFT

## Mining

### 快速开始

**你可以扮演一个矿工的角色**，选择一颗行星、对应的土地并装备好工具后，开始挖矿。每个土地的佣金率不同，如果是还未出售的土地，将有默认的20%的佣金率，而已出售的土地，则根据土地所有者自行决定佣金率。佣金率意味着你作为一个普通的矿工，在挖矿获得 TLM 后，需要按佣金率支付给土地所有者一定的佣金。

佣金率意味着，**你也可以扮演一个土地所有者的角色**，购买土地，吸引矿工到你的土地上挖矿，并收取佣金。这取决于你的策略。

![image-20220109160220288](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109160220288.png)

参与挖矿只需要每次冷却时间到，点击一下 Mine 按钮即可。

![image-20220109160202687](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109160202687.png)

### 影响收益的公式

挖矿能力百分比 = 所有已装备的工具的挖矿能力之和 * 土地的挖矿能力倍数

**单次挖矿收益由挖矿能力百分比与Current Mining Pot 共同决定**

挖矿冷却时间 = 已装备的工具冷却时间 * 土地冷却时间倍数

已装备的工具冷却时间 =

​	较长的冷却时间 + 1/2 的较短的冷却时间（如果只有两个工具）

​	**或者**

​	两个最长的冷却时间相加（如果有三个工具）



你可以借助[此工具](https://alienworlds.tools/)，帮你快速完成这个步骤。

![image-20220109153458599](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109153458599.png)

根据你自己对于冷却时间、每分钟的TLM产出等需求做出适合自己的决策即可。

![image-20220109155520651](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109155520651.png)

更好的工具 NFT 效率会更高，下面是不同工具 NFT 的总览，你可以在[这里](https://docs.google.com/spreadsheets/d/1BjpZQfG5JWnL5Vv_YMAi3LYytHJSBwhnLDp0Sc2zHCg/edit#gid=0)看到该数据：

![image-20220109173924647](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109173924647.png)

以类型为提取器（Extractor）为例，更稀有的工具，充能时间增加了，意味着普通用户不需要每天挖矿那么多次，更加的高效。TLM Mining Power 增加了，意味着用户每次挖矿的收益增加了。

我们对比下装备三个 Standard Shovel 与 三个 Standard Drill 的区别：

装备三个 Standard Shovel：

![装备三个 Standard Shovel](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109174400600.png)

装备三个 Standard Drill：

![装备三个 Standard Drill](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109174846412.png)

装备三个 Standard Drill 的效率是 0.75%/分钟，而装备三个 Standard Shovel 的效率是 0.56%/分钟，显然有一定提升。

### 原理

其实这里的挖矿，背后实现逻辑真的就是 PoW，你需要找到一个符合条件的 Hash 才能够获得 TLM 奖励。

官方在下面给出了该条件 [1]：

![image-20220109160509778](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109160509778.png)

该决策的制定，主要是防止大量机器人挖矿。举个例子，单个普通用户挖矿的速度是 1 秒，那么相同的电脑配置，10个机器人的速度就是10秒，以此类推。不过由于计算难度并不高，并且工具有充能时间（每次挖矿至少要间隔的时间），其实防范也很有限。所以 Alien Worlds 也会酌情根据内部的其他策略来判定是否为机器人，如果认定你是机器人，你会被拉入黑名单，挖矿收益会近乎为 0。

以我目前的使用体验来看，手机、电脑均能在秒级完成该 PoW 的计算。官方给出的计算公式中 PoW Reduction 显然有些多余，因为挖矿速度并没有明显慢到需要用更好的工具 NFT 属性中的 PoW Reduction 来减少 PoW 的计算量。

### 自动挖矿

手动挖矿太费功夫了，因为你即便辛苦挖一天，收益也不高。那么如何提高效率？答案是自动化。

有很多自动化的办法，我这里提两种方案：

- 按部就班的基于图像识别，点击对应的按钮进行挖矿。
- 根据上面的 PoW 计算公式，计算出 Hash，并调用智能合约进行挖矿。

也正是因为要做自动化比较容易，所以 Alien Worlds 上很多数据可能是机器人产生的，这一点毋庸置疑。而且，原本 Alien Worlds 计划挖矿能够根据 NFT luck 属性，概率性的挖矿得到 NFT，不过现在不可能了。应该是跟挖矿自动化有关，普通人通过手动挖矿得到 NFT 的概率过低（因为大部分是机器人，可以靠量取胜）。现在获取 NFT 的策略是参与官方的社区活动（在本文写作时是这个策略，不保证后续不做更改）。

大量的机器人去自动挖矿的成本，显然也比获得更高级的工具 NFT 的成本更低，收入比工具 NFT 更高。

## Shining

闪耀（Shininess）等级分为：

- 石头（Stone）
- 黄金（Gold）
- 星尘（Stardust）
- 反物质（Antimatter）

收集4张相同NFT卡片，你可以花费 100 TLM 去“闪耀”你的 NFT 卡，达到更高的闪耀等级。你可以根据“闪耀”与直接在市场上购买NFT的成本，哪个更低，而选择是“闪耀”还是直接购买。**甚至以此赚取差价**。

## Mission

### 如何玩

官方对此的介绍是：租用航天器，在元宇宙中执行任务。探索任务，发现 NFTs。

其实本质上就是：锁仓赚取 TLM + NFT。

任务的稀有度分为：

- 普通（Common）
- 稀有（Rare）
- 史诗（Epic）
- 传说（Legendary）

以已经开始的 Mission（使命、任务） 为例：

![image-20220109162806237](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109162806237.png)

这个任务将持续 1 周的时间，对本次任务将总共奖励 20000（由每艘航天器平分），并在此次任务中每艘航天器获得 1 个对应任务稀有度的 NFT。也就是说，如果任务稀有度为普通，你将获得普通 NFT 卡。每个任务最多获得 5 个 NFT。

### 收益计算

每艘航天器的奖励如何计算出的？很简单，将 总奖励 / 租用的航天器数 即可得到，例如上图数据，20000 / 24108 = 0.8296，跟官网数据一致。

![image-20220109163718140](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109163718140.png)

我将这个任务质押金额及收益列了一个简单表格，其中 APR 是年利率：

![image-20220109163402920](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109163402920.png)

任务获得的 NFT 产生的价值，随市场波动，需自行评估。

## DAO

DAO 在 Alien Worlds 中的作用很简单：用户质押 TLM 到行星上，获取行星的治理代币（每个行星的治理代币均不同），治理代币可用于投票或让自己成为候选人。行星上质押的 TLM 数量将影响每颗行星上的 TLM 分配，质押数越多，分配到的 TLM 也越多。

不过很可惜的是，目前的 Alien Worlds 的 DAO 生态还未建立起来，我们可以从它们的数据上看到，即便是目前 TLM 质押最多的 Neri 行星上的候选人也都是 0 票。

![image-20220109164622034](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109164622034.png)

当然，这可能也与没有投票的入口有关。这里只是简单的展示了候选人宣言及其选票。

![image-20220109164708911](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109164708911.png)

你可以在游戏的 [Governance 入口](https://play.alienworlds.io/governance)，质押 TLM。从[这里](https://council.alienworlds.io/)进入 DAO 的投票页面。

![image-20220109165008742](https://img-thestarboys.oss-cn-beijing.aliyuncs.com/img/image-20220109165008742.png)

## 如何变现

1. 走 WAX 链上的 alcor 交易所，将 TLM 转为 WAXP，将 WAXP 充值到一些支持 WAXP 的交易所，比如 Binance 上 [2]。
2. 走 Alien Worlds 的 Teleport，将 WAX 链上的 TLM 向 BSC 链，通过 TLM（BEP20）直接充值到 Binance 账户

## Token

### TLM 应用场景

- 用户通过质押 TLM 参与在行星上投票
- 联邦将对例如战斗游戏的玩法征收 TLM 作为费用（暂未实现）
- 用户必须质押 TLM 成为候选人，参与行星的 DAO 治理（未完整实现）
- 参与探险需要 TLM
- TLM 能够用于让 NFT 更闪耀

因此，就目前实现的情况来看，TLM 应用场景还很少。

### 架构的优势

#### 在 WAX 链上游戏的优势

- **新用户门槛低**，WAX 链在[云钱包](https://wallet.wax.io/)中内置了WAXP购买、质押，DApp导航等，操作更方便。而且支持社交账号直接登录，例如谷歌账号。

- 与其他链不同，**WAX 链没有 gas 费**。WAX 链上的交易会消耗 CPU、NET、RAM，其中 CPU 与 NET 都是 24 小时后可再生的，质押 WAXP 就可以得到对应份额的 CPU、NET，你可以随时取出 WAXP。RAM 则需要购买，而不是质押，但基本上对 RAM 的需求很小，玩 Alien Worlds 质押 2 WAXP 足以。目前，对于普通用户而言，为日常游戏所需，质押/购买 CPU、NET、RAM 的比例为 50:2:2。
- Alien Worlds 给用户的前几次交易提供了**免费额度**，你无需花任何钱，就可以开始玩 Alien Worlds。

#### 在 Binance 上的 Mission 游戏

**BSC 链背靠 Binance，上面用于巨大的消费潜力**。在 Binance 上做 Mission 游戏，对于给 Alien Worlds 引流有一定的帮助，并且用户也更容易直接在 Binance 上进行 NFT 的买卖。

#### 跨链

打通了 WAX 链与 BSC 链的 TLM 的流通，对于从 Binance 中引流，或给了用户能够在 BSC 链上添加 TLM 流动性等行为的可能。

## 总结

目前 Alien Worlds 的主要游戏玩法只有两个：挖矿与任务。就目前实现的情况来看，TLM 应用场景还很少。挖矿的设计明显是有缺陷的，无法真正阻止机器人自动挖矿。任务是通过简单的质押获得 TLM 与 NFT 收益。DAO 治理还未达到真正期望的效果。社区运营与推广、架构上的优势做的不错，值得学习。

## 参考

[1] [Alien Worlds 白皮书](https://docs.google.com/document/d/1JiA97Y3JZMcC6HG2VPXEiZDd7UtA5yJSRUY2DQ5VSRI/edit#heading=h.d56vo87xf25r)

[2] [Buying & selling trading alien worlds nfts](https://goodvibesmining.com/aliens-worlds-guide/buying-selling-trading-alien-worlds-nfts/)

