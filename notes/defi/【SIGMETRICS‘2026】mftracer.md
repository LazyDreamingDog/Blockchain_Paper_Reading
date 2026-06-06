# 【SIGMETRICS‘2026】Shedding Light on Shadows: Automatically Tracing Illicit Money Flows on EVM-Compatible Blockchains

## Metadata

- Title: Shedding Light on Shadows: Automatically Tracing Illicit Money Flows on EVM-Compatible Blockchains
- Authors: Yicheng Huo, Yufeng Hu, Yajin Zhou, Ting Yu, Lei Wu, Cong Wang
- Venue / Year: Proceedings of the ACM on Measurement and Analysis of Computing Systems, 9(3), Article 63, December 2025; accepted to ACM SIGMETRICS 2026
- Note Name: 【SIGMETRICS‘2026】mftracer
- Paper: https://doi.org/10.1145/3771578
- Author PDF: https://yajin.org/papers/sigmetrics2026_mftracer.pdf
- Local PDF: `/Users/wenkuanxiao/Zotero/storage/U5XNXBLS/Huo 等 - 2025 - Shedding Light on Shadows Automatically Tracing Illicit Money Flows on EVM-Compatible Blockchains.pdf`
- Zotero: found in collections `污染交易` and `追踪溯源`; duplicate local attachment also exists under `/Users/wenkuanxiao/Zotero/storage/HVR6GBXP/`
- Code: https://github.com/blocksecteam/MFTracer/tree/main/codes
- Dataset / Artifact: LaunderNetEvm41 dataset at https://github.com/blocksecteam/MFTracer/tree/main/LaunderNetEvm41; newly reported address/flow findings at https://github.com/blocksecteam/MFTracer/tree/main/findings
- Scope / Subfield: 攻击溯源 / 链上资金流追踪 / AML 取证系统
- Tags: AML, DeFi, Ethereum, EVM-Compatible Blockchains, Transaction Tracing, Money Laundering, Graph Search, MFA, Dataset, Forensics
- Status: DONE

## TL;DR

这篇论文解决的是 EVM-compatible blockchains 上赃款流向自动追踪的问题：给定受害者/source 地址，系统要还原黑钱如何经过 EOA、合约、DEX、桥、服务商等路径流向下游，而不是只判断某个地址是否可疑。作者提出 MFTracer：先从交易 trace、internal transaction 和 ERC-20 Transfer log 中做 transaction-level money flow analysis，再把资金流压缩成轻量 Money Flow Abstract (MFA) 以支持高速检索，最后用时间顺序的资金余额仿真输出 illicit money flow topology。

实验上，MFTracer 在两年 Ethereum 数据、8.317 亿笔交易上构建基础设施；MFA 两年总内存占用 17.2 GB，全量基础设施磁盘占用 107.1 GB，相比 Neo4j/Elasticsearch 等通用后端有 3.7-9.4x 存储效率和 14.1-300.0x 检索速度优势。效果评估使用作者构建的 LaunderNetEvm41，包含 41 个案件、1,939 个洗钱账户、6,701 条赃款流记录、超过 1.25 亿美元资产；MFTracer 达到 94.09% flow coverage、92.43% address-level terminal coverage、80.37% precision，并新报告 686 个地址和 4,183 条资金流。

## 毒舌评论

MFTracer 最强的地方不是“图搜索”，而是把链上取证里最无聊也最要命的工程问题做对了：交易级价值流解析、轻量可检索图、按时间顺序仿真。它的论文叙事有点把系统包装成“自动追踪神器”，但真正的软肋很清楚：ground truth 仍来自专家手工取证，效果也没有和一个可部署 baseline 正面对打；系统能把候选证据链大幅压小，却不能替代调查员判断某条链是否真的能上法庭。

## Research Question

- 研究对象：EVM-compatible blockchains 上由 phishing、合约攻击、DeFi exploit、私钥盗窃、社工诈骗等造成的非法资金流。
- 小领域范围：链上 AML、攻击后资金流取证、自动化 tracing system、大规模链上资金流数据基础设施。
- 具体问题：给定 victim/source addresses `A_vic`，如何在海量 EVM 交易中自动重建 illicit money flow topology `G=(V,F)`，即哪些地址参与了黑钱转移，以及黑钱在它们之间如何流动。
- 为什么重要：真实调查需要知道黑钱“怎么走”和“最后去哪”，以便交易所协作、账户冻结、资产追回和法律证据构建；但 EVM 的合约语义、多 token、DeFi 交互和大规模数据让人工追踪昂贵且慢。
- 论文边界：MFTracer 追的是已知 victim/source 地址之后的下游流向，不解决未知攻击发现；隐私混币池、CEX 内部账本、off-chain OTC 和零知识 mixer 之后的重新链接不由 MFTracer 单独解决；输出仍需人工复核。

## Motivation and Basic Idea

- Motivation：已有 AML 研究多数做 anomaly detection 或 illicit account classification；Bitcoin 方向的 taint/heuristic tracing 又依赖 UTXO 场景和固定 laundering pattern。EVM 场景下，一笔交易可以通过合约、internal transaction、ERC-20 event、DEX swap、add liquidity、wrapped token 和自部署合约形成复杂价值流；同时真实追踪任务面对的是千万级 suspicious flows，通用图数据库很难满足调查时效。
- Basic idea：把“链上交易记录”先转成标准化、可仿真的 money flow，再把全链资金流做成 build-once, reuse-indefinitely 的检索基础设施。真正调查时，只从 source addresses 之后的时间片加载相关 MFA，先用图搜索找到 suspicious topology，再取回详细 flows，用余额仿真过滤无关地址和边。
- 这个 idea 如何回应 motivation：transaction-level analysis 解决“EVM 交易内部真实资金提供者/接收者是谁”的问题；MFA 和 key-value storage 解决“海量链上数据怎么高效取出来”的问题；simulation 解决“可达不等于黑钱真的流过”的问题。
- 作者给出的证据：Fig. 1 的 Multi-call DEX aggregator 案例展示了同一交易中 wrap、swap、add liquidity 和自部署合约如何掩盖底层流向；Challenge-II 中的 Discord/X scam 例子说明 15-hop BFS 下游空间可达到 5,640 万 token transfers；Table 1/3 证明检索和效果指标。
- 我的判断：论文动机成立，而且比 TRacer / XBlockFlow 更靠近真实部署。它把“候选子图搜索”推进到“取证系统基础设施”，这比再做一个 graph ranking 模型更有实用价值。

## Background

- 背景：EVM-compatible chains 包含 EOA 和 contract account；外部交易由 EOA 发起，internal transaction 由合约执行产生；ERC-20 token 通过 Transfer event log 暴露 token type、amount、from/to 和时间；price oracle 用于把多 token 价值归一到 USD。
- 问题：犯罪者不能直接从上游地址 cash out，通常会通过多层地址、合约、DEX、桥和服务商把 upstream illicit money 变成看似干净的 downstream money。
- Gap：检测可疑行为不等于恢复资金链；Bitcoin tracing 方法难处理 EVM 合约语义；把 raw transfers 直接扔进通用 graph database 又会在存储和检索上失控。

## Threat Model / Assumptions

- 攻击者或参与者能力：攻击者控制 victim/source 之后的若干地址或合约，可进行多跳转账、跨 token swap、LP 操作、桥接、服务商转入、微额拆分和自部署合约混淆。
- 链上 / 链下假设：EVM execution traces、internal transactions、ERC-20 Transfer logs 和 token price oracle 可获取；source/victim addresses 和 attack time 已知；链上时间顺序足以约束黑钱传播。
- 市场、流动性、排序、预言机或网络假设：系统把多 token 价值转成 USD 余额进行仿真，隐含依赖价格 oracle 在交易时间点足够准确；不建模 MEV、滑点细节、CEX 内部账本和 off-chain 价值交换。
- 不覆盖的情况：Tornado Cash 这类零知识 mixer 会切断链上可链接性；NFT、跨链桥、非 EVM 链需要额外 parser 或外部工具；长期持币导致 token price drift，可能产生残余 USD 余额和 false positives。

## Method

- 核心思路：`Building Process -> Tracing Process`。Building Process 一次性处理给定时间窗口内所有链上交易，生成 MFA 和 detailed money flows 的 on-disk infrastructure；Tracing Process 面向单个案件，从 victim/source addresses 出发检索、搜索、仿真并输出 illicit money flow topology。
- 核心思路是怎么想到的：作者把两个真实痛点拆开处理。第一，EVM 交易的“显式转账边”不等于真实价值流，所以要先在交易内部推断 net source 和 net target。第二，真实调查需要反复查询海量历史数据，所以不能每个案子现抓全量数据，而要预构建可复用基础设施。
- 从 motivation 到 method 的逻辑链：EVM 资金流复杂 -> 解析交易内部所有 fungible token transfers -> 用 USD 余额变化和 max-flow 得到 transaction-level underlying flows -> 把 flows 压缩进 MFA 以便可达性搜索 -> 按时间分片和 key-value 编码降低加载成本 -> 对 suspicious flows 做余额仿真，得到真实黑钱拓扑。
- 关键设计取舍：MFTracer 选择 protocol-agnostic 解析，不硬编码特定 DEX/桥协议；代价是它依赖 USD value matching，面对价格波动、滑点和长期持币会有 valuation drift。MFA 只保留地址可达性和最小/最大时间戳，牺牲部分细节以换取内存可驻留和高速搜索；详细流信息另存，用到时再取。
- 为什么不是更直接 / 更简单的方案：直接看 raw external transfers 会漏掉 internal transfers 和 ERC-20 flows；直接按 token log 建边会把协议中间节点、LP pool、aggregator 当成真实资金终点；直接 BFS 可达性会引入大量 clean flows；通用图数据库没有针对区块链 tracing 的时间分片、压缩和按 pair flow retrieval。
- 系统流程或算法步骤：
  - Tx-Granularity Money Flow Analysis：对每笔交易收集 native transfer、internal transaction、ERC-20 Transfer log；用 oracle 把 token amount 转成 USD；构建 local money transfer graph `G_L` 和 balance change table `B`；`B[a]<0` 是 net source，`B[a]>0` 是 net target；对每组 source-target 运行 max-flow，输出 underlying money flow。
  - MFA Construction：从 transaction-level flows 构造 `G_mfa=(V,E,T)`。`V` 是地址，`E` 是存在资金流的有向边，`T` 存储每个地址对的 timestamp 集合 `tau_{u,v}`。
  - MFA Data Structure：`structure(G_mfa)=(M,P,N,T_min,T_max)`。`M` 把地址映射到整数 index；`P` 和 `N` 类似 CSR 邻接数组；`T_min/T_max` 为每条压缩边保存最早/最晚时间。
  - Graph Search and Pruning：从 `A_vic` 在相关 MFA slices 上做下游搜索，得到 suspicious topology `G_s=(V_s,F_s)`；搜索时用时间不等式剪枝，保证路径上资金流时间不倒流。
  - Encoding Scheme：用 Pebble LSM-tree key-value storage 存储 MFA slices 和 detailed flows。按 `K` 个连续 blocks 分片；实验中 `K=100,000`，两年 Ethereum 数据切成 54 个 subsets。
  - Money Flow Simulation：把 suspicious flows 按时间排序，维护地址 illicit balance `B`。source 地址释放全部 flow；中间地址最多转出其当前 illicit balance 减去 reserve；若 outflow 小于 threshold 或 from 地址未见过黑钱，则过滤。输出 `G=(V,F)`。
  - Parallel Search：时间分片天然支持并行搜索；Appendix C 给出 Algorithm 3，证明并行结果与串行结果等价，并在相似任务成本假设下把时间从 `O(Ck^2)` 降到 `O(Ck)`。
- 关键定义 / 公式 / 不变量：
  - Transaction-level flow upper bound:

```text
flowThru = G_L.maxFlow(saddr, taddr)
ubound = min(-B[saddr], flowThru, B[taddr])
```

  - MFA:

```text
G_mfa = (V, E, T)
structure(G_mfa) = (M, P, N, T_min, T_max)
```

  - Time-based pruning condition for a valid path:

```text
max tau_{a_i,a_{i+1}} >= max_{0<j<i} min tau_{a_j,a_{j+1}}
```

  - FlowCoverage:

```text
FlowCoverage =
  sum_{u,v in V intersect V_g} F(u,v) * M_g(u,v)
  / sum_{u,v in V_g} M_g(u,v)
```

  - TerminalCoverage:

```text
TerminalCoverage =
  sum_{u in V intersect V_g, v in V intersect V_t} F(u,v) * M_g(u,v)
  / sum_{u in V_g, v in V_t} M_g(u,v)
```

- 实现细节：系统用 12,381 行 Go 实现；基础设施构建依赖 transaction traces 和 token prices；代码、LaunderNetEvm41 数据集、新报告地址/交易分别在 `blocksecteam/MFTracer` 的 `codes`、`LaunderNetEvm41`、`findings` 目录。

## Evaluation

- 实验思路：评估分成 efficiency 和 effectiveness。Efficiency 看基础设施构建、内存/磁盘占用、数据检索速度；Effectiveness 用 LaunderNetEvm41 ground truth 测 flow coverage、terminal coverage 和 precision。
- 评估环境：三台相同机器，每台为双 Intel Xeon Gold 5318Y、128 GB DRAM、3 块 1 TB SSD、Ubuntu 22.04.1；Neo4j 和 Elasticsearch 用三节点部署以做检索速度对比。
- 数据规模：
  - Efficiency：Ethereum 2022-08-08 到 2024-09-07，blocks 15,300,000 到 20,700,000，共 831.7M transactions；`K=100,000`，54 个 subsets，平均每 subset 约 15.4M transactions。
  - Effectiveness：LaunderNetEvm41 含 41 个 laundering cases，来自 TxPhish、LIFI、Atomic、Harmony Bridge、XScam 等顶层事件；1,939 accounts、6,701 illicit flow records，覆盖超过 125M USD stolen assets。
- 评估指标：
  - Building speed：每 subset 构建时间。
  - Storage efficiency：MFA 内存占用、完整基础设施磁盘占用，并与 Neo4j/Memgraph/RedisGraph 比较。
  - Retrieval speed：每 ms retrieve flows 数量。
  - FlowCoverage：系统输出覆盖 ground-truth illicit fund value 的比例。
  - TerminalCoverage：系统追踪到 terminal/cash-out endpoints 的比例。
  - Precision：`|V intersect V_g| / |V|`，即输出地址中真实 laundering addresses 的比例。
- 主要结果：
  - Building Process：平均每个约 15.4M transactions 的 subset 处理 394.8 秒，而生成这些区块约需 11.8 天；作者称基础设施构建速度约比区块生成速率快 2,600x，可支持实时更新。
  - MFA storage：两年 54 个 MFA instances 总内存 17.2 GB，平均 327.1 MB；保守估计每个 400 MB 时，90 GB RAM 可容纳 230 多个 subsets，足以覆盖 Ethereum 两年数据。
  - Full infrastructure storage：包括 detailed money flows 和 MFA 的完整基础设施占 107.1 GB，平均每 subset 1.98 GB；同数据 Neo4j graph storage 为 1010.9 GB。Memgraph 和 RedisGraph 均超过 400 GB，三台机器合计内存也放不下。
  - Retrieval speed：MFTracer 为 90.02 flows/ms，Neo4j 为 6.37 flows/ms，Elasticsearch 为 0.30 flows/ms，对应 14.13x 和 300.0x speedup。作者称实际案件中 suspicious flows 常到 `10^7` 量级，Neo4j 仅加载数据就超过 4 小时，而 MFTracer 可在 15 分钟内完成。
  - Overall effectiveness：`epsilon=0` 时平均 FlowCoverage 94.09%；asset-value terminal coverage 96.27%，address-level terminal coverage 92.43%；平均 Precision 80.37%。
  - Incident-level results：Table 3 显示 HB 在多个 epsilon 下 Precision 为 1.0；LIFI 因路径很深且与流行服务交互复杂，precision 较低，但在 stress-test 下仍保持 >99% flow coverage、>60% precision。
  - Practical findings：MFTracer 新报告 686 个 laundering addresses 和 4,183 条此前未发现的 illicit fund movements；重建了价值 120.9M USD 的完整流向证据。
  - Parameter guidance：`K` 取决于线程/内存权衡，作者建议每个 MFA instance 控制在 200-500 MB；reserve ratio `epsilon` 实践中 5%-10% 较优；simulation threshold 建议为涉案金额的 0.01%-0.1%。
  - Other chains：Table 4 显示 BSC transfer volume 为 Ethereum 的 2.34x；作者估计两年 BSC MFA 内存约 40.2 GB，且 MFTracer 39,013 TPS 处理率远高于 BSC 约 150 TPS 的最高生产速率。

## Key Artifacts

- 关键图：
  - Fig. 1：真实 Multi-call DEX aggregator laundering example，展示 wrap、swap、add liquidity、自部署合约如何把 14,400 USD illicit assets 拆到 EOA 1 和 criminal-controlled smart contract。
  - Fig. 2：MFTracer overview，把 Building Process 与 Tracing Process 分开，是全文系统设计主图。
  - Fig. 3：Suspicious topology example，说明时间剪枝和 simulation 如何过滤 innocent addresses `x,y,v1,v2`。
  - Fig. 4：Building time 与 MFA size，对 RQ1/RQ2 的效率结论给证据。
  - Fig. 5/6：Appendix 中 Fig. 1 的展开版本和 local money transfer graph / balance table，解释 Algorithm 1 如何从复杂交易中恢复 source-target fund flows。
  - Fig. 7：reserve ratio `epsilon` 对 FlowCoverage、TerminusCoverage、Precision 的影响，支撑参数建议。
- 关键表：
  - Table 1：检索速度对比，MFTracer 90.02 flows/ms，Neo4j 6.37，Elasticsearch 0.30。
  - Table 2：五类顶层 cybercrime incidents，TxPhish 54M/151 addresses/292 records，LIFI 11M/93/135，Atomic 10M/389/1080，HB 50M/469/3887，XScam 583k/837/1307。
  - Table 3：按事件和 `epsilon` 分解 FlowCoverage、TerminalCoverage、Precision。
  - Table 4：六条 EVM-compatible chains 上 external native、internal native、ERC-20 transfers 总量，用来论证 Ethereum/BSC 规模下仍可部署。
- 关键公式 / 定义 / 算法：
  - Algorithm 1：Tx-Granularity Money Flow Analysis，用 local graph + balance table + max-flow 找每笔交易的 underlying money flows。
  - Algorithm 2：Money Flow Simulation，用 illicit balance、reserve ratio 和 threshold 过滤 clean/irrelevant flows。
  - Algorithm 3：Parallel Search on MFAs，说明按时间分片并行搜索的正确性。
  - FlowCoverage / TerminalCoverage / Precision 是效果评估核心。
  - Reserve-ratio analysis 给出 depth/breadth 与 `epsilon` 的关系：较低 `epsilon` 增加深度，较高 `epsilon` 增加宽度。
- 这些证据分别支撑哪些结论：Fig. 1/5/6 支撑 EVM transaction-level parsing 的必要性；Fig. 2/4 和 Table 1 支撑系统基础设施效率；Table 2/3 和 LaunderNetEvm41 支撑真实案件效果；Fig. 7 和 Section 5 支撑参数可调与 false-positive 解释。

## Findings

- 发现 1：EVM 资金追踪的关键断点往往在单笔交易内部。仅凭 external transfer 或 ERC-20 log 的表层边，很容易把 DEX aggregator、LP pool、wETH contract、attacker contract 的角色看错。
- 发现 2：工程基础设施是 AML tracing 的一等问题。对于 `10^7` 级 suspicious flows，算法本身再漂亮，如果数据加载要数小时，就不适合调查场景。
- 发现 3：MFA 的核心价值是“只保留搜索需要的信息”：地址 index、邻居数组、边时间上下界。它不是完整资金图，而是用于快速缩小 suspicious topology 的内存索引。
- 发现 4：simulation 的作用是把“可达路径”转成“黑钱余额真的能流过的路径”。这一步用物理上不超过已收到 illicit balance 的约束，避免把 clean balance 误当成黑钱。
- 发现 5：高 coverage 比极高 precision 更贴合取证。作者明确认为上法庭前总要人工复核，漏掉关键资金链比多给一些候选地址更危险。

## Strengths

- 论文最有说服力的地方：它把真实取证系统需要的三件事连起来了：交易语义解析、可部署数据基础设施、面向案件的资金流仿真，而不是只做离线分类或小图搜索。
- 方法、数据、实验或问题设定的优势：LaunderNetEvm41 的人工构建过程很重，包含 41 个真实案件和逐条 flow record，比只标 illicit/licit 地址的数据更适合 tracing evaluation；效率实验也不是玩具数据，而是两年 Ethereum 全量交易。
- 相比已有工作的有效推进：相较 XBlockFlow，它不只是分析 Ethereum heist 行为，而是提出可重复追踪的系统；相较 TRacer，它不依赖 ranking-guided local community 输出候选子图，而是进一步做 transaction-level value-flow reconstruction 和基础设施优化；相较 Bitcoin taint work，它覆盖 EVM smart contracts 和多 token。

## Limitations

- 威胁模型、假设或适用范围的限制：必须已知 victim/source addresses；进入 ZK mixer 或 CEX 内部账本后，MFTracer 本身无法重连输入输出；非 EVM 链、NFT、桥接需要额外 parser/外部工具。
- 数据集、baseline、metric、ablation 或复现性的不足：论文称没有真正可部署 baseline，因此 effectiveness 没有和 TRacer/XBlockFlow/taint 系统在同一大规模数据上比较；ground truth 由专家和 BlockSec 参与构建，质量很高但独立复核成本大；代码开源不等于全量复现实验容易，因为需要 traces、price oracle 数据和较重硬件。
- 在真实 DeFi 场景中可能失效的条件：USD-denominated balance 在长期持币、剧烈价格波动、滑点、LP 损益、低流动性 token 中会产生 valuation drift；threshold 过大时会漏掉微额拆分；address labeling 不足时，公共服务地址可能被误报为 laundering participants。

## My Takeaways

- 对 DeFi / 区块链安全研究的启发：这篇论文把“攻击后资金追踪”从算法问题拉回系统问题。后续如果只提出一个新 ranking 或 GNN，而不解释数据如何组织、检索和复核，会显得很薄。
- 可复用的方法：`transaction trace -> net balance table -> source/target max-flow -> compressed MFA -> time-ordered simulation` 这条 pipeline 可复用到 bridge tracing、attack fund recovery、phishing campaign analysis 和 CEX risk scoring。
- 可能的后续问题：用 token-specific ledger 替代 USD balance 以降低 valuation drift；把 address labels、protocol semantics、bridge parsers 接进 simulation；输出更强的 path explanation，让每条边都能附带交易哈希、token、金额和为什么是 illicit 的证据。

## Related Papers

- 前置阅读：TRacer、XBlockFlow、Bitcoin taint/risk scoring work、EGRET、Meiklejohn et al. 的 Bitcoin payment characterization。
- 后续阅读：GuideEnricher、Tornado Cash deanonymization work、MetaSleuth/Phalcon 类实务工具、跨链 bridge tracing 工作。
- 可对比论文：XBlockFlow 更偏 Ethereum heist 洗钱行为画像和数据集；TRacer 更偏 ranking-guided local search；MFTracer 更偏可部署 tracing infrastructure 和 transaction-level value-flow simulation。
- 最接近的相关工作：TRacer 和 XBlockFlow。TRacer 解决“从 source 找相关子图并排序”，XBlockFlow 解决“Ethereum heist 洗钱行为如何分析”；MFTracer 的关键差异是 protocol-agnostic transaction-level fund flow analysis + MFA storage backend + simulation。
- 关键差异：MFTracer 不依赖 predefined laundering patterns，也不以节点分类为目标；它把黑钱追踪定义为从已知 source 恢复 illicit money flow topology，并强调效率足够支撑实际调查。

## Open Questions

- LaunderNetEvm41 的 ground truth 是否会随着后续安全社区发现更新？如果有遗漏，94.09% coverage 的解释会如何变化？
- transaction-level max-flow 得到的是 underlying flow upper bound；在复杂 DeFi 交易里，这个 upper bound 与法律意义上的“黑钱实际流向”是否总能对齐？
- Token-specific ledger 会带来多大存储和检索开销？是否仍能保持 3.7-9.4x storage advantage？
- 对桥接、mixer、CEX 之后的断点，MFTracer 和 off-chain/side-channel 工具如何组合成可审计证据链？
- 输出 precision 80.37% 已经可用，但人工复核“几十分钟”的说法是否有真实 user study 或调查员操作日志支撑？
