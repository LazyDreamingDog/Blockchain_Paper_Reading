# 【TIFS‘2023】TRacer: Scalable Graph-Based Transaction Tracing for Account-Based Blockchain Trading Systems

## Metadata

- Title: TRacer: Scalable Graph-Based Transaction Tracing for Account-Based Blockchain Trading Systems
- Authors: Zhiying Wu, Jieli Liu, Jiajing Wu, Zibin Zheng, Ting Chen
- Venue / Year: IEEE Transactions on Information Forensics and Security, 18, 2609-2621, 2023
- Note Name: 【TIFS‘2023】tracer
- Paper: https://doi.org/10.1109/TIFS.2023.3266162
- Local PDF: `/Users/wenkuanxiao/Zotero/storage/SP2CM7XE/Wu 等 - 2023 - TRacer Scalable Graph-Based Transaction Tracing for Account-Based Blockchain Trading Systems.pdf`
- Zotero / IEEE URL: https://ieeexplore.ieee.org/document/10098630/
- Code: https://github.com/wuzhy1ng/BlockchainSpider
- Dataset / Artifact: 论文给出代码和数据仓库；实验贡献 20 个真实交易追踪案例，覆盖 Ethereum、BNB Chain 和 Polygon，包含 20 个 source nodes、约 0.87K target nodes、23.5M blocks、4.83B money transfers。
- Scope / Subfield: 攻击溯源 / 链上资金流追踪
- Tags: DeFi, Transaction Tracing, Attack Attribution, Graph Search, Personalized PageRank, Local Community Discovery, Taint Analysis, Ethereum, BNB Chain, Polygon
- Status: DONE

## TL;DR

TRacer 解决的是事后资金追踪：给定一个风险源账户，如何从海量 account-based blockchain 交易里找出尽可能多的资金去向，同时把输出子图压到人工审计可处理的规模。它把交易记录建成 directed、weighted、temporal、multi-relationship graph，并把 DeFi 交易粗分为 Xfer 和 Swap，再用 Transaction Tracing Rank (TTR) 做排名引导的局部图扩展，最后用 local community detection 提取核心资金流社区。

实验上，TRacer 在 20 个真实案例中达到 92.31% recall，平均输出 0.87K nodes，平均深度 5.05，平均运行 0.65 小时；相比 BFS、Poison、Haircut、APPR，它的 recall 明显更高，同时不像 LPA/Louvain 那样产出巨大子图。它的价值不是“检测攻击”，而是在攻击已经发生、至少知道一个源账户时，把资金流追踪变成可扩展、可排序、可审计的图搜索问题。

## 毒舌评论

TRacer 最有含金量的部分不是“又做了一个 PageRank 变体”，而是把链上资金追踪里最脏的几个约束放进搜索偏置：方向、金额、时间和 Swap 后的 token redirection。缺点也很直白：它的 benchmark 依赖安全公司和专家标注，论文里很难完全复核 target ground truth；方法也仍然默认资金流能被交易图语义持续跟踪，一旦进入 Tornado Cash 这类隐私协议，图搜索会立刻撞墙。所以这篇论文更像一个实用的追踪候选生成器，而不是一套端到端的链上“破案机器”。

## Research Question

- 研究对象：account-based blockchain 中由黑客攻击、Rug pull、诈骗等事件引发的非法资金流。
- 小领域范围：链上交易追踪、攻击溯源、事后资金流取证。
- 具体问题：给定一个已知风险源账户 `s`，如何搜索出连接 `s` 与目标账户的资金转移子图 `G_s`，并让 `G_s` 包含尽可能多的 target nodes，同时规模尽可能小，便于后续人工审计。
- 为什么重要：区块链账户是伪匿名的，攻击者可通过交易所、DeFi、混币协议和多跳转账清洗资金。若能定位交易所充值地址，理论上可借助交易所 KYC 信息推进线下身份识别和资产追回。
- 论文边界：论文主要面向 Ethereum、BNB Chain、Polygon 这类 account-based chains；默认至少已知一个 source account；方法输出的是待审计资金流子图，不直接完成真实身份识别；隐私增强系统和混币后的下游流向不是它能稳定解决的部分。

## Motivation and Basic Idea

- Motivation：已有 Bitcoin tracing 多依赖 BFS、taint analysis、co-spending/change-address 等启发式规则；这些方法迁移到 account-based chains 时会遇到三类问题：账户伪匿名导致身份特征稀缺，智能合约和 DeFi 让交易意图不再等价于简单转账，链上交易量太大导致无偏搜索和全图计算代价过高。
- Basic idea：把资金追踪改写成局部子图搜索。搜索不是盲目 BFS，而是用个性化排名估计“某账户与风险源资金流的相关性”，每轮优先扩展残差最高的节点；排名时纳入方向、金额、时间和 DeFi token 转换语义。
- 这个 idea 如何回应 motivation：局部搜索避免全图计算，个性化排名避免被全局重要节点误导，DeFi pattern 建模让搜索能跨 token swap 继续追踪资金，local community detection 则把结果压缩成审计友好的核心子图。
- 作者给出的证据：Bitfinex 案例中，BFS 在访问同等数量账户时没到达目标，而 Poison 这类 biased search 能到达；PageRank 的全局排名容易偏向全网重要账户，而 personalized PageRank 更聚焦源账户邻域；消融实验证明去掉 DeFi pattern 后 recall 从 92.3% 降到 80.3%。
- 我的判断：论文的核心抽象是“ranking-guided local search + DeFi-aware redirection”。这比单纯把 taint analysis 改成图算法更强，因为它把审计优先级作为算法目标，而不是只问某条路径是否存在。

## Background

- 背景：account-based blockchain 有 EOA 和 smart contract account，交易可触发 external transaction 和一串 internal transaction。DeFi DApps 通过 ERC20 等 token 标准实现交易、借贷、流动性供给等操作。
- 问题：事前风险预警只能标记风险交易，无法阻止攻击者在得手后洗钱和提现；事后 tracing 需要从大量交易里恢复资金去向，但 DeFi 操作会把 token 转换、LP token 铸造/销毁和多合约调用混在同一个交易上下文中。
- Gap：传统 BFS 输出过大，taint analysis 依赖启发式且多面向 Bitcoin，账户聚类方法找的是“相似账户”而不是“资金路径上的相关账户”，全局 PageRank 又容易被与具体案件无关的高中心性节点干扰。

## Threat Model / Assumptions (Optional)

- 攻击者或参与者能力：攻击者可以控制源账户并进行多跳转账、跨 DeFi 协议换币、向交易所或混币服务转入资金。论文按最坏情况假设只有一个相关账户已知。
- 链上 / 链下假设：链上交易数据可通过公开 API 获取；交易所通常执行 KYC，因此 exchange deposit account 可作为重要 target；target labels 来自 Certik、PeckShield、Chainalysis 等安全公司或专家报告。
- 市场、流动性、排序、预言机或网络假设：论文不建模交易排序或市场价格形成过程，只把已发生交易抽象为资金流边；DeFi 语义主要按 Xfer 和 Swap 捕捉。
- 不覆盖的情况：进入 Tornado Cash 等隐私增强协议后的下游追踪很弱；跨链桥、复杂合约内部账本、非标准 token、批量聚合器、多层嵌套 DeFi 操作可能超出 Xfer/Swap 的表达能力；方法不能替代人工取证和线下身份确认。

## Method

- 核心思路：`Graph construction -> Graph expansion -> Local community detection`。先把交易记录建成多关系资金流图，再从风险源出发用 TTR 排名引导局部扩展，最后抽取源账户的局部社区作为核心审计结果。
- 核心思路是怎么想到的：论文先排除两种直觉但不合适的方案：账户聚类会把角色相似性误当成资金相关性，盲目 BFS/DFS 又会爆炸。真正需要的是围绕特定 source 的 biased local search，并且搜索偏置要理解链上交易的金额、方向、时间和 token 转换。
- 从 motivation 到 method 的逻辑链：account-based chains 上交易意图复杂且数据巨大 -> 需要局部搜索而非全图计算 -> 局部搜索需要相关性排序 -> 普通 personalized PageRank 只看拓扑不够 -> TTR 把交易语义纳入 local push -> 输出子图仍可能偏大 -> 用 conductance 做 local community detection 压缩审计范围。
- 关键设计取舍：TRacer 没有尝试完整理解每个智能合约的业务逻辑，而是抓住 tracing 最常见的资金流语义：Xfer 和 Swap。这让方法具有跨平台通用性，但也牺牲了对复杂协议语义的精确理解。
- 为什么不是更直接 / 更简单的方案：BFS 快但输出巨大且方向不受控；Poison/Haircut 这类 taint 方法可偏置搜索但难处理 account-based DeFi 语义；全局 PageRank 要处理整张交易图且会偏向全局重要节点；账户聚类会错过 hacker、DEX 中间账户、交易所地址这些“角色不同但资金相关”的节点。
- 系统流程或算法步骤：
  - Graph construction：建图 `G=(V,E)`，节点是 accounts，边 `e=(u,v,w,t,b,h)` 表示账户 `u` 在时间 `t` 通过交易哈希 `h` 向 `v` 转移 `w` 单位 token `b`。映射函数 `f_src`、`f_tgt`、`f_amt`、`f_ts`、`f_sym` 分别取边的源、目标、金额、时间和 token 类型。
  - DeFi action modeling：Xfer 包含 transfer、minting、burning；Swap 包含 add liquidity、remove liquidity、trade。若同一 transaction hash 下同时涉及发送和接收 token，TRacer 将其作为 Swap pattern 处理，以便资金流从一种 token 重定向到另一种 token。
  - Graph expansion：每轮执行 Expand、Push、Rank、Pop。Expand 收集当前节点相关边，Push 合并进子图，Rank 用 TTR 更新节点与 source 的相关性，Pop 选择残差最高节点继续扩展。
  - TTR local push：用 `p_s` 表示 rank，用 `r_s(u,t,b)` 表示节点 `u` 在时间 `t`、token `b` 上的 residual。初始化后，每次把一部分 residual 转成 rank，其余 residual 按交易语义传播到邻居。
  - Tracing tendency：用 `β` 控制向出边或入边传播的注意力。追踪资金去向时通常令 `β > 0.5`，让 out-degree neighbors 获得更高 residual。
  - Weighted pollution：金额越大的资金关系越强，residual 按边金额占比分配。
  - Temporal reasoning：追踪去向时沿着更晚的出边走，回溯来源时沿着更早的入边走；如果没有可传播边，residual 留在当前节点。
  - Token redirection：定义递归函数 `ρ`，把 Swap 前后的 token flow 接起来。例如某账户收到 USDC 后在同一交易语境中换成 ETH，后续 residual 应沿 ETH 出边传播，而不是机械停留在 USDC 边上。
  - Termination：当所有节点的 residual 和都低于阈值 `ε` 时停止，即 `max_u Σ_t Σ_b r_s(u,t,b) < ε`。
  - Local community detection：从 `S={s}` 开始，每次加入剩余节点中 TTR score 最高的节点，直到 conductance `Φ(S)=p_s(∂S)/p_s(S)` 低于阈值 `φ`，输出 `G_s.subgraph(S)`。
- 关键定义 / 公式 / 不变量：
  - 图边：`e=(u,v,w,t,b,h)`。
  - 理论成本：图扩展访问次数与输出非零 TTR 节点数均为 `O(1/(εα))`，因此论文称其与全图规模无关。
  - 最大可达深度：若 source 的 `n`-hop 邻居能被图扩展发现，则 `n <= log(ε)/log(1-α) + 1`；实验参数下作者称最多可到 42-hop。
  - Conductance：`Φ(S)=p_s(∂S)/p_s(S)`，用于判定局部社区是否足够紧密。
- 实现细节：论文脚注给出代码和数据仓库 `wuzhy1ng/BlockchainSpider`；实验通过 open APIs 获取交易数据，比较对象包括 LPA、Louvain、Ethprivacy、Metapath2Vec、BFS、Poison、Haircut、APPR。

## Evaluation

- 实验思路：评估分为可扩展性、方法对比、消融、TopN recall 和案例可视化。核心目标不是分类精度，而是“能找到多少 target，同时输出图有多小、能跑多快”。
- 评估指标：Recall、Number of nodes、Tracing depth、Runtime。Recall 定义为找出的 target nodes 占所有 target nodes 的比例；输出节点数越小越利于人工审计。
- 数据集：20 个真实案例，覆盖近 5 年 Ethereum、BNB Chain、Polygon 上的攻击、Rug pull 和诈骗；共 20 个 source nodes、约 0.87K target nodes、23.5M blocks、4.83B money transfers。target 来自安全公司专家报告和人工验证。
- 参数设置：APPR 和 TTR 使用 `α=0.15`、`ε=10^-3`；TTR 使用 `φ=10^-3`、`β=0.7`；BFS 和 Poison 限制最大深度为 2；Haircut 追踪到所有节点 dirty money 占比低于源资金 0.1%。
- 主要结果：
  - Scalability vs. performance：`ε=10^-1` 时 recall 已约 70%；`ε < 10^-3` 后 recall 增长变慢，但输出节点数快速增加，因此实验选 `ε=10^-3`。
  - Baseline comparison：TRacer recall 92.31%，输出 0.87K nodes，深度 5.05，运行 0.65h。APPR recall 71.92%，输出 0.66K nodes，深度 3.60，运行 0.51h。BFS recall 77.02%，但输出 52.50K nodes；Poison recall 70.06%，输出 41.45K nodes；Haircut recall 58.85%，输出 10.35K nodes。LPA 和 Louvain 输出分别达到 682.30K 与 111.84K nodes，Metapath2Vec 和 Ethprivacy-1 单案例超过 24h 后被提前终止。
  - Ablation：完整 TRacer 为 92.3% recall / 0.87K nodes；去掉 graph construction 里的 DeFi patterns 后，recall 降到 80.3%，输出 0.55K nodes；去掉 local community detection 后，recall 升到 95.9%，但输出暴涨到 57.5K nodes。
  - TopN recall：按 rank 审计前 `N` 个节点时，TRacer 曲线优于 Haircut 和 APPR；当 `N > 50`，TRacer 相比 APPR 有约 25% recall gain。
  - Cryptopia case：源账户持有 30.8K 被盗 ETH，专家报告标注约 10K ETH 流向 EtherDelta；TRacer 除了找到 4 个 EtherDelta target，还发现 Yobit.net 和 OKEx，分别约 1420 ETH 与 18.47K ETH，论文称覆盖超过 97% 的被盗 ETH。
  - Kucoin case：TRacer 追踪到约 13.8K ETH 流入 Tornado Cash 100 ETH pool；论文也承认进入隐私混币协议后很难继续获得有价值的下游 KYC 信息。
  - Ronin / Axie Infinity case：相比 Certik 报告中的 Tornado Cash、FTX、Huobi 流向，TRacer 还识别了 Binance、Kucoin 作为源资金相关节点，以及 Hotbit 作为额外 target exchange。

## Key Artifacts

- 关键图：
  - Fig. 1：区分 proactive risk warning 与 remedial money tracing，说明本文定位是事后追踪而非事前检测。
  - Fig. 2：TRacer 框架图，展示 Graph construction、Graph expansion、Local community detection 三段流程。
  - Fig. 3：Bitfinex 示例中 BFS 与 Poison 的追踪结果对比，用来支持 biased search 比 blind search 更实际。
  - Fig. 4：PageRank 与 personalized PageRank 在 Bitfinex 账户图上的差异，用来说明全局重要性不等于与风险源相关。
  - Fig. 5：Xfer 与 Swap 两类 DeFi action pattern，是图构建里处理 token flow 的基础。
  - Fig. 6：TTR 的四个传播策略：tracing tendency、weighted pollution、temporal reasoning、token redirection。
  - Fig. 7：`ε` 与 recall / nodes 的关系，支撑 `ε=10^-3` 的参数选择。
  - Fig. 8：TopN recall，说明 rank 可作为人工审计优先级。
  - Fig. 9-11：Cryptopia、Kucoin、Ronin 三个案例可视化，展示 TRacer 输出能形成可读资金流图。
- 关键表：
  - Table I：20 个案例的数据规模：20 source nodes、0.87K target nodes、23.5M blocks、4.83B transfers。
  - Table II：各方法 recall、输出节点数、深度、运行时间对比，TRacer 在 recall 上最高，同时输出规模远小于 BFS/Poison/LPA/Louvain。
  - Table III：消融结果，证明 DeFi pattern 提升 recall，local community detection 牺牲少量 recall 换来巨大规模压缩。
- 关键公式 / 定义 / 算法：
  - Algorithm 1：TTR local push，核心是把 residual 按方向、金额、时间和 token redirection 传播。
  - Algorithm 2：TTR-based local community detection，按 TTR score 加点直到 conductance 满足阈值。
  - Proposition 1：TRacer 成本不依赖全图规模，而依赖 `ε` 与 `α`。
  - Proposition 2：给出可追踪 hop depth 的上界。
  - Personalized PageRank / APPR appendix：说明 TTR 从 local push 思路扩展而来，但加入交易语义。
- 这些证据分别支撑哪些结论：Fig. 3-4 支撑方法选择；Fig. 5-6 与 Algorithm 1 支撑机制设计；Table II-III 支撑有效性与模块必要性；case studies 支撑实用价值，但不等于完整真实世界取证能力。

## Findings

- 发现 1：account-based chains 的资金追踪不是账户聚类问题。资金路径上的 hacker、DEX、LP pool、exchange deposit address 角色差异很大，但它们仍可能高度相关。
- 发现 2：DeFi token swap 是追踪失败的关键断点之一。只按同一种 token 传播 taint 会在 swap 处断掉，TRacer 的 token redirection 正是为了解这个问题。
- 发现 3：审计可用性与 recall 同等重要。没有 local community detection 时 recall 只有小幅提升，但节点数从 0.87K 暴涨到 57.5K，人工审计成本会失控。
- 发现 4：隐私协议是清晰边界。Kucoin case 中，一旦资金进入 Tornado Cash，TRacer 只能定位进入混币池的事实，不能可靠恢复出池后的身份线索。

## Strengths

- 论文最有说服力的地方：问题定义非常贴近真实调查流程，目标不是“全自动识别罪犯”，而是给审计人员一个更小、更相关的资金流子图。
- 方法、数据、实验或问题设定的优势：TTR 把交易方向、金额、时间和 DeFi swap 放进 local push，比只看拓扑的 APPR 更贴合链上资金追踪；消融实验也确实显示 DeFi pattern 和 local community detection 都有作用。
- 相比已有工作的有效推进：相较 BFS/Poison/Haircut，TRacer 在 recall 和输出规模之间取得更好的平衡；相较账户聚类和全图社区发现，它不需要把整个交易网络作为输入，更适合按事件做局部取证。

## Limitations

- 威胁模型、假设或适用范围的限制：方法依赖至少一个已知 source；目标账户的定义和 ground truth 依赖专家报告；对隐私增强协议、跨链桥、复杂聚合器、多 token 批量交换和合约内部账本的覆盖有限。
- 数据集、baseline、metric、ablation 或复现性的不足：论文说代码和数据在线，但正文没有详细列出 20 个案例的完整 target label 构造过程，复核 target ground truth 成本高；评价主要看 recall 和输出节点数，缺少对输出子图中无关节点比例、证据路径质量、调查人员实际工作量的直接测量。
- 在真实 DeFi 场景中可能失效的条件：若攻击者使用隐私池、跨链桥、CEX 内部账本、非标准 token 或协议级内部余额转移，Xfer/Swap 这层抽象可能不足；若 DeFi 操作在多个交易/链之间拆分，时间和 token redirection 规则也可能变脆。

## My Takeaways

- 对 DeFi / 区块链安全研究的启发：TRacer 与 DeFiRanger 互补。DeFiRanger 偏“从交易 trace 恢复 DeFi 语义并检测价格操纵”，TRacer 偏“攻击后从源账户追资金去向”。一个做语义检测，一个做事后取证候选生成。
- 可复用的方法：TTR 的四类传播偏置很值得复用到其他链上取证任务，尤其是把 token swap 作为 residual redirection，而不是把不同 token 图硬拆开。
- 可能的后续问题：把 TRacer 的图搜索与更强的 smart contract semantics 结合，例如 bytecode、event logs、函数签名、协议 ABI 和 LP/债务 token 语义；同时需要更明确的 path explanation，让每个 target 的证据链能被调查人员逐跳复核。

## Related Papers

- 前置阅读：Möser et al. 的 Bitcoin taint analysis，Andersen et al. 的 local graph partitioning using PageRank vectors，PageRank / personalized PageRank 系列。
- 后续阅读：DeFiRanger、HOUSTON、POMABuster、BERT4ETH、Following Devils' Footprint。
- 可对比论文：BFS/EGRET/Bitconeview 代表图搜索和可视化方向；Ethprivacy、Metapath2Vec、Ethereum node clustering 代表账户聚类方向；Haircut/Poison 代表 taint analysis 方向。
- 最接近的相关工作：APPR 是最直接的算法基线，因为它同样是 personalized PageRank 风格的局部排名；TRacer 的关键差异是把区块链交易语义注入 residual propagation。
- 关键差异：TRacer 不试图证明账户属于同一实体，也不直接识别攻击交易，而是围绕一个已知 source 找资金流相关子图，并把排序结果用于审计优先级。

## Open Questions

- 20 个案例的 target ground truth 是否包含所有已知交易所充值地址和混币入口？遗漏 target 会如何影响 recall 解释？
- 输出图里无关节点比例是多少？仅报告节点数还不能完全说明审计成本。
- `β=0.7` 是否对不同攻击类型、链、token 和 DeFi 结构都稳定？是否需要按 case 自适应？
- token redirection 对多跳聚合器、MEV bundle、跨链桥和 CEX 内部归集地址是否仍可靠？
- 如何把 TRacer 输出转换成可交付给交易所、执法或受害者的逐跳证据报告？
