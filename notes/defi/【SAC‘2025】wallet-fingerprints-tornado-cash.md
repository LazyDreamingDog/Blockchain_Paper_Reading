# 【SAC‘2025】Attacking Anonymity Set in Tornado Cash via Wallet Fingerprints

## Metadata

- Title: Attacking Anonymity Set in Tornado Cash via Wallet Fingerprints
- Authors: Martina Soleti, Ankit Gangwal, Mauro Conti
- Venue / Year: The 40th ACM/SIGAPP Symposium on Applied Computing (SAC '25), 2025
- Note Name: 【SAC‘2025】wallet-fingerprints-tornado-cash
- Paper: https://doi.org/10.1145/3672608.3707896
- PDF: https://dl.acm.org/doi/pdf/10.1145/3672608.3707896?download=true
- Local PDF: `/Users/wenkuanxiao/Zotero/storage/WP6XYNI5/Soleti 等 - 2025 - Attacking Anonymity Set in Tornado Cash via Wallet Fingerprints.pdf`
- Related source: https://hdl.handle.net/20.500.12608/64782
- Code: N/A. 论文描述了 Python / JavaScript 脚本，但未发现公开代码仓库。
- Dataset / Artifact: N/A. 论文没有公开 gas fee suggestion collection 或链上匹配结果数据集。
- Scope / Subfield: 混币服务匿名性攻击 / 钱包指纹关联
- Tags: Ethereum, Tornado Cash, Mixing Service, Wallet Fingerprint, Gas Fee, EIP-1559, Privacy, Address Clustering, Measurement
- Status: DONE

## TL;DR

这篇论文研究的问题是：Tornado Cash 展示的 anonymity set 只按同池等额 deposit 计数，但如果攻击者能从交易 gas fee 参数推断用户使用的钱包软件，那么 deposit 和非 relayer withdrawal 可以先按钱包类型缩小候选集。作者分析了 MetaMask、Trust Wallet、OneKey、Rainbow、Unstoppable Wallet、ShapeShift Wallet 的 gas fee 建议逻辑，动态采集建议值，再用 `maxFeePerGas` 和 `maxPriorityFeePerGas` 与链上交易做时间邻近的精确匹配。

实验显示，在一个约 3.5 小时的 Ethereum transfer 子集中，66,248 笔交易里有 13,203 笔能匹配到六个钱包之一，约 20%。在 2024-03-01 到 2024-04-09 的 Tornado Cash ETH pool 交易里，排除 relayer withdrawal 后的 2,491 笔可覆盖交易中，91 笔 deposit 和 22 笔非 relayer withdrawal 能匹配到钱包指纹，论文据此给出同一 100 ETH pool 内 deposit/withdrawal 的潜在链接例子。

## 毒舌评论

这篇论文抓到的侧信道很值得重视：钱包的 gas fee 推荐算法确实可能把“不该公开的钱包软件信息”泄露到链上。但它把“能识别钱包类型”推进到“攻击 Tornado Cash anonymity set”的证据链还偏薄，核心验证停在 91 个 deposit、22 个 withdrawal 和一个同池例子，没有按 pool 量化 anonymity set 到底缩小了多少，也没有给出误匹配率。它更像一篇很好的攻击面发现论文，不是一篇已经把 Tornado Cash 匿名集打穿的完整 deanonymization 论文。

## Research Question

- 研究对象：Ethereum 上钱包软件生成 gas fee 建议的方式，以及这些建议是否会在链上交易参数中留下可识别的钱包指纹。
- 小领域范围：Tornado Cash 混币匿名性、钱包侧信道、链上地址关联。
- 具体问题：攻击者能否根据交易的 `maxFeePerGas` / `maxPriorityFeePerGas` 推断交易来自哪个钱包，并据此把 Tornado Cash deposit 与 withdrawal 缩小到同一钱包来源的候选集合。
- 为什么重要：Tornado Cash 的密码学机制切断 deposit 和 withdrawal 的直接链接，但匿名性还依赖用户操作、钱包、relayer、gas 参数等外围系统。若钱包默认行为本身可被识别，即使用户没有明显复用地址或设置独特 gas price，也可能泄露额外关联信息。
- 论文边界：不破解 zk-SNARK，不恢复真实身份，不证明单个 deposit 与 withdrawal 一定属于同一用户；只构造一个基于钱包软件类型的启发式候选关联。

## Motivation and Basic Idea

- Motivation：既有 Tornado Cash 分析主要利用用户在 dApp 内的错误行为，例如短时间 deposit/withdraw、地址复用、自定义 gas price、deposit/withdraw 次数模式、TORN anonymity mining 等。问题是，如果用户操作更谨慎，这些 heuristic 的覆盖会下降。论文想找一种不直接依赖 TC 内部行为错误的新信号。
- Basic idea：不同钱包为了给用户推荐 gas fee，会调用不同 API/RPC，或使用不同公式处理 EIP-1559 参数。攻击者可以持续模拟这些钱包在实时网络状态下会给出的建议值，再把链上交易的 `maxFeePerGas` 与 `maxPriorityFeePerGas` 与采集值对齐。如果精确匹配，就把该交易标记为来自某个钱包。
- 这个 idea 如何回应 motivation：在 Tornado Cash 场景中，deposit 和非 relayer withdrawal 都需要用户钱包发起 EIP-1559 交易。若二者都被推断为同一钱包软件、发生在同一个 TC pool，且 deposit 早于 withdrawal，则 withdrawal 的候选来源可以从全池 deposit 缩小到“同钱包指纹的 deposit cluster”。
- 作者给出的证据：六个 WalletConnect-compatible 开源钱包的 gas fee 逻辑互不相同；通用 Ethereum transfer 子集有约 20% 的交易参数与钱包建议精确匹配；TC 可覆盖交易中有 91 个 deposit 与 22 个非 relayer withdrawal 匹配；论文给出一个 100 ETH pool 内 deposit 和 withdrawal 同指纹的例子。
- 我的判断：基本想法成立，且比“用户设置了同一个奇怪 gas price”更隐蔽，因为默认钱包建议也会留下模式。但它本质上只能做 cluster-level narrowing，不能单独完成强链接；要真正评估匿名性损失，还需要和时间、金额、relayer、地址图等信号联合，并按每个 pool 给出候选集缩小比例。

## Background

- 背景：Ethereum 使用账户模型，交易由 EOA 发起，EIP-1559 type-2 交易包含 `maxFeePerGas` 和 `maxPriorityFeePerGas`。钱包通常会根据链上 base fee、历史区块、mempool 或第三方 gas oracle 给用户提供默认 gas fee 建议。
- Tornado Cash 流程：用户先向固定金额 pool deposit，得到 secret note；之后用 note 生成证明 withdraw。withdraw 可以由 relayer 代付 gas，也可以由用户自己连接钱包发起。只有后者会把用户钱包的 gas 参数暴露在 withdrawal 交易里。
- 既有 gap：Tornado Cash 展示的 anonymity set 等于某个 pool 中等额 deposit 数量，但已有研究说明很多 deposit/withdraw 可被启发式关联。本文的 gap 是：现有启发式多围绕 TC 使用行为，较少研究钱包软件本身作为链上指纹。

## Threat Model / Assumptions

- 攻击者或参与者能力：攻击者可以观察公开链上交易，知道交易时间、目标合约、input、`maxFeePerGas`、`maxPriorityFeePerGas` 等字段；攻击者还能持续采集多个钱包在相近时间会给出的 gas fee 建议。
- 链上 / 链下假设：被分析钱包的 gas fee 推荐逻辑可通过开源代码或 API/RPC 复现；钱包建议值在交易发起与链上确认之间仍可对应；不同钱包建议值可被近似看作一对一指纹。
- 用户行为假设：用户接受钱包默认 gas fee，不手动自定义；同一用户在 TC deposit 和 withdrawal 时使用同一种钱包软件；withdrawal 不通过 relayer，否则 gas 由 relayer 支付。
- 不覆盖的情况：自定义 gas fee、未分析的钱包、钱包算法更新、API 返回值碰撞、交易确认延迟过长、Tornado Cash 自身 gas fee 建议、relayer withdrawal。

## Method

- 核心思路：`wallet source-code analysis -> dynamic gas suggestion collection -> chain transaction extraction -> exact fee-parameter matching -> TC same-pool candidate clustering`。
- 核心思路是怎么想到的：EIP-1559 交易把用户愿意支付的 fee cap 和 tip 公开到链上，而多数用户直接接受钱包默认建议。若钱包推荐算法有差异，这些公开参数就可能成为 wallet fingerprint。
- 从 motivation 到 method 的逻辑链：TC anonymity set 被既有启发式削弱 -> 既有启发式依赖用户在 mixer 内的行为错误 -> 钱包默认 gas fee 不是 TC 内部行为，却会参与 deposit/withdrawal 发起 -> 识别钱包软件后可按钱包 cluster 缩小候选匿名集。
- 关键设计取舍：论文使用精确匹配而不是机器学习分类。这样解释性强，也能直接指出交易参数与某个钱包建议一致；代价是覆盖率低，对采集时间窗口、确认延迟和钱包算法更新非常敏感。
- 为什么不是更直接 / 更简单的方案：仅比较 gas price 是否相等已经是旧 heuristic，且更多针对自定义 gas price。本文转向“默认钱包建议值”，需要复现各钱包的推荐逻辑和实时网络状态，才能把相同或相近的公开 fee 参数解释成钱包来源。
- 钱包选择标准：软件钱包、开源、WalletConnect-compatible。论文分析六个 Ethereum 钱包：MetaMask、Trust Wallet、OneKey、Rainbow、Unstoppable Wallet、ShapeShift Wallet。
- 钱包指纹来源：
  - MetaMask：type-2 交易优先调用 `gas.api.cx.metamask.io/networks/1/suggestedGasFees` 或 `gas-api.metaswap.codefi.network/networks/1/suggestedGasFees`；fallback 使用 `eth_feeHistory`。前端提供 low / market / aggressive 三档。
  - Trust Wallet：结合 `eth_feeHistory` 与 `eth_getBlockByNumber`。`eth_feeHistory` 参数为 `[10, "latest", [5]]`，取最近 10 个区块 5th percentile priority fee 的中位数作为 `maxPriorityFee`，再用 `MaxFeePerGas = BaseFeePerGas * 1.2 + MaxPriorityFee`。
  - ShapeShift Wallet：调用 `api.ethereum.shapeshift.com/api/v1/gas/fees`，返回 slow / average / fast，每档包含 `gasPrice`、`maxFeePerGas`、`maxPriorityFeePerGas`；官方 `maxFeePerGas` 取 `max(gasPrice, maxFeePerGas)`，priority fee 取 API 返回值。论文指出它不支持用户自定义 gas fee。
  - Rainbow：调用 `metadata.p.rainbow.me/metarology/v1/gas/mainnet`，获得 `baseFeeSuggestion` 和各档 `maxPriorityFeeSuggestions`；normal / fast / urgent 的 `maxFeePerGas` 分别对 base fee 增加 0%、5%、10% 后取近似整数。
  - OneKey：依赖 Blocknative `api.blocknative.com/gasprices/blockprices`，按 99%、95%、90%、80%、70% next-block inclusion confidence 给出预测。
  - Unstoppable Wallet：依赖 `eth_feeHistory [10, "latest", [5]]`，取最近两个区块的 `baseFeePerGas` 最大值作为 `maxFeePerGas`，再用最近 10 个区块 5th percentile 的平均值作为 `maxPriorityFeePerGas`。
- Tornado Cash 自身 gas 建议：TC deposit/withdrawal 会调用 `generateTransaction` 和 `fetchGasPrice`，`maxPriorityFeePerGas` 默认设为 3，`maxFeePerGas` 由 `gas-price-oracle` 相关逻辑计算。论文还经验性观察到部分 TC 建议符合 `eth_gasPrice + 3 - 0.01`。
- 系统流程或算法步骤：
  - 用 Python 3.12.1 脚本复现六个钱包的建议逻辑。
  - 从 2024-03-01 到 2024-04-09 持续采集建议值，平均每 15 秒采集一次。
  - 每次采集记录 UTC timestamp、钱包、priority level、`maxFeePerGas`、`maxPriorityFeePerGas`。
  - 从链上抽取目标时间窗内交易，保存 hash 与 gas fee 参数。
  - 对每笔交易寻找“确认时间前几秒内”的钱包建议值；若 `maxFeePerGas` 和 `maxPriorityFeePerGas` 都相等，则记为 full match。
  - 对 TC 交易，只把 deposit 和非 relayer withdrawal 纳入钱包指纹启发式；relayer withdrawal 不纳入，因为 gas 由 relayer 支付。
  - 若同一 TC pool 中 deposit 早于 withdrawal，且两者匹配到同一钱包指纹，则视为潜在可链接。
- 关键定义 / 公式 / 不变量：
  - Anonymity set：某个 TC pool 中等额 user deposits 的数量。
  - Wallet fingerprint：由钱包 gas fee 推荐算法在链上公开交易参数中留下的可识别模式。
  - Full match：交易的 `maxFeePerGas` 与 `maxPriorityFeePerGas` 同时等于某次钱包建议，且建议采集时间早于交易确认时间几秒。
  - EIP-1559 effective gas price：`gasprice = min(BaseFeePerGas + maxPriorityFeePerGas, maxFeePerGas)`。
  - Trust Wallet 公式：`MaxFeePerGas = BaseFeePerGas * 1.2 + MaxPriorityFee`。
  - TC 经验公式：`eth_gasPrice + 3 - 0.01`。
- 实现细节：通用 Ethereum 验证使用 2024-04-09 20:02:00 到 23:28:47 的较短窗口，区块范围为 19620335 到 19621362。TC 验证使用 Tornado.Cash Router `0xd90e2f925DA726b50C4Ed8D0Fb90Ad053324F31b`，覆盖 ETH 相关 0.1 / 1 / 10 / 100 ETH pools。

## Evaluation

- 实验思路：先在通用 Ethereum transfer 交易上验证“gas fee 参数能反推出钱包”的可行性，再把同样方法用于 TC Router 交易，检查 deposit 与非 relayer withdrawal 是否能被钱包指纹标记。
- 评估指标：full match 数量、full match 比例、TC 中可覆盖交易数量、deposit/withdrawal 匹配数量、relayer withdrawal 是否如预期无法匹配。
- 主要结果：
  - 通用 Ethereum 验证：2024-04-09 约 3.5 小时窗口内抽取 66,248 笔 transfer 相关交易，其中 13,203 笔 full match，约 20%。论文用一笔 Trust Wallet 匹配示例展示链上交易参数与采集建议一致。
  - TC 交易结构：2024-03-01 到 2024-04-09，TC Router 上共有 4,579 笔相关交易，包括 2,210 deposits、2,088 withdrawals involving a relayer、281 withdrawals not involving a relayer。
  - TC 钱包指纹覆盖：排除 relayer withdrawals 后，2,491 笔可覆盖交易中有 91 个 deposits、22 个 non-relayer withdrawals full match；relayer withdrawal full match 为 0，符合预期。
  - TC 潜在链接例子：论文给出 100 ETH pool 中一个 deposit hash `0x17b0dff7f8702f0ce9f282a427d3d2b542fc91d1c7cebfc53d2df8c3f1abc4c1` 和一个 withdrawal hash `0xa4c27df983d42cc4b6dc3c309523fa134dda3560e4703b4f895a434e467976f6`，二者同 pool、deposit 早于 withdrawal、且 full match 到同一钱包指纹，因此被视为潜在可链接。
  - TC 自身建议干扰：在 2024-03-17 17:28:55 到 2024-04-09 23:28:47 的子窗口中，2,856 笔 TC 交易里有 343 笔约 12% 符合 TC 经验 gas 建议公式，其中 140 个 deposits、253 个 withdrawals；253 个 withdrawals 中 11 个是不经 relayer 的 withdrawal。

## Key Artifacts

- 关键图：
  - Figure 1：Tornado Cash deposit/withdrawal 流程，说明 relayer 与 non-relayer withdrawal 的差异，是确定启发式覆盖范围的基础。
  - Figure 2：六个 Ethereum 钱包使用的 API/RPC 列表，是 wallet fingerprint 的来源清单。
  - Figure 3：MetaMask type-2 transaction 中 try-catch gas estimation 代码片段，说明其优先 API 与 fallback 路径。
  - Figure 4 / Figure 5：MetaMask 前端 low priority 建议与 API 采集值对齐，证明作者复现钱包建议逻辑的方式可行。
  - Figure 6：ShapeShift API 返回示例，展示同一钱包对 slow / average / fast 三档同时给出 `gasPrice`、`maxFeePerGas`、`maxPriorityFeePerGas`。
  - Figure 7 / Figure 8：TC relayer withdrawal 的链上 gas price 与 `eth_gasPrice + 3 - 0.01` 经验公式匹配，是识别 TC 自身建议的证据。
  - Figure 9 / Figure 10：一笔真实 Ethereum 交易与 Trust Wallet 建议值 full match，是通用链上验证的核心例子。
  - Figure 11：Polygon 网络钱包 gas fee API/RPC 列表，支撑 future work 中跨链扩展的可行性。
- 关键表：论文没有正式 table，主要证据以 figure 中嵌入的表格或截图呈现。
- 关键公式 / 定义 / 算法：
  - `Transaction fees = gas units used * gas price`
  - `gasprice = min(BaseFeePerGas + maxPriorityFeePerGas, maxFeePerGas)`
  - `MaxFeePerGas = BaseFeePerGas * 1.2 + MaxPriorityFee`
  - `eth_gasPrice + 3 - 0.01`
  - Full match 判定规则。
- 这些证据分别支撑哪些结论：Figure 2-6 支撑“不同钱包有可复现差异”；Figure 9-10 支撑“链上交易可匹配钱包建议”；TC 交易统计和 Figure 7-8 支撑“TC 场景需要排除 relayer 和 TC 自身建议”；TC 100 ETH example 支撑“同池同钱包指纹可形成候选链接”。

## Findings

- 发现 1：钱包默认 gas fee 建议不是纯用户体验细节，而是可外泄的钱包软件元数据。交易不直接公开“来自 MetaMask/Trust Wallet”，但 fee 参数可能间接公开这个事实。
- 发现 2：这个侧信道对普通 Ethereum 交易覆盖更高，对 Tornado Cash 覆盖显著下降。原因不是启发式本身完全无效，而是 TC 有 relayer、dApp 自身建议、自定义 fee 和钱包多样性这些混杂因素。
- 发现 3：relayer 对该启发式是天然屏障。relayer withdrawal 的 gas 参数反映的是 relayer 端，而不是 withdrawer 钱包，因此论文得到 0 个 relayer withdrawal full match 是预期结果。
- 发现 4：论文攻击的是匿名集的候选范围，不是确定性身份链接。即使 deposit 和 withdrawal 共享钱包指纹，也只能说明它们属于同一钱包软件 cluster，而不是同一用户。
- 发现 5：TC 自身 gas fee 建议必须作为 confounder 处理。否则攻击者可能把 dApp 建议误认为钱包建议，造成错误的 wallet attribution。

## Strengths

- 论文最有说服力的地方：选题角度新，跳出了“用户在 mixer 里犯错”的传统启发式，把钱包软件和 gas oracle 这层基础设施纳入隐私攻击面。
- 方法、数据、实验或问题设定的优势：作者不是只观察链上 gas price，而是回到钱包源码/API/RPC 复现 fee suggestion 逻辑；验证路径也先做通用 Ethereum，再做 TC 专门场景，逻辑顺序清楚。
- 相比已有工作的有效推进：Tutela、Wang et al.、Tang et al. 更关注已发生的 Tornado Cash 用户行为模式；本文提供的是一种可与既有启发式组合的新 side channel。
- 工程可复用性：动态采集钱包建议值再和链上交易匹配的 pipeline 可以扩展到更多钱包、更多链和更多隐私 dApp。

## Limitations

- 威胁模型、假设或适用范围的限制：启发式依赖用户接受默认 gas fee、同一用户 deposit/withdrawal 使用同一钱包软件、且 withdrawal 不走 relayer。现实中这些条件不一定稳定成立。
- 数据集、baseline、metric、ablation 或复现性的不足：论文没有公开脚本和采集数据，也没有给出 full match 的 false positive rate、不同钱包之间的碰撞率、每个 TC pool 的 anonymity set 缩小比例。
- 评价粒度偏粗：TC 部分给出 91 个 deposit 和 22 个 withdrawal，但没有系统枚举最终 candidate pairs、每个 withdrawal 的候选 deposit 数、或与 existing heuristics 组合后的边际增益。
- 时间窗口较窄：通用 Ethereum 验证只用了 2024-04-09 约 3.5 小时窗口；TC 分析是一月左右窗口。钱包算法、API、网络拥堵和用户行为都会随时间变化。
- 覆盖钱包有限：只分析六个钱包。未分析钱包会造成 false negative，且如果两个钱包恰好在某些时间给出相同 fee 参数，精确匹配也可能产生 false positive。
- 交易类型表述有疑点：论文说通用验证过滤 transfer action `0xa9059cbb`，这是 ERC-20 `transfer(address,uint256)` selector，而文中又多次说采集的是 ETH transfer 的钱包建议。EIP-1559 fee cap 与 tip 可能仍共用推荐逻辑，但这里至少需要更清楚地解释交易类型一致性。
- 在真实 DeFi 场景中可能失效的条件：用户使用 relayer、隐私钱包标准化 fee、手动设置 fee、钱包引入随机化/rounding、或 dApp 自身接管 fee 参数时，这个指纹都会变弱。

## My Takeaways

- 对 DeFi / 区块链安全研究的启发：隐私协议的匿名性不仅由密码学和合约决定，还受钱包、RPC、gas oracle、relayer 等外围组件影响。Operational metadata 本身就是攻击面。
- 可复用的方法：把“公开链上字段”和“客户端默认配置/推荐算法”做差分匹配，是一种很实用的侧信道研究套路。类似方法可以看 nonce 管理、gas limit、priority fee rounding、RPC provider 特征、bundle/relay 路径等。
- 可能的后续问题：最直接的下一步不是再列更多钱包，而是给出每个 pool 的候选集缩小曲线和误匹配评估，然后与时间窗口、地址图、旧 TC heuristics 联合，看看实际匿名集能缩小到什么程度。
- 防御启发：隐私敏感钱包不应让 fee cap / tip 直接暴露唯一推荐算法。可考虑标准化、bucketization、随机扰动、与隐私 dApp 协调 fee policy，或优先使用不会暴露 withdrawer 钱包的 relayer 机制。

## Related Papers

- 前置阅读：Tutela: An Open-Source Tool for Assessing User-Privacy on Ethereum and Tornado Cash；Analysis of Address Linkability in Tornado Cash on Ethereum。
- 后续阅读：[【USENIX‘2024】GuideEnricher](notes/defi/【USENIX‘2024】guideenricher.md)，它从 guidebook / DRL 角度主动发现 mixer anonymity-compromising patterns。
- 可对比论文：On How Zero-Knowledge Proof Blockchain Mixers Improve, and Worsen User Privacy；Breaking the Anonymity of Ethereum Mixing Services Using Graph Feature Learning；Blockchain is Watching You。
- 最接近的相关工作：Tutela 和 Wang et al. 都是 Tornado Cash deanonymization / privacy degradation 方向的核心前置工作；本文关键差异是把钱包 gas fee 逻辑作为 linkability signal，而不是只看 TC 使用模式。
- 关键差异：已有工作更像从历史链上行为中总结用户错误，本文更像从钱包实现中挖默认行为侧信道。

## Open Questions

- 91 个 TC deposit 和 22 个 non-relayer withdrawal 分别来自哪些钱包？论文说 full matches come from the same wallet，但没有在正文清楚展开这个钱包的分布。
- 如果把 wallet fingerprint 与时间间隔、地址复用、linked ETH address、gas price uniqueness 等旧 heuristic 合并，能把每个 withdrawal 的候选 deposit 数缩到多少？
- full match 的误报率是多少？在多个钱包、多个 priority levels、多个 API 之间，是否存在同一时间同参数碰撞？
- 钱包更新 gas fee 逻辑后，collector 需要多快同步？攻击者在真实链上维护这种指纹库的成本是多少？
- 论文的 `0xa9059cbb` transfer 过滤与 ETH transfer fee suggestion 是否完全一致？如果目标交易类型改变，钱包是否会使用不同 fee policy？
- 对隐私钱包或 TC 前端而言，应该统一 fee policy、随机化 fee、还是默认鼓励 relayer withdrawal？这些防御分别损失多少成本和用户体验？
