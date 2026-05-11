# 【TIFS‘2024】Toward Understanding Asset Flows in Crypto Money Laundering Through the Lenses of Ethereum Heists

## Metadata

- Title: Toward Understanding Asset Flows in Crypto Money Laundering Through the Lenses of Ethereum Heists
- Authors: Jiajing Wu, Dan Lin, Qishuang Fu, Shuo Yang, Ting Chen, Zibin Zheng, Bowen Song
- Venue / Year: IEEE Transactions on Information Forensics and Security, 19, 1994-2009, 2024
- Note Name: 【TIFS‘2024】xblockflow
- Paper: https://doi.org/10.1109/TIFS.2023.3346276
- Local PDF: `/Users/wenkuanxiao/Zotero/storage/NSSGTF8A/Wu 等 - 2024 - Toward Understanding Asset Flows in Crypto Money Laundering Through the Lenses of Ethereum Heists.pdf`
- Zotero / IEEE URL: https://ieeexplore.ieee.org/document/10371347/
- Code: https://github.com/lindan113/EthereumHeist
- Dataset / Artifact: EthereumHeist supplementary repository and linked Dropbox artifact. Public GitHub files include Etherscan heist labels, incident metadata, and service-provider category mapping. The paper reports an Ethereum ML dataset with more than 160,000 addresses and rich phase/layer/service-provider metadata.
- Scope / Subfield: 链上洗钱行为分析 / 资金流取证 / AML 数据集构建
- Tags: AML, Ethereum, Money Laundering, Taint Analysis, Transaction Tracing, Dataset, Graph Analysis, DeFi, Mixer, Cross-chain, Counterfeit Token
- Status: DONE

## TL;DR

这篇论文解决的是 Ethereum 上缺少公开、细粒度 AML 数据和系统化洗钱行为分析的问题。作者提出 XBlockFlow：从 Etherscan 标记的 115 个 `Heist` 源地址出发，把洗钱过程抽象为 placement、layering、integration 三阶段，再用改造过的 taint analysis 追踪下游地址，最终构建 EthereumHeist 数据集并分析 2016-2022 年 73 起 Ethereum 盗币事件。

方法上，论文的核心不是训练检测器，而是做一个从已知盗币源地址到下游洗钱地址和出口服务商的半自动取证框架。最有价值的结果是把 Ethereum 洗钱的实际形态拆清楚了：层数可达 15 层，2021 年事件激增且 DeFi exploit 占主导，洗钱出口从 CEX 逐步转向 DEX、跨链、借贷和混币服务，还观察到 token swap、伪造 token、低价抛售 NFT 等 Bitcoin 场景里不典型的高级洗钱手法。

## 毒舌评论

这篇论文真正有用的是数据和行为画像，不是算法。XBlockFlow 的追踪规则本质上仍是“从已知脏地址向外做截断污染传播”，只是在 Ethereum 场景里加了服务商停止条件、交易数阈值和标签库过滤；它能扩展出很大一批可疑地址，但并不能证明这些地址背后一定是黑客。换句话说，它是一个很好的洗钱研究地图和候选生成器，不是链上 AML 的判官。

## Research Question

- 研究对象：Ethereum 安全事件后，被盗 crypto-assets 从黑客源地址流向下游账户、DeFi 协议、交易所、跨链桥、混币器等服务商的过程。
- 小领域范围：链上 AML、攻击后资金流取证、Ethereum 账户级洗钱数据集构建。
- 具体问题：给定真实世界盗币事件和 placement address，如何系统化收集下游 layering 地址、integration 地址和相关交易，并分析这些地址在三阶段洗钱中的行为特征。
- 为什么重要：Bitcoin 有 Elliptic 这样的公开 AML 数据集，但它只有 illicit / licit 二分类标签，且 UTXO 模型无法覆盖 Ethereum 上的 account reuse、智能合约、ERC20 token、DeFi swap、跨链和混币服务等行为。
- 论文边界：论文从已知 heist 源地址出发，不解决未知攻击的发现；追踪范围止于服务商入口，不继续解决 mixer 出口映射、跨链桥提现端映射和 CEX 内部账本；标签和 ground truth 大量依赖 Etherscan、公开报告和作者维护的 label library。

## Motivation and Basic Idea

- Motivation：传统 AML 研究依赖身份信息和银行内部数据；Bitcoin AML 研究依赖 UTXO 交易图和 Elliptic 数据集；而 Ethereum 场景同时有伪匿名账户、可无限开户、智能合约、多 token、DeFi 兑换、跨链桥和混币服务。现有数据集和方法很难解释盗币后资产到底如何被拆散、换币、聚合和提现。
- Basic idea：把真实盗币事件当作洗钱源头，用 `Heist` 标签地址作为 placement set `P`，沿着 Ethereum 交易图逐层传播污染，过滤掉普通用户行为和已知服务商，得到 layering set `L`、integration set `I` 和 transaction set `T`。之后按三阶段分别做统计、图分析和出口服务商演化分析。
- 这个 idea 如何回应 motivation：Ethereum 洗钱不是单一交易二分类，而是一条从攻击源到账户层、服务商层、协议层的资产流。先从可信源地址开始追踪，能绕开“完全无监督识别洗钱”的困难，并为后续 AML 模型提供细粒度阶段标签和事件上下文。
- 作者给出的证据：作者从 Etherscan `Heist` 标签得到 115 个源地址，聚合为 73 个事件；用 Upbit Hack 的 815 个已标记地址观察洗钱层级和交易特征，进而设计 Truncated Poison Policy；在 Upbit Hack 上评估时，深度 8 仍保持 90% 以上 precision，recall 为 96.2%。
- 我的判断：论文的动机成立，而且比“直接训练一个 Ethereum AML 分类器”更务实。它先补数据和现象理解，再谈检测；这对后续研究更有用。但它也把“公开标签可信且足够完整”作为关键前提，这个前提在真实执法和交易所内部风控中未必成立。

## Background

- 背景：Ethereum 使用 account model，包含 EOA 和 contract account；交易包括 external、internal 和 ERC20 token transactions；VASP / CEX / DEX / mixing service / cross-chain bridge 都可能成为洗钱出口。
- 问题：盗币后，攻击者通常先把资金放入 placement 地址，再通过多跳账户和多 token 转账进行 layering，最后将资金汇入 exchange、DEX、bridge、mixer、lending pool 等 integration 服务商进行提现或进一步混淆。
- Gap：Elliptic 数据集只覆盖 Bitcoin 且标签粗；Ethereum 公开研究缺少按事件、按阶段、按层级、按出口服务商标注的 AML 数据；传统 taint analysis 的 Poison / Haircut 规则面向 UTXO，不适合账户可复用的 Ethereum。

## Threat Model / Assumptions

- 攻击者或参与者能力：攻击者能控制多个 Ethereum 地址，能进行多跳 ETH/ERC20 转账，能通过 DEX 换币、跨链桥转移、混币器混淆、借贷池存取、NFT 市场出售等方式转移价值。
- 链上 / 链下假设：链上交易数据可通过 Etherscan API 获取；Etherscan 的 `Heist` 标签、公开安全报告、项目方公告可用于确认源地址和事件信息；服务商地址可通过 label library 识别。
- 市场、流动性、排序、预言机或网络假设：论文不建模 MEV、交易排序、滑点或价格形成，只把已发生交易作为资金流边；token swap 被当成洗钱出口或高级混淆手法，而不是市场策略本身。
- 不覆盖的情况：进入 mixer、跨链桥或 CEX 之后的内部映射不追踪；off-chain 场外交易会产生不可消除的假阳性；未被 label library 覆盖的新服务商和新 token 项目会影响追踪终止条件。

## Method

- 核心思路：`Incident search -> Downstream tracking -> Phase-wise analysis`。先收集真实 Ethereum heist 源地址，再用 Truncated Poison Policy 逐层追踪下游地址，最后按照 placement、layering、integration 三阶段分析事件、交易网络和出口服务商。
- 核心思路是怎么想到的：作者先用 Upbit Hack 做观察。Etherscan 对该事件标出 815 个相关外部账户，并按从源地址出发的 hop 数表示层级。观察发现，洗钱层数可达 15 层，但单个洗钱地址交易数并不大，全部交易数最大 603、平均约 9.31、中位数仅 3。因此可以用“逐层污染传播 + 大交易数地址当作服务商截断”的方式控制搜索范围。
- 从 motivation 到 method 的逻辑链：Ethereum 没有细粒度 AML 数据集 -> 从公开 `Heist` 标签构造 placement 源头 -> 真实 laundering 可能有多层、多 token、多服务商 -> 用 taint analysis 逐层追踪 -> Bitcoin Poison/Haircut 不适合 account model -> 设计 TPP，保留 Poison 的传播思想，同时用服务商标签和交易数阈值截断 -> 得到可分析的 `P, L, I, T`。
- 关键设计取舍：论文选择从公开标签出发，牺牲覆盖率换取较高可信度；选择规则追踪而非学习模型，牺牲泛化表述换取可解释和可复现；选择在服务商入口停止，避免 mixer / bridge / CEX 内部不可见区域带来的过度推断。
- 为什么不是更直接 / 更简单的方案：直接把 Elliptic 模型迁移到 Ethereum 会丢失事件、层级和服务商语义；直接 BFS 下游会爆炸；纯 Poison 会把所有接收方都污染，容易误伤普通用户和服务商；Haircut 的金额比例思想依赖 UTXO，难以处理 Ethereum 账户余额混合和账户复用。
- 系统流程或算法步骤：
  - Phase I, placement：从 Etherscan `Heist` label word cloud 收集 115 个已知盗币相关地址，按 name tag 聚合为事件。排除没有 name tag 的地址后，得到 73 个 2016-2022 年 Ethereum 事件，类型包括 CEX hack、DeFi exploit、scam 和 others。
  - Phase II, layering：对每个事件，从 placement set `P` 开始逐层查询 external、internal 和 ERC20 交易。每层维护当前可疑集合 `Cur_k`，把满足条件的未标记下游地址加入下一层，把已知服务商加入 integration set `I`。
  - Phase III, integration：根据服务商标签库，把出口地址映射为 CEX、DEX、crossing chain services、loan services、mixing services 和 others，并分析这些出口类型随年份变化。
  - On-chain crawling：作者不用同步全量 block 数据，而使用 Etherscan API 基于地址抓一跳 incoming / outgoing transaction records，因为 AML 追踪需要按地址查询，是 space-intensive query。
  - Off-chain label library：服务商标签来自交易所、DEX、Tornado.Cash 等 label 地址，规模超过 260,000 项；token 标签来自作者此前的 ERC20TokenInfo 和 ERC721TokenInfo 数据集，分别超过 313,000 个 ERC20 token 与 15,000 个 ERC721 token。
- 关键定义 / 公式 / 不变量：
  - 洗钱过程抽象为四元组 `(P, L, I, T)`，其中 `P` 是 placement 地址集，`L` 是 layering 地址集，`I` 是 integration 地址集，`T` 是参与洗钱过程的交易集。
  - `k`-th laundering layer 表示从原始 placement 账户出发经过 `k` 跳到达的可疑地址集合；最大层数即最大 tracking level / depth。
  - 交易网络为 `G=(V,E)`，`V` 是 Ethereum 地址，`E` 是有向交易边 `(u,v)`。
  - Truncated Poison Policy:

```text
Cur_{k+1} = { v | (u, v) in E, u in Cur_k } - { v | v in SP or User(v) }
```

  - 其中 `SP` 是已知或疑似服务商地址集，`User(v)` 表示地址行为像普通用户。
  - Algorithm 1 输入 placement set `P`、label library `Lib`、最大追踪深度 `K`、每层最大地址数 `Psi`、未知服务商交易数阈值 `Omega`；论文实验设置 `K=20`、`Psi=10,000`、`Omega=1000`。
  - Dirty amount 过滤阈值 `beta` 用于丢弃小额污染交易，Upbit Hack 中 `beta=0.01`。
  - Algorithm 2 用于统计 DeFi token swap：过滤零金额、自环、zero address mint/burn 后，在同一 transaction hash 下寻找不同 token、方向相反、from/to 互换的两笔转账。
- 实现细节：论文和补充材料给出 EthereumHeist 仓库，包含 heist label、事件信息、服务商映射和外部数据链接。正文中的 XBlockFlow 更像数据构建和分析流程，不是完整开源的生产级 AML 系统。

## Evaluation

- 实验思路：实验不是训练模型，而是验证追踪规则和分析数据。作者先用 Upbit Hack 的部分标注地址评估 layering 识别，再对 73 个事件构造 EthereumHeist 数据集，最后从 placement、layering、integration 三阶段做统计、图结构和高级洗钱手法分析。
- 评估指标：Upbit Hack 上用 precision-depth 曲线和 recall；后续分析主要用事件数、年份分布、地址层数、交易数、金额、网络密度、motif 分布、服务商类别比例和案例研究。
- 主要结果：
  - Placement：Etherscan `Heist` 标签中 115 个源地址聚合为 73 个事件。年份分布为 2016: 1、2017: 1、2018: 4、2019: 5、2020: 6、2021: 49、2022: 7。2021 年激增，且论文称 2021 年 49 起事件中 67% 是 DeFi exploit。
  - Upbit case study：Upbit Hack 有 815 个 Etherscan 标记的相关外部账户，层数最高到 15。各层地址数量呈中间多、两端少的形态，最大一层有 190 个地址。815 个地址的总交易数最大 603、中位数 3、平均 9.31。
  - Layering identification：在 Upbit Hack 上，追踪深度越大，预测出的 layering 地址数指数增长；深度 8 时 precision 仍超过 90%，recall 为 96.2%。
  - Layering overview：不同事件持续时间差异很大，从不足 1 天到接近 3 年；早期事件通常持续更久。2022 年 LCX hack 中，攻击者约一天内通过 DEX 把 ERC20 token 换成 ETH，并转入 Tornado.Cash。
  - Graph properties：Layering 地址具有更短 lifespan 和明显的 used-and-dumped 特征；交易金额显著大于普通账户，平均 inflow/outflow 超过 50 ETH，大约是普通账户的 3-5 倍；HeistEthNet 与 HeistTokenNet 比普通交易网络更稠密。
  - Motifs：ML 网络中闭合三角 motif `M3-M9` 比例较低，开放三角 `M10-M12` 更常见，分别对应扩散、分层转移和聚合提现；DeFi exploit 事件中 `M13-M15` 较多，反映 token swap 造成的双向边。
  - Integration：Top service providers 包括 Uniswap V2/V3 Router、Tether USD、USDC、Wrapped Ether、Binance、OpenSea、EtherDelta 等，说明黑钱常进入 DEX、稳定币、wrapped token、CEX 和 NFT 平台。
  - Exit evolution：2018 年 CEX 是主要出口，之后比例下降；DEX、跨链服务、借贷服务和 Tornado.Cash 等混币服务占比上升。作者将其解释为 CEX AML/KYC 趋严后，攻击者转向更去中心化或更难追踪的出口。
  - Advanced obfuscation：论文总结三类高级手法：DEX token swap、counterfeit token deployment、cheap sale of stolen NFTs。典型例子包括 AscendEX exploiter 将 USDT 换为 DAI/USDC，Bitmart hacker 将 MANA 换成 ETH 后进入 Tornado.Cash，Nexus Mutual hacker 将 ETH 换成 renBTC 后跨到 Bitcoin，Akropolis hacker 通过伪造 RZN token 和 Uniswap 流动性池伪装成普通投机者。

## Key Artifacts

- 关键图：
  - Fig. 1：Elliptic Bitcoin AML 数据集示例，说明现有公开 AML 数据主要是 Bitcoin 交易图，标签粒度粗。
  - Fig. 2：Kucoin Hack 后的 Ethereum 洗钱流程示例，展示 placement、layering、integration 三阶段以及 ETH/token 多资产转移。
  - Fig. 3：XBlockFlow 框架图，分为 identification 和 analysis 两部分。
  - Fig. 4：Crypto money laundering 抽象模型，用灰色椭圆表示 laundering layers。
  - Fig. 5：Etherscan `Kucoin Hacker` 标签截图，说明 source address 的来源。
  - Fig. 6：Bitcoin Poison 与 Haircut taint policies，用来引出为什么 Ethereum 需要 TPP。
  - Fig. 7：Upbit Hack 上不同 depth 的预测地址数与 precision，用来支撑追踪算法的可用性。
  - Fig. 8：Heist layering 地址与普通交易图节点在 lifespan、金额等交易特征上的差异。
  - Fig. 9：ML 网络 motif 分布，支撑开放三角和 DeFi swap motif 的观察。
  - Fig. 10：2018-2022 年出口服务商类别演化，支撑 CEX 下降、DEX/跨链/混币上升的结论。
  - Fig. 11：Akropolis fake token case，展示伪造 token 如何在洗钱中充当伪装层。
- 关键表：
  - Table I：Upbit Hack 每层地址数量，最高 15 层，最大一层 190 个地址。
  - Table II：Upbit Hack 815 个地址交易特征，总交易数最大 603、中位数 3、平均 9.31。
  - Table III：73 个 Ethereum incidents 的 case name、type、year、amount stolen、placement set size。
  - Table IV：2016-2022 年事件数量分布，2021 年 49 起，占绝对多数。
  - Table V：HeistEthNet / HeistTokenNet 与普通 TransactionNet / TokenNet 的网络属性对比，说明洗钱网络更稠密、更局部聚集。
  - Table VI：交易量排名前 10 的 integration service providers。
  - Table VII：服务商到类别的映射样例，如 Binance -> CEX、Uniswap -> DEX、Polygon -> cross-chain services、Tornado.Cash -> mixing services。
- 关键公式 / 定义 / 算法：
  - 四元组 `(P, L, I, T)` 是全文的洗钱过程抽象。
  - TPP 公式是下游候选地址生成的核心。
  - Algorithm 1 是 AML Tracking Algorithm，负责从 `P` 推出 `L`、`I`、`T`。
  - Algorithm 2 是 DeFi Token Swap Counting，用 transaction hash、asset type 和反向转账关系识别 swap。
- 这些证据分别支撑哪些结论：Fig. 2-4 支撑问题抽象；Table I-II 和 Fig. 7 支撑 TPP 参数和追踪设计；Table III-IV 支撑数据集规模和事件分布；Fig. 8-9 支撑 layering 行为画像；Table VI-VII 和 Fig. 10-11 支撑 integration 演化和高级混淆手法。

## Findings

- 发现 1：Ethereum 洗钱层数可以远超传统金融 AML 常见实验设定。Upbit Hack 的层数达到 15，而传统 AML 实验常用 3-5 层。
- 发现 2：洗钱地址往往交易次数不多，但金额大、生命周期短，呈 used-and-dumped 特征。这说明单靠“交易频繁”并不能抓到链上洗钱地址。
- 发现 3：2021 年是样本中 Ethereum 盗币事件的爆发点，DeFi exploit 是主因。DeFi 的资产聚集、协议复杂度和 AML 缺位共同扩大了洗钱空间。
- 发现 4：出口服务商明显演化。CEX 仍重要，但 DEX、跨链桥、借贷池、Tornado.Cash 等服务被越来越多用于分散、换币、跨链和混淆。
- 发现 5：Ethereum AML 和 Bitcoin AML 的差异不只是账户模型。Token swap、wrapped asset、fake token liquidity pool、NFT cheap sale 让洗钱行为更像“协议级资产变形”，而不是简单转账路径。

## Strengths

- 论文最有说服力的地方：它把 Ethereum 洗钱拆成了可操作的数据构建流程和三阶段分析框架，补上了 Elliptic 之外缺少 Ethereum AML 细粒度公开数据的空缺。
- 方法、数据、实验或问题设定的优势：从真实 heist 标签出发，事件语境清楚；数据包含事件年份、类型、被盗金额、层级、交易和出口服务商类别，远比二分类标签有研究价值；对 token swap、fake token、NFT sale 的观察贴合 Ethereum/DeFi 生态。
- 相比已有工作的有效推进：相比只做 illicit account detection 或 fraud transaction classification，这篇论文关注攻击后资金流全链条；相比 TRacer 这类通用交易追踪工具，它更强调 AML 数据集构建和洗钱行为统计。

## Limitations

- 威胁模型、假设或适用范围的限制：XBlockFlow 依赖已知 `Heist` 源地址，不能发现没有公开标签的新事件；追踪到服务商入口即停止，不能还原 mixer、bridge、CEX 之后的完整资金链。
- 数据集、baseline、metric、ablation 或复现性的不足：Upbit Hack ground truth 来自 Etherscan 部分标记，不是完整独立审计标签；precision / recall 评估只在一个代表性事件上展开；完整数据依赖补充材料和外部链接，正文没有给出可直接复跑的完整 pipeline 细节。
- 在真实 DeFi 场景中可能失效的条件：off-chain 交易会把无辜用户地址卷入污染链；label library 更新不及时会把新服务商误判为 layering 地址；复杂聚合器、跨链桥、多交易拆分、CEX 内部归集、隐私协议会削弱链上 taint 传播的解释力。
- 过度解读风险：TPP 输出的是 suspicious ML accounts，不是法律意义上的犯罪账户。论文对伦理问题有说明，但下游使用者如果把“收到脏钱”直接等同于“参与洗钱”，会造成严重误伤。

## My Takeaways

- 对 DeFi / 区块链安全研究的启发：Ethereum AML 不能只做交易分类，需要事件级、阶段级、协议级语义。尤其是 integration 阶段，服务商类型变化本身就是攻击者策略演化的信号。
- 可复用的方法：`P/L/I/T` 四元组和 TPP 追踪流程可以作为构建其他链 AML 数据集的起点；Algorithm 2 的 swap counting 思路可以作为识别 DeFi 洗钱动作的轻量规则。
- 可能的后续问题：把 XBlockFlow 的事件级标签和 TRacer 的排名式局部追踪结合，可能得到更强的“候选生成 + 审计优先级”系统；再结合 DeFiRanger/DeFiScope 这类交易语义恢复工具，可以更准确解释 token swap、LP、借贷和跨链动作。

## Related Papers

- 前置阅读：Elliptic AML dataset；Moser et al. 的 Bitcoin taint analysis；FlowScope；Ethereum graph analysis；XBlock-ETH。
- 后续阅读：TRacer、HOUSTON、BERT4ETH、Following Devils' Footprint、SoK: DeFi Attacks。
- 可对比论文：TRacer 关注 account-based chains 上的通用交易追踪；BERT4ETH 和 fraud detection 方向关注账户/交易分类；DeFiRanger 和 DeFiScope 更关注 DeFi 攻击语义和价格操纵检测。
- 最接近的相关工作：Fu et al. 的 Upbit Hack money laundering 分析和 Liu et al. 的 GTN2vec Ethereum money laundering detection。本文把单事件或检测模型扩展为多事件 EthereumHeist 数据集和三阶段洗钱画像。
- 关键差异：本文的贡献重点是数据集和现象分析，而不是提出一个最终 AML 分类模型。它从 heist 事件出发理解资产流，补的是“研究对象和证据链”这层基础设施。

## Open Questions

- EthereumHeist 的标签能否持续维护？如果 Etherscan 标签、服务商地址、token 项目和 Tornado.Cash 替代品不断变化，静态数据集很快会老化。
- TPP 参数 `K=20`、`Psi=10,000`、`Omega=1000` 是否对 CEX hack、DeFi exploit、scam、NFT theft 都稳定？不同事件类型是否应该自适应调参？
- 如何给每个 suspicious layering address 生成可审计解释，而不只是“它在第 k 层收到了脏钱”？
- Off-chain OTC、CEX 内部转账、跨链桥 deposit-withdraw 映射和 mixer 出入池映射如何系统接上？
- 能否把 integration 服务商的 KYC/合规反馈引入闭环，验证哪些链上 suspicious addresses 真正与洗钱实体相关？
