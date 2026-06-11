# 【TDSC‘2026】Unveiling Ethereum Mixing Services Using Enhanced Graph Structure Learning

## Metadata

- Title: Unveiling Ethereum Mixing Services Using Enhanced Graph Structure Learning
- Authors: Yan Wu, Cong Wu, Yebo Feng, Lin Li, Xinyue Zhang, Zhen Li, Jiahang Sun, Zijian Zhang, Jincheng An, Yong Liu, Zhitao Guan, Liehuang Zhu
- Venue / Year: IEEE Transactions on Dependable and Secure Computing, Vol. 23, No. 2, 2026
- Note Name: 【TDSC‘2026】lasc
- Paper: https://doi.org/10.1109/TDSC.2025.3640941
- IEEE Xplore: https://ieeexplore.ieee.org/document/11283073/
- Local PDF: `/Users/wenkuanxiao/Zotero/storage/BEFLKIHT/Wu 等 - 2026 - Unveiling Ethereum Mixing Services Using Enhanced Graph Structure Learning.pdf`
- Code: TBD. PDF and quick source search did not expose an official code repository.
- Dataset / Artifact: TBD. The paper claims a public / robust Tornado Cash deanonymization ground-truth dataset, but I did not find an artifact URL in the PDF or official result pages.
- Scope / Subfield: 混币服务去匿名化 / 图方法地址关联
- Tags: Ethereum, Tornado Cash, Mixing Service, Deanonymization, Address Correlation, Graph Representation Learning, Link Prediction, ENS, GCN, BST, DRNL
- Status: DONE

## TL;DR

这篇论文研究 Tornado Cash 这类 smart contract-based mixing service 的 deposit / withdrawal 地址关联问题。作者先形式化定义 SC-CMS，再提出 LASC：通过六类 Tornado Cash 使用模式恢复真实转账路径，构造 mixing transfer graph，把混币账户关联转成图上的 link prediction。实验证明 LASC 在真实 Ethereum/Tornado Cash 数据上比启发式、BERT4ETH 和 Du et al. 的图方法有更高 F1，核心收益来自更细的路径恢复、更多 ENS 标签和同时利用邻域结构与 pairwise structure。

## 毒舌评论

这篇论文真正有价值的是工程化数据清洗和图建模，不是它那个“理论证明”。Theorem 1 基本把“训练好的模型会让真链接相似度高于假链接”作为前提，然后推出比随机猜好；这更像合理性论证，不是密码学意义上的 break。它比前作强，但强在把 Tornado Cash 演化后的 router/proxy/relayer 噪声处理得更仔细，以及把 ground truth 从 103 对扩到 949 对；若 ENS 标签假设不稳，或者目标用户刻意不留下这些行为偏差，模型的漂亮 F1 会立刻缩水。

## Research Question

- 研究对象：Ethereum 上 Tornado Cash 的混币账户，尤其是 deposit account 与 withdrawal account 是否属于同一用户实体。
- 小领域范围：混币服务去匿名化、链上地址关联、图表示学习。
- 具体问题：在 Tornado Cash 使用 relayer、old proxy、proxy、router 等辅助合约后，如何从公开链上交易中恢复真实混币路径，并预测 deposit / withdrawal 账户对是否相关。
- 为什么重要：Tornado Cash 的 zk-SNARK 和 note 机制切断了 deposit 与 withdrawal 的直接链上链接，但洗钱追踪和取证仍需要在行为层面恢复可能关联。既有启发式覆盖窄，神经网络方法又受真实标签稀缺和合约演化影响。
- 论文边界：不破解 zk-SNARK、不恢复现实身份、不证明单个预测可作为法律证据；只输出混币账户对的概率性 linkability。

## Motivation and Basic Idea

- Motivation：已有 Tornado Cash deanonymization 方法有三个痛点。第一，缺少统一的 SC-CMS 定义和安全目标，导致讨论混乱。第二，Tornado Cash 的实现从简单 direct deposit/withdraw 演化到 old proxy、proxy、router、relayer 组合，旧特征会被噪声稀释。第三，可靠 ground truth 太少，Du et al. 只约 103 对，模型泛化受限。
- Basic idea：先把 Tornado Cash 的真实交互拆成可解释的六类 usage patterns，恢复真正的 deposit / withdrawal account 和转账路径；再把混币交易、非混币邻居交易、时间/金额/频次/pattern 特征压进 MTG；最后用 DRNL+GCN 学目标节点局部结构，用 BST 学 deposit-withdrawal pairwise structure，再由 MLP 做 link prediction。
- 这个 idea 如何回应 motivation：路径恢复解决 Tornado Cash 演化带来的合约噪声；ENS controller/renewal 规则扩充标签；GNN 和 BST 分别处理“目标节点周围长什么样”和“这一对节点之间是什么关系”。
- 作者给出的证据：真实 mixing transaction dataset 有 272,236 records、83,089 accounts；ground truth 扩到 949 mixing account pairs；LASC 的 Precision/Recall/F1 为 0.905/0.841/0.871，高于 Hu et al. 的 0.695/0.512/0.590 和 Du et al. 的 0.832/0.768/0.798。
- 我的判断：论文的主线成立，且确实把前作 MixBroker 类方法推进了一步。但它仍是行为统计攻击，不是协议层匿名性被数学击穿；实验结论主要适用于 2019-12-15 到 2022-08-08 这段 Tornado Cash ETH pool 历史。

## Background

- 背景：Tornado Cash 用户先向固定金额 pool deposit，生成 note；withdraw 时用 note 生成零知识证明和 nullifier，在不暴露原 deposit 的情况下把同额资金转给目标地址。匿名性依赖足够大的 anonymity set，以及用户不在外围操作中泄露关联。
- 问题：链上交易公开，攻击者可以看到外部交易、内部交易、合约调用、时间、金额、gas、邻居交互等行为信号。router、proxy 和 relayer 提高了隐私，也增加了 deanonymization 的路径恢复难度。
- Gap：早期启发式主要抓地址复用、unique gas price、时间近邻或简单交易链接；后续 BERT4ETH 和 MixBroker 引入学习方法，但对 Tornado Cash 最新辅助合约模式、pairwise structure 和标签稀缺处理仍不足。

## Threat Model / Assumptions

- 攻击者或参与者能力：攻击者可访问 Ethereum 全量公开交易、Tornado Cash 核心合约地址、ENS 交易，并掌握少量可靠账户关联标签；不需要控制合约、窃取 note 或破解 zk-SNARK。
- 链上 / 链下假设：Ethereum full node + Ethereum-ETL 能抽取外部/内部交易；Tornado Cash 核心合约集合完整；ENS owner/controller/renewal 行为可作为同一实体或强关联账户标签。
- 数据范围假设：实验覆盖 2019-12-15 到 2022-08-08，即 Tornado Cash 在制裁前正常运行的 ETH mixing 交易。
- 不覆盖的情况：Tornado Cash 制裁后的新使用模式、多链部署、USDC/USDT 等非 ETH 池、Typhoon Cash/Typhoon Network 等其他 SC-CMS、现实身份确认、法律证据链。

## Method

- 核心思路：LASC = mixing data preparation + mixing transfer graph construction + mixing account correlation。
- 核心思路是怎么想到的：Tornado Cash 的密码学链接被隐藏，但用户和辅助合约仍会在链上留下行为结构。若先把合约噪声还原成真实用户路径，再让图模型学习“同一用户实体”的局部交易结构，link prediction 就可能优于手写规则。
- 从 motivation 到 method 的逻辑链：SC-CMS 缺少统一抽象 -> 定义 SerInit/Deposit/Withdrawal/UnlinkJudge 和安全游戏 -> Tornado Cash 实现复杂 -> 抽取六类 usage patterns 恢复路径 -> ground truth 稀缺 -> 用 ENS owner/controller/renewal 扩充标签 -> 单节点 GNN 不够 -> 加 DRNL、GCN、BST 联合编码邻域和 pairwise structure。
- 关键设计取舍：作者没有直接在原始 MDG 上训练，因为 mixing contract 和 neighbor accounts 会形成异质 supernode 噪声；他们把 mixing contract 信息合并到 mixing user features，再构造只保留 mixing users 与相关 neighbor 的 MTG。
- 为什么不是更直接 / 更简单的方案：启发式精度高但覆盖窄，例如 Beres-Tx 精度 0.908 但召回只有 0.063；单纯 GNN 能覆盖更多，但容易忽略目标 deposit-withdrawal pair 的相对结构。LASC 牺牲解释性，换取更高召回和 F1。
- 系统流程或算法步骤：
  - Raw data collection：从 Ethereum full node 抽取交易，用 Ethereum-ETL 解析；匹配 Tornado Cash mixing contracts、old proxy、proxy、router 和 ENS core contracts。
  - Transfer path restoration：抽取六类 Tornado Cash 使用模式。`Pa` direct deposit；`Pb` router-based deposit；`Pc` direct withdrawal；`Pd` relayer-based withdrawal；`Pe` router-based withdrawal；`Pf` relayer & router-based withdrawal。
  - Label account extraction：复用 ENS setOwner/setSubOwner 标签，并新增 controller-based account correlation 和 renewal-based account correlation 两条规则。
  - Ground-truth dataset construction：把真实 mixing transactions 与 ENS labeled account pairs 求交，得到 Tornado Cash deanonymization 的 labeled pairs。
  - MTG construction：先构造 MDG，其中节点包括 mixing contracts、mixing users、neighbors，边包括 contract-user、user-user、user-neighbor 和 deposit-withdrawal label edges；再移除/合并 contract supernodes，抽取 pattern/quantity/time/amount 等账户特征。
  - Feature processing：原始 100 维特征，经标准化、Pearson correlation、VIF 去冗余，再用 PCA 得到 32 维特征。
  - Subgraph sampling：对目标 deposit/withdrawal 节点对采样 first-hop enclosing subgraph。
  - Information encoding：DRNL 给目标节点对周围节点打结构标签；GCN 编码邻域结构；FFN 编码节点特征；BST 用 CN、Adamic-Adar、Jaccard、shortest path distance、shortest path number 等 pairwise structure 增强注意力。
  - Link decoding：拼接 GCN similarity embedding 和 BST relation embedding，用 MLP + sigmoid 输出链接概率。
- 关键定义 / 公式 / 不变量：
  - SC-CMS syntax：`SerInit` 初始化 mixing service contract；`Deposit` 输出 deposit transaction 和 private note；`Withdrawal` 用 note 生成 withdrawal transaction；`UnlinkJudge` 判断 deposit/withdrawal pair 是否 linked。
  - Security properties：unforgeability 保护 note 不能伪造；unlinkability 让外部观察者难以关联 deposit 与 withdrawal。
  - Theorem 1：作者把 LASC 成功率写成真链接相似度分布 `D*` 相对非链接分布 `D` 的 order statistics 问题，声称只要训练模型让真链接相似度随机占优，成功率就高于随机猜测 `1/|V_D|`。
  - DRNL label：节点标签由它到两个目标节点的 shortest path distance 共同决定，用于让 GNN 感知节点相对目标 pair 的位置。
  - BST pairwise features：CN、AA、Jaccard、SPD、SPN 被映射成 query/key structure vectors，参与 binary structure attention。
- 实现细节：Python 3.8、PyTorch 1.7；三层 GCN，hidden dimension 128；BST 8 attention heads；batch size 32，epoch 50；10-fold cross validation，train/test = 9:1。

## Evaluation

- 实验思路：依次验证账户特征维度、与相关工作的性能对比、训练/测试效率和 ablation。
- 评估指标：Precision (M1)、Recall (M2)、F1-score (M3)，另报告 training time 和 testing time。
- 主要结果：
  - 数据规模：约 817,500 raw Tornado Cash transactions，包含 299,701 external transactions 和 517,733 internal transactions。正文称 43,218 deposit accounts、4,151 withdrawal accounts；Table IV total row 对 deposit unique account 的呈现与正文略不一致，复核时应以原始数据为准。数据准备后得到 272,236 real mixing transaction records、83,089 accounts、949 ground-truth account pairs。
  - Feature dimensions：32-dim 特征最佳，M1/M2/M3 = 0.905/0.841/0.871；42-dim 为 0.772/0.689/0.728；51-dim 为 0.791/0.683/0.733；100-dim 为 0.886/0.813/0.842。
  - Baselines：Beres-Gas 为 0.894/0.096/0.173；Beres-Tx 为 0.908/0.063/0.118；Tang 为 0.699/0.008/0.016；Hu et al. 为 0.695/0.512/0.590；Du et al. 为 0.832/0.768/0.798；LASC 为 0.905/0.841/0.871。
  - Efficiency：启发式里 Beres-Gas 和 Tang 推理很快，但覆盖差；Beres-Tx testing time 高达 597,642s。神经模型中 LASC training/testing time 为 21,142s/5,428s，快于 Hu 的 35,917s/7,518s 和 Du 的 25,608s/6,976s。
  - Ablation：去掉 pattern 与 non-mixing transaction features 后降到 0.791/0.683/0.733；去掉 pairwise structure 后 0.827/0.779/0.802；去掉 neighbor structure 后 0.867/0.743/0.799。说明特征、邻域结构、pairwise structure 都有贡献。

## Key Artifacts

- 关键图：
  - Fig. 1：Tornado Cash balance / transaction volume over time，用来说明服务规模和洗钱问题背景。
  - Fig. 2：Tornado Cash 用户交互流程，区分 direct path、relayer、old proxy/proxy/router。
  - Fig. 3：LASC 三阶段框架图，是方法主线。
  - Fig. 4：六类 Tornado Cash usage patterns，是路径恢复算法的核心。
  - Fig. 5：pattern features 与其他 features 的 Pearson correlation heatmap，支撑特征筛选。
  - Figs. 6/7：不同 pattern 和金额池在时间上的交易量变化，支撑 discussion 中关于 2021 router/proxy 变化和 2022 交易量下降的观察。
- 关键表：
  - Table I：Tornado Cash deanonymization prior work 对比，突出 LASC 覆盖 formal notion、six patterns、ENS 标签扩展和 graph learning。
  - Table II：符号表。
  - Table III：100 维原始账户特征，包括 pattern、quantity、time、amount 四类。
  - Table IV：Tornado Cash raw data collection，列出 0.1/1/10/100 ETH mixers、old proxy、proxy、router 的交易和账户统计。
  - Tables V/VI/VII/VIII：特征维度、baseline、效率和 ablation，是实验结论的核心证据。
  - Table IX：四个 ETH denominations 下六类 patterns 的交易和账户统计。
- 关键公式 / 定义 / 算法：
  - Definitions 1-13：Ethereum external/internal/full transaction、SC-CMS security、Tornado Cash deposit/withdrawal transaction、MDG、MTG。
  - Algorithms 1-4：抽取 direct/router deposit 与 direct/relayer/router withdrawal patterns。
  - Equations (1)-(6)：LASC 相对随机猜测的理论成功率论证。
  - Equation (7)：DRNL node labeling。
  - Equations (9)-(16)：BST 的结构特征和 binary structure attention。
  - Equations (17)-(19)：GCN/BST embedding 拼接后经 MLP 解码链接概率。
- 这些证据分别支撑哪些结论：Fig. 4 和 Algorithms 1-4 支撑“能处理 Tornado Cash 演化后的路径恢复”；Tables V/VIII 支撑特征和结构模块有效；Table VI 支撑整体优于 baseline；Table VII 支撑相对神经模型更高效；Figs. 6/7 和 Table IX 支撑 pattern 演化观察。

## Findings

- 发现 1：Tornado Cash 早期以 pattern a direct deposit 和 pattern d relayer-based withdrawal 为主，后续 proxy/router 发布后，pattern b/e/f 成为常见选择，deposit-withdrawal 对应关系更复杂。
- 发现 2：router/proxy 并不是简单噪声，而是改变 deanonymization 特征分布的机制。忽略它们会把真实用户账户、合约账户和 relayer 账户混在一起。
- 发现 3：启发式仍然有用，但只能抓“弱隐私意识用户”的明显行为。Beres-Tx 精度最高，召回极低，说明规则像 scalpel，不像覆盖性检测器。
- 发现 4：ENS 是标签扩充的关键，但也是偏差来源。owner/controller/renewal 行为确实常表示同一实体控制，但也可能包含代理管理、服务商代操作或组织账户。
- 发现 5：模型的提升来自组合拳：路径恢复 + pattern/non-mixing features + MTG + DRNL-GCN + BST。单看一个模块都不是不可替代的银弹。

## Strengths

- 论文最有说服力的地方：它没有只在旧 Tornado Cash direct deposit/withdraw 语义上做模型，而是认真处理了 old proxy、proxy、router、relayer 引入的路径变化。
- 方法、数据、实验或问题设定的优势：数据准备过程具体；ground truth 从 103 对扩到 949 对；baseline 覆盖启发式、BERT4ETH 和 Du et al. 的图方法；ablation 能解释各模块收益。
- 相比已有工作的有效推进：相比 BERT4ETH，LASC 更面向 Tornado Cash 场景和 pairwise link prediction；相比 Du et al. 的 MixBroker，LASC 增加六类 usage pattern restoration、非混币邻居特征、DRNL-GCN 与 BST 的双结构编码。
- 工程可复用性：六类 pattern extraction 和 MDG->MTG 的处理方式可以迁移到其他 smart-contract mixer 或 privacy pool 的路径恢复任务。

## Limitations

- 威胁模型、假设或适用范围的限制：这不是密码学破坏。若用户操作足够标准化、避免可学习行为偏差，或者迁移到不同链/不同 mixer，模型优势不一定保持。
- 理论论证偏弱：Theorem 1 的关键前提是训练后真链接相似度分布随机占优于非链接分布；这正是模型要证明的效果。它能解释为什么图学习可能超过随机猜测，但不是严格证明 LASC 必然 break unlinkability。
- 数据集、baseline、metric、ablation 或复现性的不足：没有找到官方代码或数据集链接，尽管论文声称发布 public dataset。复现实验需要全量 Ethereum 数据、ENS 解析、Tornado Cash 合约集合和 pattern extraction 实现，成本较高。
- Ground truth 偏差：ENS controller/renewal heuristics 增加样本，但也可能把非同一自然人账户标成同一实体。模型训练和评估都依赖这类标签，可能高估真实 forensic 精度。
- 时间与协议范围：实验截止 2022-08-08，主要覆盖 Tornado Cash ETH pools，不能直接代表制裁后行为、其他链、ERC-20 pool 或 Typhoon Cash/Typhoon Network。
- 取证解释性：作者承认 GCN component 降低 interpretability。对于洗钱调查，false positives 可能有严重后果；仅给概率分数不够，需要可解释证据链。
- 负样本与部署阈值：论文主要报告 Precision/Recall/F1，但现实部署需要控制不同阈值下的 false positive risk，以及每个 withdrawal 的候选排序质量。
- 指标口径小问题：正文与 Table IV 中 deposit account 统计存在轻微不一致，说明数据口径需要复核。

## My Takeaways

- 对 DeFi / 区块链安全研究的启发：混币服务匿名性研究不能只看密码学协议，还要看合约调用路径、辅助合约演化、relayer 机制、用户邻居交易和链下标签来源。
- 可复用的方法：复杂协议的 deanonymization 可以先做 usage pattern restoration，再把路径恢复结果变成图特征。这个套路也适用于跨链桥、隐私钱包、AA paymaster 和多合约 laundering flow。
- 可能的后续问题：能否把 LASC 的预测解释成可审计证据，例如给出导致高 link probability 的 pattern、共同邻居、时间/金额证据？能否用 GuideEnricher 类仿真生成对抗性混币策略，专门降低 LASC 的可学习信号？能否把 wallet fingerprint、gas behavior、LASC 图特征做多信号融合？

## Related Papers

- 前置阅读：Blockchain is Watching You: Profiling and Deanonymizing Ethereum Users；Analysis of Address Linkability in Tornado Cash on Ethereum；BERT4ETH: A Pre-trained Transformer for Ethereum Fraud Detection。
- 后续阅读：[【USENIX‘2024】GuideEnricher](notes/defi/【USENIX‘2024】guideenricher.md)，它从匿名性 guidebook 和 DRL 角度主动发现 mixer 操作风险；[【SAC‘2025】Attacking Anonymity Set in Tornado Cash via Wallet Fingerprints](notes/defi/【SAC‘2025】wallet-fingerprints-tornado-cash.md)，它提供钱包 gas fee 侧信道。
- 可对比论文：[【WWW‘2023】BERT4ETH](notes/defi/【WWW‘2023】bert4eth.md)；Breaking the Anonymity of Ethereum Mixing Services Using Graph Feature Learning；On How Zero-Knowledge Proof Blockchain Mixers Improve, and Worsen User Privacy。
- 最接近的相关工作：Du et al., Breaking the Anonymity of Ethereum Mixing Services Using Graph Feature Learning，是最直接的图方法前作。
- 关键差异：Du et al. 更像把 Tornado Cash address association 转成 GNN link prediction；LASC 把前处理做得更细，显式建模六类 usage patterns，并引入 BST 处理目标节点对结构。

## Open Questions

- 949 对 ENS-derived ground truth 中，controller/renewal 标签的误标率是多少？
- 如果只用 post-router 时期数据训练和测试，LASC 的 F1 会不会明显下降？
- 每个 withdrawal 的 top-k candidate ranking 效果如何？相比二分类 F1，这更接近实际调查工作流。
- 论文声称 public dataset，但官方 artifact URL 不明显。数据是否真的公开，还是只在作者内部实验中可用？
- LASC 与 wallet fingerprint、gas price heuristics、GuideEnricher 发现的 guidebook patterns 组合后，能否形成更强但仍可解释的 hybrid linker？
- 攻击者如果知道 LASC 的特征和模型，是否可以设计 transaction padding、relayer selection 或 neighbor-noise 注入来降低链接概率？
