# 【WWW‘2023】BERT4ETH: A Pre-trained Transformer for Ethereum Fraud Detection

## Metadata

- Title: BERT4ETH: A Pre-trained Transformer for Ethereum Fraud Detection
- Authors: Sihao Hu, Zhen Zhang, Bingqiao Luo, Shengliang Lu, Bingsheng He, Ling Liu
- Venue / Year: Proceedings of the ACM Web Conference 2023 (WWW 2023)
- Note Name: 【WWW‘2023】bert4eth
- Paper: https://doi.org/10.1145/3543507.3583345
- Local PDF: `/Users/wenkuanxiao/Zotero/storage/U46GEGXT/Hu 等 - 2023 - BERT4ETH A Pre-trained Transformer for Ethereum Fraud Detection.pdf`
- Code: https://github.com/git-disl/BERT4ETH
- Dataset / Artifact: 论文称发布 code and dataset；Zotero/PDF 中给出的公开入口是 code repository。
- Scope / Subfield: 全范围恶意检测
- Tags: Ethereum, Fraud Detection, Phishing Detection, De-anonymization, Account Representation, Transformer, BERT, Transaction Analysis, Graph Baselines
- Status: DONE

## TL;DR

BERT4ETH 把 Ethereum EOA 的交易历史视为时间序列，用类 BERT 的 Masked Address Prediction 预训练 Transformer encoder，从交易序列中抽取账户表示，用于 phishing account detection 和 de-anonymization。论文的核心不是简单套 BERT，而是针对 Ethereum 交易的三个数据特性 - 高重复、长尾偏斜、交易异质性 - 设计去重/高掩码、频率感知负采样、in/out 分离和 ERC-20 log encoder。实验上，它在 phishing 检测和 ENS/Tornado 去匿名化上明显优于 DeepWalk/Trans2Vec/Dif2Vec/Role2Vec/GCN/GraphSAGE/GAT 等图方法，说明交易序列建模在账户行为表示上有独立价值。

## 毒舌评论

这篇论文最有价值的地方是敢于指出“Ethereum 账户交互不一定非得先压成图”。它真正赢的是数据建模假设：图方法把多次交易、顺序和账户动态状态压扁，BERT4ETH 则直接吃交易序列。但它也没有神到“通用 fraud detector”的程度 - 下游只验证了 phishing 和两类去匿名化，标签来自 Etherscan/ENS/Tornado，且很多关键收益来自非常任务化的预训练技巧和大规模链数据工程。换句话说，它是一个很强的账户表示学习 baseline，不是一个已经证明能泛化到所有 Ethereum fraud 的检测系统。

## Research Question

- 研究对象：Ethereum EOA 的交易历史和账户表示。
- 小领域范围：transaction-sequence representation learning for fraud detection。
- 具体问题：相比把账户交互建成图，能否用预训练 Transformer 更好地捕捉交易顺序、动态状态和异质交易信息，并服务于多个 fraud detection 任务。
- 为什么重要：Ethereum fraud 包括 phishing、Ponzi、ICO scam、洗钱、bot arbitrage 等，账户行为随交易不断变化；图方法常把多边合并成单边、把账户视为固定节点，容易丢掉顺序和状态变化。
- 论文边界：论文主要评估 phishing account detection 与 de-anonymization，不覆盖 DeFi 价格操纵、智能合约漏洞定位、跨链攻击或实时部署检测。

## Motivation and Basic Idea

- Motivation：已有 Ethereum fraud detection 多依赖图表示学习，但交易数据有三个对图方法不友好的特点：同一地址间交易高度重复、地址出现频率服从 power-law、交易类型/账户类型/金额/时间/方向高度异质。图方法为了计算方便常合并多边，导致顺序信息、账户动态状态和细粒度交易语义被弱化。
- Basic idea：把一个 EOA 的交易历史构造成按时间排序的 transaction sequence，插入代表自身的 dummy self-transaction，再用 Transformer encoder 做序列表示；预训练任务是 Masked Address Prediction，即遮住若干交易中的对手方地址，让模型根据上下文预测地址。
- 这个 idea 如何回应 motivation：序列模型天然保留交易顺序和重复模式；self-attention 能在长交易历史中选择相关交易；预训练得到的 self-transaction 表示可作为账户级 representation，迁移到不同 fraud detection 任务。
- 作者给出的证据：高重复统计为 48.4%；去重后重复率从 48.0% 降到 14.3%；Fig. 3 中 10% masking 的 phishing F1 约为 0.1243，原始 BERT 常用的 15% masking 为 0.1350，而 80% masking 升到 0.3245；频率感知负采样和 intra-batch sharing 把固定训练 phishing F1 推到 0.5044。
- 我的判断：动机和方法之间的逻辑链比较扎实。论文不是直接说 Transformer 更强，而是具体解释了为什么原始 BERT 预训练会被 Ethereum 交易的重复和长尾分布破坏，再给出对应补丁。

## Background

- 背景：Ethereum 账户分为 EOA 和 contract account。EOA 由私钥控制，可发起 external transactions；contract account 由代码控制，可通过合约逻辑发起 internal transactions。论文主要建模 EOA，因为它们更直接反映人或攻击者的行为。
- 问题：Ethereum transaction graph 是多边、动态、异质的。一个 EOA 会反复与交易所、DEX、token contract、受害者或攻击者控制账户交互，简单图聚合会丢失“何时、以何种方向、多少金额、连续交互了几次”这些信息。
- Gap：DeepWalk/Trans2Vec/Dif2Vec/Role2Vec 和 GNN 方法可学习账户图表示，但对交易序列、动态状态和多任务预训练支持不足。

## Threat Model / Assumptions

- 攻击者或参与者能力：论文没有给出传统安全威胁模型；默认 fraud accounts 在链上留下可观测交易历史，且这些历史中包含可被表示学习利用的行为模式。
- 链上 / 链下假设：可运行 Geth archive node，并用 Ethereum-ETL 提取 external transactions 和 trace/log 数据；可获得 Etherscan phishing 标签、ENS/Tornado ground-truth pairs。
- 市场、流动性、排序、预言机或网络假设：N/A。本文不是 DeFi 市场攻击或 MEV 论文。
- 不覆盖的情况：交易历史过短或过长被过滤；只用链上交易行为难以覆盖完全链下组织关系；新的 fraud 类型若行为模式与训练任务不同，泛化能力未被证明。

## Method

- 核心思路：transaction sequence generation -> embedding layer -> Transformer encoder -> Masked Address Prediction pre-training -> downstream representation extraction。
- 核心思路是怎么想到的：如果 fraud detection 依赖账户行为模式，而行为模式体现在一串带方向、金额、时间、账户类型的交易中，那么“账户表示”应从交易序列而不是压缩图邻域中学习。BERT 的遮蔽预测提供了一个不用 fraud 标签也能预训练的自监督任务。
- 从 motivation 到 method 的逻辑链：图方法丢顺序和动态 -> 用交易序列保留原始行为 -> BERT/MAP 学 co-occurrence 和上下文 -> Ethereum 的重复/长尾/异质性会破坏普通 BERT -> 加 RR/SA/MH 三组策略修正。
- 关键设计取舍：
  - 不把所有地址 softmax，因为 Ethereum 地址数量可达十亿级；改用 positive address + negative addresses 的 contrastive loss。
  - 不直接保留所有连续重复交易；先做 transaction de-duplication，以降低 label leakage。
  - 不只用原始完整序列；为 phishing 等任务额外构造 in-type 和 out-type 子序列，让少数方向性信号不被主序列淹没。
- 为什么不是更直接 / 更简单的方案：直接套 BERT 15% masking 会让重复地址泄露标签，任务过于简单，预训练梯度弱；直接用图方法会把多边合并并弱化交易顺序；直接把高频地址当正常 token 训练会让大量账户表示向 Uniswap 等热门地址聚拢。
- 系统流程或算法步骤：
  - 对 EOA `A0`，收集其作为发送方或接收方参与的所有交易，按 timestamp 降序排序。
  - 每笔交易抽取 address、timestamp/amount、account type、in/out type；再加入 count 和 position；amount/count 用 binning 离散化。
  - 在序列头插入 dummy self-transaction，address 为 `A0`，其他特征为 `Null`。
  - 去除 failed transactions，并将 72 小时内连续重复、地址相同、in/out type 相同的交易聚合，金额求和并记录 count。
  - 对一部分交易地址置为 `[MASK]`，用 Transformer 输出的 masked transaction hidden state 预测被遮住的 address embedding。
  - 下游任务中取 self-transaction 的最终表示作为账户表示；若序列超过最大长度，则切成多个序列，分别编码后 mean pooling。
- 关键定义 / 公式 / 不变量：
  - 初始交易表示：第 `i` 笔交易的七类 feature embeddings 相加，得到 `h_i^(0) in R^d`；整段序列为 `H^(0) = [h_0^(0), ..., h_{N-1}^(0)] in R^{N x d}`。
  - Transformer layer：`H^(l) = Attention(H^(l) W_Q^(l), H^(l) W_K^(l), H^(l) W_V^(l))`，`Attention(Q,K,V)=softmax(QK^T/sqrt(d))V`，再接 position-wise FFN。
  - MAP contrastive loss：
    `L = -1/|M| * sum_{m in M} log(exp(h_m^T a_p) / (exp(h_m^T a_p) + sum_{n in N} exp(h_m^T a_n)))`。
    其中 `M` 是 masked address 集合，`h_m` 是 masked transaction 的上下文表示，`a_p` 是真实地址 embedding，`a_n` 是负采样地址 embedding。
  - Zipfan negative sampling：`P_neg(A_i) = (log(r(A_i)+2) - log(r(A_i)+1)) / log(max+1)`，`r(.)` 是按频率降序的 rank。
  - Frequent negative sampling：`P_neg(A_i) = f(A_i)^b / sum_j f(A_j)^b`，`b=0` 时退化为 uniform sampling。
  - ERC-20 log encoder：先对接收 ERC-20 token 的 EOA 地址 embedding 做 mean pooling 得到 `a_u`，再用 gate `beta = Sigmoid(W_beta * [a_c || a_u] + b_beta)` 融合 contract address embedding：`a_c' = beta * a_c + (1 - beta) * a_u`。
- 实现细节：BERT4ETH 使用 8 层 Transformer、2 个 attention heads、最大序列长度 100、hidden dimension 64、batch size 256、dropout 20%。预训练样本为随机 1,000,000 个普通 EOAs，过滤 phishing/exchange/miner/mining pool，交易覆盖 2017-01-01 到 2022-05-01，并过滤交易数少于 3 或多于 10,000 的 EOAs。

## Evaluation

- 实验思路：用 phishing account detection 验证表示的分类能力，用 ENS/Tornado de-anonymization 验证表示的近邻检索能力；同时与 DeepWalk-based、GNN-based 和 BERT4ETH variants 比较，并做五类 ablation。
- 评估指标：phishing detection 使用 Precision、Recall、F1；de-anonymization 使用 HR@k 和 Tornado candidate set 中目标 deposit account 的平均 Rank。
- 主要结果：
  - 数据规模：phishing 标签账户 7,057 个，其中 97% 是 EOA；de-anonymization 使用 ENS 和 Tornado Cash ground-truth pairs；Table 2 中 phishing/ENS/Tornado/Normal 分别含 3,220/1,335/2,301/594,038 个 EOA，以及 328,261/821,140/1,056,674/11,350,640 笔交易。
  - Phishing fixed-training：basic BERT4ETH F1 为 0.5044，高于最佳 GAT 0.2883；加入 in/out separation 的 BERT4ETH 变体 F1 为 0.5388，precision 0.5826，recall 0.5012；ERC-20 log encoder 变体 F1 为 0.5212。
  - Phishing fine-tuning：in/out 变体最佳，precision 0.7421、recall 0.6125、F1 0.6711。无预训练时相当于普通 Transformer，最佳 F1 只有 0.4637，说明预训练贡献显著。
  - ENS de-anonymization：basic BERT4ETH HR@1 为 16.32%，in/out 变体为 20.14%，明显高于 GraphSAGE 5.90%、Trans2Vec 6.60%、Dif2Vec 3.82%；in/out 变体 HR@1000 达 68.75%。
  - Tornado de-anonymization：0.1 ETH mixer 上 basic BERT4ETH HR@1/HR@3 为 40.20%/56.86%，in/out 变体为 51.96%/58.64%；1 ETH mixer 上 in/out 变体 HR@1/HR@3 为 53.75%/66.25%。
  - Ablation for phishing：去掉 80% masking 后 F1 从 0.5044 降到 0.3073，是最大跌幅；去掉 de-duplication、freq sampling、batch sharing、transaction features 后 F1 分别为 0.3857、0.3804、0.4239、0.4217。
  - Ablation for ENS de-anonymization：去掉 freq sampling 后 HR@1 从 16.32% 降到 4.52%，去掉 batch sharing 后降到 0.69%；相反，去掉 de-duplication 或 80% masking 在 HR@1 上略升，说明去匿名化任务不像 phishing 那样怕高重复。

## Key Artifacts

- 关键图：
  - Fig. 1：地址出现频率服从 power-law distribution，是 skew alleviation 的核心动机。
  - Fig. 2：BERT4ETH 预训练框架，展示交易序列、七类 feature embeddings、Transformer encoder、masked address prediction 的连接方式。
  - Fig. 3：masking ratio 与 phishing F1 的关系，80% masking 最优，说明原始 BERT 的 15% masking 在 Ethereum 重复交易上会导致 label leakage。
  - Fig. 4：uniform 与 Zipfan negative sampling 下第一层 attention 分布；Zipfan 降低高频地址 attention，让模型更关注低频地址。
  - Fig. 5：phishing/normal 账户的 t-SNE 可视化；BERT4ETH 生成的 phishing 表示更集中、更可分。
- 关键表：
  - Table 1：skew alleviation 对 phishing F1 的影响；Zipfan + intra-batch sharing + 1:5000 P/N ratio 达到 0.5044。
  - Table 2：数据集统计，说明 pre-training 与两类 downstream tasks 的规模。
  - Table 3/4：phishing detection 的 fixed-training 与 fine-tuning 对比。
  - Table 5/6：ENS/Tornado 去匿名化对比，说明 BERT4ETH 在近邻检索任务上优于图表示。
  - Table 7/8：ablation study，显示 RR 对 phishing 关键，SA 对 de-anonymization 关键。
  - Table 9：hot-to-cold de-anonymization case study。Dif2Vec 在 hot-to-cold 查询中排名 47,641/49,059，而 BERT4ETH 排名 20/54，说明图邻域噪声对热账户查询很致命。
- 关键公式 / 定义 / 算法：
  - MAP contrastive loss 是预训练目标核心。
  - Zipfan/Frequent sampling 是 skew alleviation 的核心机制。
  - ERC-20 transfer log gate 解释了如何把 token transfer trace 融入外部交易序列。
- 这些证据分别支撑哪些结论：Fig. 1/3 和 Table 1/7 支撑“Ethereum 数据特性需要专门预训练策略”；Table 3-6 支撑“序列表示强于图表示”；Table 8/9 支撑“长尾/热账户噪声是去匿名化的关键困难”。

## Findings

- 发现 1：Ethereum fraud account representation 不能只看图结构，交易顺序和交易方向在 phishing 与 de-anonymization 中都有实证价值。
- 发现 2：普通 BERT 的预训练设定不能原样迁移到链上交易。高重复会造成 masked address label leakage，长尾分布会把表示推向热门地址，因此 80% masking、去重、频率感知负采样不是锦上添花，而是模型可用的前提。
- 发现 3：不同下游任务依赖不同数据特性。Phishing 检测强依赖 repetitiveness reduction；ENS de-anonymization 更依赖 skew alleviation，甚至不一定从去重和高 masking 中受益。

## Strengths

- 论文最有说服力的地方：问题拆解细，且每个数据特性都有对应机制和 ablation。它没有把 Transformer 当黑盒，而是说明了为什么 Ethereum 交易会让普通 BERT 失效。
- 方法、数据、实验或问题设定的优势：同时评估二分类和近邻检索两类任务，覆盖 phishing、ENS、Tornado；baseline 包括 DeepWalk-based 和 GNN-based 两条图方法路线；case study 能解释图方法在热账户查询中的失败。
- 相比已有工作的有效推进：Trans2Vec 等方法仍围绕图随机游走，GNN 则面临多跳噪声和多边压缩；BERT4ETH 把账户表示学习转成 transaction sequence pre-training，为后续链上行为模型提供了更自然的接口。

## Limitations

- 威胁模型、假设或适用范围的限制：论文把 BERT4ETH 称为 universal solution，但实验只覆盖 phishing detection 和 de-anonymization；对 Ponzi、ICO scam、wash trading、pump-and-dump、MEV bot、DeFi exploit 等是否泛化并未证明。
- 数据集、baseline、metric、ablation 或复现性的不足：phishing 标签依赖 Etherscan，可能存在标注偏差和漏标；Tornado/ENS ground-truth pair 数量不大，Tornado 仅 182 pairs；F1 报告使用固定阈值 0.3 且每组取 5 次实验最佳 F1，这会让结果偏乐观。
- 在真实 DeFi 场景中可能失效的条件：如果攻击行为通过合约内部状态、跨链桥、中心化交易所、混币器或链下协调隐藏，单链 external transaction sequence 表示可能不足。对交易历史很短的新地址，过滤规则和序列模型都容易失效。
- 工程与部署成本：需要 archive node、Ethereum-ETL、大规模预训练和地址词表/embedding 管理；地址空间极大，负采样策略是必要工程补丁，但也意味着模型训练分布会强烈影响下游表示。

## My Takeaways

- 对 DeFi / 区块链安全研究的启发：BERT4ETH 提醒我们，链上安全检测里的“行为”不一定要先还原成图，也可以保留原始时序和交易特征；对交易意图识别、攻击溯源、资金流聚类都有启发。
- 可复用的方法：Masked Address Prediction 可作为账户行为预训练任务；frequency-aware negative sampling 可复用于任何长尾地址/合约/token 建模；in/out separation 可复用于 phishing、资金归集和洗钱流向分析。
- 可能的后续问题：把 BERT4ETH 表示接入 DeFi semantic actions、transaction intent、source-code semantics 或 graph neural representations；用时间切分评估模型对新 fraud campaign 的泛化；研究短历史地址和新地址冷启动。

## Related Papers

- 前置阅读：BERT；Transformer；DeepWalk；Trans2Vec；Dif2Vec；Role2Vec；GraphSAGE；GAT。
- 后续阅读：Hunting in the Dark Forest: A Pre-trained Model for On-chain Attack Transaction Detection in Web3；TxSum: User-Centered Ethereum Transaction Understanding with Micro-Level Semantic Grounding。
- 可对比论文：Trans2Vec 专门做 phishing scam detection；Beres et al. 做 Ethereum profiling/deanonymization；HGATE/TTAGNN 做图神经网络 phishing detection。
- 最接近的相关工作：Trans2Vec、Dif2Vec、Role2Vec、GraphSAGE，以及 Beres et al. 的 Ethereum deanonymization benchmark。
- 关键差异：BERT4ETH 不再以账户图随机游走或多跳聚合为核心，而是直接建模单个 EOA 的交易序列，并通过 MAP 预训练得到可迁移账户表示。

## Open Questions

- 预训练账户表示能否迁移到 DeFi 攻击检测、MEV 行为识别、洗钱路径聚类等更复杂任务？
- 如果按时间严格切分训练/测试，BERT4ETH 对 2022 以后新型 fraud campaign 的泛化会不会显著下降？
- 地址 embedding 如何处理新地址、合约升级、跨链映射和中心化交易所批量热钱包？
- 能否把 transaction sequence encoder 和 graph encoder 合并，让模型同时保留顺序、局部结构和跨账户传播路径？
