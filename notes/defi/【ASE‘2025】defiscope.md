# 【ASE‘2025】Detecting Various DeFi Price Manipulations with LLM Reasoning

## Metadata

- Title: Detecting Various DeFi Price Manipulations with LLM Reasoning
- Authors: Juantao Zhong, Daoyuan Wu, Ye Liu, Maoyi Xie, Yang Liu, Yi Li, Ning Liu
- Venue / Year: 40th IEEE/ACM International Conference on Automated Software Engineering (ASE 2025)
- Note Name: 【ASE‘2025】defiscope
- Paper: https://arxiv.org/abs/2502.11521
- Local PDF: `/Users/wenkuanxiao/Zotero/storage/IX5EBBEA/Zhong 等 - 2025 - Detecting Various DeFi Price Manipulations with LLM Reasoning.pdf`
- Code: https://github.com/AIS2Lab/DeFiScope
- Dataset / Artifact: https://github.com/AIS2Lab/DeFiScope/tree/main/dataset and https://github.com/AIS2Lab/DeFiScope/blob/main/supplementary_material.md
- Scope / Subfield: 价格操纵检测
- Tags: DeFi, DEX, Oracle, Lending, Yield Farming, LLM, Fine-tuning, Attack Detection, Transaction Analysis, Source Code Analysis
- Status: DONE

## TL;DR

这篇论文提出 DeFiScope，用 LLM 从智能合约价格计算源码和交易中的 token balance changes 推断价格变化趋势，再结合 Transfer Graph 恢复出的高层 DeFi operations，匹配 8 类价格操纵攻击模式。它要补 DeFiRanger / DeFort 的短板：这些方法依赖标准价格模型或 exchange rate，对 custom / non-standard price model 不够有效，而作者统计的 95 个真实攻击中有 44.2% 涉及非标准价格模型。评估上，DeFiScope 在 95 个真实攻击 D1 上 recall 80%，高于 DeFort 52.6%、DeFiRanger 51.6%、DeFiTainter 35.8%；在 968 个 suspicious transactions 上 precision 96%，并在 96,800 个 benign transactions 上零误报。

## 毒舌评论

DeFiScope 真正聪明的地方不是“把 LLM 接进检测器”，而是把 LLM 限定在一个相对窄的任务里：读价格函数和余额变化，判断价格趋势。这样它避开了让 LLM 直接做攻击判定的不可控问题。但它仍然是规则系统，只是把最难手写的 price-model reasoning 外包给微调模型；一旦源码缺失、Slither 抽不出价格函数、攻击跨多笔交易，或者需要精确数值计算，漂亮的 80% recall 很快就露出边界。

## Research Question

- 研究对象：DeFi price manipulation attacks，尤其是标准 AMM 价格模型之外的 custom price model 场景。
- 小领域范围：LLM-assisted price change reasoning + DeFi operation recovery + rule-based attack detection。
- 具体问题：如何在不显式计算 token exchange rate 的情况下，从合约源码和交易状态变化推断 token price trend，并据此检测价格操纵攻击。
- 为什么重要：大量 DeFi 协议使用自定义价格模型，价格可能依赖中位数、EMA、total supply、多个 liquidity pools 或协议内部状态；传统基于 exchange rate 或标准 AMM 公式的方法容易漏报。
- 论文边界：需要交易相关合约源码或可识别的二币种 closed-source liquidity pool；主要分析单笔 transaction；最终攻击识别仍依赖预定义 pattern。

## Motivation and Basic Idea

- Motivation：DeFiRanger、DeFort 等 SOTA 对 CPMM / Stableswap 这类标准 price model 较自然，但 custom price model 里“价格如何随余额变化而变化”不再能靠固定公式或 exchange rate 轻松判断。UwULend 案例中，sUSDe 价格由多个池子的 instant price 和 EMA price 的 median 决定，小幅但关键的价格变化能逃过基于历史波动范围的检测。
- Basic idea：让 LLM 负责抽象 price calculation model 并判断 balance change 对 token price 的方向性影响；再把这个价格方向与交易中的 high-level DeFi operations 组合成攻击模式。
- 这个 idea 如何回应 motivation：custom model 的难点是读懂价格函数和进行趋势推理，而这正是 LLM 代码理解能力可以发挥的狭窄位置；操作语义和最终攻击匹配仍保持规则化，降低 LLM 直接判案的风险。
- 作者给出的证据：Foundry 合成 fine-tuning 数据；D1 与三种 SOTA 对比；D2/D3 展示 suspicious 和 benign 场景下的 precision / false alarm；supplementary 给出 prompt、attack patterns、false positives、TG vs CFT 对比。
- 我的判断：这是一个合理的 hybrid design。LLM 不负责全流程，而是插在“价格趋势推断”这个最难泛化的环节，工程上比纯 prompt-based audit 更可控。

## Background

- 背景：DeFi price model 可以是 CPMM、Stableswap Invariant，也可以是协议自定义公式。攻击者通过 swap、mint、burn、deposit、withdraw 等操作改变余额、reserve 或 total supply，从而影响价格计算。
- 问题：标准模型下价格方向能通过固定公式推导；custom model 下需要理解源码中的价格函数，并结合交易内余额变化判断价格涨跌。
- Gap：已有监控工具通常基于 exchange rate、历史波动范围、CFT 资金流或静态 taint，难以覆盖复杂 custom price model 和多 relayer swap。

## Threat Model / Assumptions

- 攻击者或参与者能力：攻击者可调用 DeFi 协议、使用 flash loan 或大额资金，操纵 liquidity pool、token balance、total supply 或协议价格函数依赖的状态。
- 链上 / 链下假设：系统可获取 raw transaction、internal calls、token transfer events、相关合约源码，并可通过 QuickNode 等 RPC 获取链上数据。
- 市场、流动性、排序、预言机或网络假设：主要考虑单笔 transaction 内完成的价格操纵；跨 transaction 攻击较少但存在。
- 不覆盖的情况：closed-source custom price model、源码编译失败、非 ERC20 token、跨 transaction 攻击、需要精确科学计算而非趋势判断的复杂模型。

## Method

- 核心思路：price change inference with LLM + Transfer Graph based DeFi operation recovery + price-change-directed attack patterns。
- 核心思路是怎么想到的：直接计算 exchange rate 在 custom model 中不可靠；但很多攻击只需要知道“某 token 在某 contract 中价格上升/下降”，这个趋势可由 price model 和 balance changes 推断。
- 从 motivation 到 method 的逻辑链：custom price model 难以公式化 -> 用源码切片抽取 price calculation functions -> 用微调 LLM 给价格变化 statement 打分 -> 用 Transfer Graph 恢复 Swap / Deposit / Borrow 等操作 -> 把价格变化和操作序列映射到 8 类攻击 pattern。
- 关键设计取舍：LLM 只做 price trend reasoning，不做 pattern recognition；这保留了可解释规则，但也意味着攻击类型仍被作者预定义。
- 为什么不是更直接 / 更简单的方案：只看价格波动范围会漏掉 UwULend 这种小幅但关键的操纵；只用 CFT 会漏掉复杂 multi-relayer swap；让 LLM 直接判断攻击则很难控制误报和解释。
- 系统流程或算法步骤：
  - Step 1：decode / slice raw transaction data，并取回交易中调用合约的源码。
  - Step 2/5：基于函数签名抽取可能的 price calculation function code snippet，以及 relevant accounts 的 token balance changes。
  - Step 3/4：构造 Transfer Graph，恢复高层 DeFi operations。
  - Step 6-8：把 code snippet、price-change statement、balance change description 填入 prompt，查询 fine-tuned LLM 得到价格变化趋势与置信分数。
  - Step 9：把 price change information 和 high-level operations 匹配到 8 类 attack patterns。
- 关键定义 / 公式 / 不变量：
  - Transfer：`T := <s, r, t, v>`，表示 sender `s` 向 receiver `r` 转移数量 `v` 的 token `t`。
  - Transfer Graph：`TG = (A, E)`，顶点是账户，边是带时间索引的 transfer actions；比 CFT 更细粒度，适合恢复 multi-relayer swaps。
  - CPMM：`x * y = k`，swap 后满足 `(x + Delta x) * (y - Delta y) = k`。
  - UwULend custom model：`P_sUSDe = median({IP_USDe,Pool1 ... IP_USDe,Pool5, EMAP_USDe,Pool1 ... EMAP_USDe,Pool5})`。攻击通过操纵多个 pool 的 instant price 影响 median。
  - 六类 DeFi operations：Swap, Deposit, Withdraw, Borrow, Stake, Claim。
  - 八类 attack patterns：Buy & Sell 两类，Deposit & Borrow 两类，Stake & Claim 两类，Deposit & Withdraw 两类。
- 实现细节：约 3,900 行 Python；默认使用 GPT-3.5-turbo-1106 fine-tuned model，也评估 GPT-4o-2024-08-06 和 Phi-3 LoRA SFT；支持 Ethereum 和 BSC，supplementary 展示 Polygon / Arbitrum 案例。

## Evaluation

- 实验思路：RQ1 比较 DeFiScope 与 DeFiTainter、DeFort、DeFiRanger 的检测能力；RQ2 做 fine-tuning ablation；RQ3 测真实 suspicious/benign traffic 下的 precision、false alarm 和耗时。
- 评估指标：recall、precision、false positives、average time cost、LLM inference cost；还用 McNemar Test 验证相对 baseline 的显著性。
- 主要结果：
  - D1：95 个真实 price manipulation attacks，来自 DeFort、DeFiHackLabs 和行业伙伴，覆盖 90 个 DeFi apps，总损失 $381.16M。DeFiScope 检出 76/95，recall 80%；DeFort 50，DeFiRanger 49，DeFiTainter 34。
  - D1 pattern 分布：Pattern I/II/III/IV/V/VI/VII/VIII 分别有 49/20/5/6/1/2/11/1 个案例，对应损失 $55.3M/$70M/$141.2M/$43M/$61K/$83K/$71.4M/$9K。
  - 失败案例：19 个未检出，其中 11 个理论可检但受 LLM reasoning、非 ERC20 token、single-transaction 限制影响；8 个因缺源码或 Slither 编译错误无法分析。
  - Fine-tuning：GPT-3.5 fine-tuning 后 TP 从 58 增至 76，recall 从 0.61 到 0.80；GPT-4o fine-tuning 后 TP 从 63 到 75，recall 从 0.66 到 0.79。GPT-3.5 fine-tuned 每次推理成本 $0.0107，fine-tuning cost $8；GPT-4o fine-tuned 每次 $0.0131，fine-tuning cost $25。
  - Custom price model：GPT-3.5 fine-tuning 后 custom model detection success rate 从 60% 提升到 93.3%；GPT-4o 从 86.7% 到 96.7%。
  - D2：968 个 suspicious transactions 中标记 153 个 price manipulation，147 个验证为真，precision 96%；其中 66 个已公开，81 个 previously unknown historical incidents。
  - D3：96,800 个 benign transactions 零误报。平均每笔 2.5 秒左右；supplementary 细分为 data retrieve/preprocess 0.17s、static analysis 0.97s、price change inference 1.40s，operation recovery 和 attack detection 均小于 0.01s。

## Key Artifacts

- 关键图：
  - Fig. 1：DeFiScope workflow，展示 raw transaction -> source code / balance changes -> TG operation recovery -> LLM price inference -> attack detection 的十步流程。
  - Fig. 2：fine-tuning prompt template，说明模型被训练去先抽取 price model，再对四个 price-change statements 打 1-10 分。
  - Fig. 3：UwULend Type-I prompt 和 response，展示 LLM 如何从源码中概括 median-based sUSDe price model 并判断价格下降。
  - Fig. 4：Transfer Graph 示例，展示如何从带时间顺序的资金流图恢复 Swap。
  - Fig. 5：按 Token / Yield-farming / DEX / Lending 以及 price model 分类的检测结果。
  - Fig. 6：fine-tuning 对 GPT-3.5 / GPT-4o 在 CPMM、custom、Stableswap 上的提升。
- 关键表：
  - Table I：8 个 price-change-information-directed attack patterns，是检测规则核心。
  - Table II：D1/D2/D3 数据集用途。
  - Table III：D1 中各 pattern 的案例数与损失。
  - Table IV：95 个 ground-truth attacks 上 DeFiScope 与三个 baselines 的逐案对比。
  - Table V：GPT-3.5 与 GPT-4o fine-tuning 前后的 recall、推理成本、微调成本。
  - Table VI：GPT tuning 与 Phi-3 LoRA SFT 的 recall 对比。
- 关键公式 / 定义 / 算法：
  - `T := <s, r, t, v>` 和 `TG = (A, E)` 定义 operation recovery 的输入表示。
  - CPMM / Stableswap / UwULend median model 用于说明标准与非标准价格模型差异。
  - DFS 在 TG 上找从 user-controlled accounts 出发并回到 user-controlled accounts 的 cycle，以恢复 Swap。
- 这些证据分别支撑哪些结论：prompt 和 fine-tuning 表支撑“LLM 能做 price trend reasoning”；TG vs CFT supplementary 支撑“operation recovery 比 DeFiRanger 的 CFT 更适合复杂 swap”；D1/D2/D3 支撑“检测效果和低误报具有实证基础”。

## Findings

- 发现 1：custom price model 是 DeFi price manipulation 检测的重要漏报来源，不能只依赖标准 AMM exchange rate。
- 发现 2：LLM 在这里最适合承担“从源码抽象价格模型并推断方向”的窄任务，而不是直接端到端判断攻击。
- 发现 3：Transfer Graph 对复杂 multi-relayer swap 的恢复能力明显优于 CFT；supplementary 中 TG 的 TPR 为 0.912，CFT 为 0.559，precision 分别为 0.984 和 0.991。

## Strengths

- 论文最有说服力的地方：问题定位很准，custom price model 的确是 DeFiRanger/DeFort 这类方法的薄弱点；方法把 LLM 放在一个可解释、可评估的中间环节。
- 方法、数据、实验或问题设定的优势：开源了代码、D1/D2/D3、fine-tuning 数据和 supplementary；对 baseline、ablation、成本、误报和跨链泛化都有实验。
- 相比已有工作的有效推进：相比 DeFiRanger，它不只从交易语义推断 hoard-and-dump，而是直接理解价格函数；相比 DeFort，它减少了对历史 exchange-rate range 的依赖；相比 DeFiTainter，它不是只做源码 taint。

## Limitations

- 威胁模型、假设或适用范围的限制：single-transaction 假设仍然很强；cross-transaction attacks 是主要漏报来源之一。
- 数据集、baseline、metric、ablation 或复现性的不足：DeFiRanger 未开源，作者对部分案例做了重实现；LLM prompt 每个只运行一次，虽然 temperature=0，但没有充分讨论 LLM 服务版本漂移。
- 在真实 DeFi 场景中可能失效的条件：closed-source custom price model、源码编译失败、Slither 提取失败、非 ERC20 transfer event、需要精确数值计算的复杂公式，都会削弱效果。
- 额外成本与部署风险：默认路径依赖 OpenAI fine-tuned model 和 API 成本；虽然有 Phi-3 LoRA 实验，但 recall 0.66，尚未达到 GPT fine-tuned 的 0.80。

## My Takeaways

- 对 DeFi / 区块链安全研究的启发：DeFi 检测不应只恢复“发生了什么操作”，还需要恢复“这些操作如何改变协议内部价格”。DeFiScope 把 price semantics 明确提升为一等信息。
- 可复用的方法：TG 可复用于交易解释；LLM price-trend scoring 可复用于 oracle/LP/tokenomics 风险分析；八类 patterns 可作为 price manipulation taxonomy。
- 可能的后续问题：用 PAL 或符号执行补 LLM 数值计算短板；跨 transaction attack graph；自动识别价格函数；用历史所有权聚类减少 closed-source pool 误报。

## Related Papers

- 前置阅读：DeFiRanger；DeFort；DeFiTainter；DeFiHackLabs；SoK: Decentralized Finance (DeFi) Attacks。
- 后续阅读：PriceSleuth；PMDetector；AiRacleX；Following Devils' Footprint。
- 可对比论文：DeFiRanger 用 CFT 和语义规则；DeFort 用价格监控和行为模型；DeFiTainter 做静态 source-code analysis；FlashSyn 合成攻击合约但目标更宽。
- 最接近的相关工作：DeFiRanger、DeFort、DeFiTainter。
- 关键差异：DeFiScope 把 LLM 用于 price model abstraction 和 price-change inference，并用 TG 改善 DeFi operation recovery，目标是覆盖 standard + custom price models。

## Open Questions

- LLM 对复杂价格公式的错误能否用程序生成、SMT、symbolic execution 或 PAL 可靠纠正？
- 如果攻击分散在数小时或数万区块里，如何构造跨 transaction 的 price-dependency graph？
- 能否自动定位 price calculation functions，而不是依赖签名、Slither 和源码可编译性？
- 对 closed-source custom model，Type-II prompt 只覆盖二币种 CPMM-like pool，是否会给复杂闭源协议带来系统性盲区？
