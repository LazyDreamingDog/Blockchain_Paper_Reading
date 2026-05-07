# 【NDSS‘2026】HOUSTON: Real-Time Anomaly Detection of Attacks against Ethereum DeFi Protocols

## Metadata

- Title: HOUSTON: Real-Time Anomaly Detection of Attacks against Ethereum DeFi Protocols
- Authors: Dongyu Meng, Fabio Gritti, Robert McLaughlin, Nicola Ruaro, Ilya Grishchenko, Christopher Kruegel, Giovanni Vigna
- Venue / Year: Network and Distributed System Security (NDSS) Symposium, 2026
- Note Name: 【NDSS‘2026】houston
- Paper: https://doi.org/10.14722/ndss.2026.241534
- NDSS Page: https://www.ndss-symposium.org/ndss-paper/houston-real-time-anomaly-detection-of-attacks-against-ethereum-defi-protocols/
- Local PDF: `/Users/wenkuanxiao/Zotero/storage/Q9W8RHTB/Meng 等 - HOUSTON Real-Time Anomaly Detection of Attacks against Ethereum DeFi Protocols.pdf`
- Slides: https://www.ndss-symposium.org/wp-content/uploads/f1534-meng-slides.pdf
- Code: Unknown. 论文、NDSS 页面和作者主页未给出公开代码仓库。
- Dataset / Artifact: 论文描述了 115 个 Ethereum DeFi incident benchmark，并报告每个攻击的受害合约、区块、日期和首个攻击交易哈希；未在论文、NDSS 页面或作者主页发现公开下载链接。
- Scope / Subfield: 攻击检测
- Tags: DeFi, Ethereum, Smart Contract, Attack Detection, Anomaly Detection, Runtime Monitoring, Likely Invariants, Control Flow, Mempool, Incident Response
- Status: DONE

## TL;DR

HOUSTON 做的是 Ethereum DeFi 协议攻击的实时异常检测：不预设某类漏洞 pattern，而是为每个协议在线学习“正常行为”，再用两类可解释信号判断新交易是否异常。第一类是 Interaction Model，记录关键入口调用序列；第二类是 Invariant Model，从函数参数和 storage 写入中挖 likely invariants。论文在 115 个历史 DeFi incident 上检测到 109 个攻击，TPR 94.8%，FP rate 0.16%，并在 20 个协议的 live monitoring 中维持约 0.08% 的告警率。

## 毒舌评论

HOUSTON 最值得肯定的是把“通用 DeFi 攻击检测”落到了可解释、可部署、可增量更新的工程系统上；但它所谓 generic 的上限很大程度来自“过去正常行为足够丰富 + 合约源码/ABI/storage layout 可用 + 攻击会留下调用或 storage 写入异常”这组条件。它不是能理解任意业务逻辑的神奇检测器，而是一个信号选得很聪明的行为白名单系统：成熟协议上很香，冷启动、私有交易、二进制合约、storage read 型漏洞和可投毒场景里就没那么神。

## Research Question

- 研究对象：Ethereum DeFi protocol 中针对智能合约代码缺陷或协议逻辑缺陷的攻击交易。
- 小领域范围：链上 / mempool 交易级实时攻击检测，重点是 anomaly detection 和可解释告警。
- 具体问题：如何在不依赖已知漏洞 pattern 的情况下，自动为任意 DeFi 协议建立正常行为模型，并在攻击交易进入 mempool 或上链后快速识别异常。
- 为什么重要：DeFi 攻击常常不是单个合约里的典型 reentrancy / overflow，而是协议业务逻辑、跨合约交互和状态更新组合出的异常路径；传统漏洞扫描和事后分析无法承担实时防护职责。
- 论文边界：只覆盖攻击者利用合约代码缺陷或协议逻辑缺陷触发非预期行为的交易；不覆盖 rug pull、私钥泄露、治理接管、MEV front-running / sandwich 等不以合约 defect 为核心的威胁。

## Motivation and Basic Idea

- Motivation：现有 DeFi 攻击检测两头都不理想。Pattern-based 方法精度高但只能抓已知攻击模式；神经网络式 anomaly detection 更通用但可解释性弱、误报多。DeFi 协议管理员真正需要的是能在攻击交易刚出现时给出可解释告警，并且能随协议行为演化持续更新的系统。
- Basic idea：把每个 DeFi 协议当成一个有历史行为分布的程序系统。正常用户通常通过有限的入口函数序列修改状态，并维持一些稳定的参数 / storage 关系；攻击交易为了触发 bug，往往要走没见过的关键调用路径，或破坏这些历史关系。
- 这个 idea 如何回应 motivation：HOUSTON 不去总结“价格操纵 / 重入 / 访问控制”这些攻击模式，而是学习协议自身的正常控制流和数据状态约束。这样一来，新型漏洞只要表现为协议行为异常，就可能被发现；告警还能指出具体异常 fingerprint 或被违反的不变量，方便 triage。
- 作者给出的证据：历史实验覆盖 115 个 DeFi incident，live 实验覆盖 20 个协议的 mempool 与已确认区块，比较对象包括 APE、BlockGPT、TXSPECTOR、DeFiRanger 和 transaction length baseline。
- 我的判断：论文的基本思想并不复杂，本质上是“协议局部正常性建模”；真正的贡献在于选信号：只看 incoming critical calls，避免全 trace 噪声；只挖少量简单 likely invariants，避免不可解释和组合爆炸。

## Background

- 背景：DeFi 协议由多个智能合约组成，借贷、DEX、oracle、vault、risk manager 等组件在同一笔交易中相互调用。攻击者可以直接调用链上合约，而不受 Web2 前端限制。
- 问题：很多攻击利用的是协议开发者隐含假设没有被合约强制执行。例如论文中的 TicketMonster 例子：`setVipTicket` 错误公开导致攻击者低价买普通票后免费升级；`buy(token, isvip)` 未校验 token 地址导致攻击者用恶意 ERC20 伪造支付。
- Gap：现有工具要么静态检查已知漏洞类型，要么用交易 trace pattern 检测已知攻击，要么用黑盒 embedding 做异常检测。它们缺少同时满足实时、通用、低误报、可解释这四点的方案。

## Threat Model / Assumptions

- 攻击者或参与者能力：攻击者可提交普通交易、构造合约、直接调用协议公开接口、利用代码缺陷或协议逻辑缺陷造成资金损失或状态破坏。
- 链上 / 链下假设：HOUSTON 是 off-chain monitoring solution；可分析 pending mempool 交易，也可分析已上链交易。若交易在 mempool 可见，系统可以本地模拟执行并在交易确认前告警。
- 合约元数据假设：系统理想情况下由协议管理员部署，因此可获得协议合约地址、ABI、源码和 storage layout。实验中由于没有管理员输入，作者用受害合约 deployer 近似恢复协议合约集合。
- 市场、流动性、排序、预言机或网络假设：系统本身不依赖特定 AMM / oracle / price formula，但预防能力依赖 mempool 可见性；私有 relay 交易只能上链后检测，无法提前拦截。
- 不覆盖的情况：rug pull、私钥泄露、治理接管、MEV 活动；二进制合约只能用于 Interaction Model，无法可靠挖 storage invariant；source / ABI / storage layout 缺失会降低精度。

## Method

- 核心思路：HOUSTON = Contract Processor + Transaction Processor + Protocol Interaction Analysis + Invariant Analysis + Anomaly Detection + Continuous Update。
- 核心思路是怎么想到的：攻击交易要么以异常方式调用协议关键入口，要么让协议 storage 状态出现历史上不该出现的关系。调用序列捕捉 control-flow signal，likely invariant 捕捉 data-state signal，两者互补。
- 从 motivation 到 method 的逻辑链：全 trace 太噪 -> 只保留协议外部发起的关键入口调用 -> 建立 interaction fingerprint；手写复杂安全属性太贵 -> 从历史交易动态挖简单、可解释的 likely invariants -> 新交易违反任一模型就告警；协议会演化 -> 正常交易和确认的 false positive 反哺模型。
- 关键设计取舍：论文没有追求完整语义理解，也没有训练端到端模型，而是选择轻量增量模型。代价是检测能力绑定在“历史上见过什么”和“当前 invariant classes 能表达什么”上。
- 为什么不是更直接 / 更简单的方案：直接看所有 function call 会引入大量外部 / 内部调用噪声；直接使用交易长度、profit 或高维 embedding 可解释性弱；直接写攻击规则又回到 pattern-based 的覆盖问题。
- 系统流程或算法步骤：
  - Contract Processor：给定协议合约集合，提取 ABI、public functions、argument types 和 storage layout。
  - Storage Variables Decoder (SVD)：把 SSTORE 的 slot id 映射回精细 storage 变量。它利用 Solidity storage layout 和交易 trace 中记录的 KECCAK256 preimage，处理 packed slot、mapping、array 和嵌套动态结构。
  - Transaction Processor：通过 Ethereum archive node 和自定义 tracer 记录 `SSTORE`、`CALL`、`DELEGATECALL`、`STATICCALL`、`KECCAK256`。输出 Call Report 和 Storage Report。
  - Protocol Interaction Analysis：先只保留 protocol 外部实体进入 protocol 合约的 incoming calls；再标记 critical calls，即自身或子调用会修改协议 storage，或触发 `transfer`、`transferFrom`、`safeTransferFrom`、`approve` 等关键 ERC20 操作；然后过滤直接打到协议 ERC20 token 合约的普通 ERC20 调用；最后把剩余关键调用序列作为 fingerprint。
  - Invariant Analysis：在函数边界上结合参数和 storage 变量，动态挖 likely invariants。论文最终只保留四类简单不变量：C1 整数变量 never zero；C2 address / bytes 变量等于固定值或来自有限集合；C3 两个整数变量满足 `=` / `>=` / `<=`；C4 两个 string / address / bytes 变量相等。
  - Selective Mining：对二元 invariant 先用 GPT-4.1 判断变量对是否语义可比，避免挖出 `timeStamp < reserve` 这类无意义关系。
  - Anomaly Detection：Interaction Model 若发现从未见过的 fingerprint 就报警；Invariant Model 若发现成熟 likely invariant 被违反就报警。Invariant Model 使用 waiting period：只有支持交易数至少 `N = 10` 且合约年龄至少 `T = 12h` 的 invariant 才会在违反时触发告警。
  - Continuous Update：非异常交易加入 baseline；误报经人工确认后也加入 corpus，后续不再对同类行为重复告警。Interaction Model 加新 fingerprint，Invariant Model 挖新不变量、移除被正常行为违反的不变量、提高已有不变量置信度。
- 关键定义 / 公式 / 不变量：
  - Protocol Interaction：只由协议外部触发、与协议状态变化或关键 token 操作相关的 critical incoming call 序列。
  - Likely invariant：从历史执行观测中动态推断、并不保证 soundness 的稳定关系。它不是形式化证明，而是异常检测信号。
  - C1-C4 invariant classes：never-zero、固定值 / 有限集合、整数比较、同类型地址 / 字节 / 字符串相等。
  - Final decision：两个模型任一报告异常，HOUSTON 即触发 alert。
- 实现细节：系统基于 Ethereum archive node、自定义 Erigon tracer 和自研 incremental invariant miner；binary-only contracts 不参与 invariant 挖掘，但仍可进入 Interaction Model。

## Evaluation

- 实验思路：分三组评估。第一，历史回放 115 个 Ethereum DeFi incident，按时间顺序处理协议历史交易直到首个攻击交易，衡量能否在首个攻击交易上报警，以及攻击前产生多少误报。第二，与 APE、BlockGPT、TXSPECTOR、DeFiRanger 和 transaction length baseline 比较。第三，在真实主网流量中实时监控 20 个协议，测试处理时延、资源消耗和告警率；另外把系统移植到 BSC 做泛化验证。
- 评估指标：true positive rate、false positive rate、每协议每日 false positives、因果 / 指示 / 偶然三类告警解释质量、端到端时延、吞吐和资源开销。
- 主要结果：
  - 历史检测：115 个 incident 中检测 109 个，TPR 94.8%；在 8,586,421 笔历史交易上产生 13,827 个 false positives，FP rate 0.16%，平均每协议每天 0.4 个误报。
  - 告警解释质量：在 109 个 TP 中，57 个 (52%) 是 causal detection，异常信号直接指向漏洞根因；43 个 (39%) 是 indicative detection，异常反映攻击副作用或攻击意图；9 个 (8%) 是 incidental detection，能检出但对 triage 帮助弱。
  - 典型 TP：RevestFinance 中，`fnftID` 与 `fnft_created` 在 `mint` 入口不再相等，且 `mintAddressLock` 重复调用 fingerprint 提示重入；Euler 中，`donateToReserves` 先于 `liquidate` 的新 interaction，以及 `deferLiquidityStatus` 在 `liquidate` 中变为 0，暴露攻击路径。
  - False negatives：6 个 incident 未检测。4 个需要额外 invariant 类型，例如 Nomad 需要对 storage read 值挖 invariant；AzukiDAO 部署后 4 天即被攻击，历史数据不足；Snood 的漏洞在自实现 `transferFrom` 上，而系统为了降低噪声过滤了 incoming ERC20 calls。
  - False positives：误报主要来自冷启动数据不足、spurious invariant、operation repetition / permutation。对二元 invariant 使用 GPT-4.1 语义筛选后，Invariant Model 误报从 10,579 降到 7,702，减少 27.2%，TP 不变；作者抽样 100 个 invariant，84 个被认为语义有意义。
  - 成熟协议误报：对 TVL 大于 500M USD 的热门 Ethereum DeFi 协议额外回放超过 13M 笔交易，aggregate FP rate 降到 0.08%，支持“数据越成熟误报越低”的判断。
  - 与 APE 比较：重叠时间内 20 个攻击，APE countered 4 个，HOUSTON flagged 19 个；同一时间框架内 APE FP rate 0.15%，HOUSTON 0.10%。
  - 与 BlockGPT 比较：28 个共同范围内攻击，HOUSTON 检测 27 个，BlockGPT 检测 15 个；BlockGPT 论文报告 58% 的协议 FP rate < 10%，HOUSTON 在 92.2% 的协议上 FP rate < 10%，并且 55.7% 的协议 FP rate <= 1%。
  - 与 TXSPECTOR 比较：TXSPECTOR 只检测 25/115 个攻击 (21.7%)，主要受限于 user-defined rules 和特定漏洞类型。
  - 与 DeFiRanger 比较：DeFiRanger 专注 price manipulation。HOUSTON 检测了 DeFiRanger 数据集中可比较的 6 个攻击；HOUSTON 数据集另有 23 个 price manipulation 攻击理论上可能适合 DeFiRanger，但工具未开源无法验证；其余 86 个攻击基本超出 DeFiRanger 范围。
  - Transaction length baseline：只看交易调用长度只能检测 46/115 个攻击，说明“攻击交易通常很长”不是可靠信号。
  - Live monitoring：2024-04-09 至 2024-04-28 监控 20 个协议，观察 42.6M 主网交易，过滤 50.2% 简单 ETH transfer 或普通 token 调用，最终 95.6K 笔交易涉及被监控协议。系统产生 75 个告警，占相关交易 0.08%；9/20 个协议无告警，10 个协议少于 10 个告警，ConicFinance 因新增 ConicPool 地址产生 24/901 个告警。
  - 性能：平均 trace 3ms，process 180ms，detect/update 28ms，平均 end-to-end 约 211ms；最坏组合估算监控 20 个协议需约 110GB 内存和 120 cores。实际 live setup 使用 8 个 tracer、4 个 processing workers、20 个 detection workers，内存峰值约 80GB，112 cores 上平均 CPU load 8.0。
  - Additional live：2025-10-27 至 2025-11-16 同样 20 个协议，108.5K 相关交易产生 71 个告警 (0.07%)，人工确认非攻击，外部也无 incident 报告。
  - BSC 泛化：移植到 BSC 后，2025-05 至 2025-10 的 DeFiHackLabs incident 中，排除 7 个 binary-only 和 1 个信息不足案例，剩余 12 个中检测 11 个，FP rate 0.19%。未检测的 `pdz` 实际上触发了 202 天前一笔高度相似、未公开报告的可疑交易。

## Key Artifacts

- 关键图：
  - Fig. 1 / Fig. 2：TicketMonster 的 Web2 前端和 Web3 合约示例，用两个简单 bug 展示“前端假设没有合约强制执行”为什么会成为攻击入口。
  - Fig. 3：HOUSTON 总览图，展示 Contract Processor、Transaction Processor、Protocol Interaction Analysis、Invariant Analysis、两类模型、alert 和 continuous update 的闭环。
  - Fig. 4：Protocol Interaction Analysis 示例，把复杂跨合约调用压缩成 `[borrow, pay]`，说明为什么只保留 incoming critical calls 可以降噪。
  - Fig. 5：历史实验中每协议 FP rate 的累积分布，支撑“多数协议误报低”的结论。
  - Fig. 6：交易数量与 FP rate 关系图，显示高误报协议通常历史交易数极低，支撑 cold-start 解释。
  - Fig. 7：ablation，使用全部 function calls 而非 protocol interaction 时，Interaction Model false positives 增加到 1,258,841，是主设置的 192 倍，说明 selective attention 是核心设计。
- 关键表：
  - Table I：Appendix A 中 KECCAK256 look-up table 示例，支撑 SVD 解析 nested mapping / dynamic slot 的机制。
  - Table II：live 实验执行时间统计，mean end-to-end 0.211s，99% 1.165s，max 3.085s。
  - Table III：live 实验交易负载统计，报告 filter / trace / process-detect 每秒交易数。
  - Table IV：115 个历史 incident 的 per-protocol 总表，列出总交易、生命周期、误报、两模型是否检测、HOUSTON 结果和各 baseline 结果。
- 关键公式 / 定义 / 算法：论文没有给出复杂数学公式或伪代码；核心“算法”以 pipeline 和 C1-C4 invariant classes 形式呈现。关键定义是 Protocol Interaction、likely invariant、Storage Variables Decoder 和 waiting period。
- 这些证据分别支撑哪些结论：Fig. 3 / Fig. 4 支撑系统设计；C1-C4 与 SVD 支撑可解释 invariant mining；Table IV 支撑历史 TPR / FPR；Fig. 5 / Fig. 6 / Fig. 7 支撑误报分析和设计必要性；Table II / III 支撑实时部署可行性。

## Findings

- 发现 1：DeFi 攻击检测里，“通用性”未必来自更大的模型，而可以来自更好的协议局部信号。HOUSTON 把异常定义成协议自己的关键调用序列和状态关系偏离，而不是全链统一 embedding。
- 发现 2：Interaction Model 和 Invariant Model 的互补性很强。某些攻击的异常在调用路径上很明显，另一些攻击更像状态关系被破坏；两者 OR 组合能提高覆盖，但也把两类误报都带进系统。
- 发现 3：冷启动是 anomaly detection 的根本成本。作者用 waiting period、continuous update 和 synthetic/test traces bootstrap 来缓解，但新协议、低频函数和部署初期攻击仍是硬伤。
- 发现 4：HOUSTON 的可解释性不是装饰，而是系统可运维的前提。只有能指出异常 call fingerprint 或 violated invariant，管理员才可能把误报反馈给模型，或把真攻击接给 pause / frontrun 防御系统。

## Strengths

- 论文最有说服力的地方：系统设计非常务实。它避开全 trace 建模的噪声，避开黑盒模型的不可解释，也避开手写攻击 pattern 的窄覆盖，选了两组能部署、能解释、能增量更新的信号。
- 方法、数据、实验或问题设定的优势：115 个 incident benchmark 覆盖多种协议和漏洞类型；每个 incident 以首个攻击交易为 ground truth，比“攻击序列里任意一笔命中”更严格；false positive 被按原因拆解，且有 ablation 支撑关键设计。
- 相比已有工作的有效推进：相比 DeFiRanger，HOUSTON 不只做 price manipulation；相比 TXSPECTOR，减少手写规则依赖；相比 BlockGPT，解释性和误报更好；相比 APE，HOUSTON 更像检测触发器，可与自动防御系统组合。

## Limitations

- 威胁模型、假设或适用范围的限制：不覆盖 rug pull、私钥泄露、治理攻击和 MEV；私有 relay 交易无法在 mempool 阶段预防；管理员恶意或管理员权限被盗的行为可能在历史上看起来“正常”。
- 数据集、baseline、metric、ablation 或复现性的不足：论文没有公开 HOUSTON 代码或 115 incident benchmark 下载链接；APE 和 BlockGPT 都无法直接运行，比较依赖作者提供列表或论文公开结果；历史实验把攻击前所有非 reference attack 告警都算 FP，虽是上界但仍可能混入未知攻击或有价值异常。
- 方法表达能力限制：Invariant Model 只关注 storage writes 和四类简单 invariant，无法表达复杂经济安全条件，例如 `collateralValue * collateralFactor / debtValue >= 1`；Nomad FN 显示 storage read invariant 缺失会漏检。
- 工程依赖限制：精确 SVD 依赖源码、ABI 和 storage layout；binary-only contract 无法参与 invariant mining。论文提到 LLM decompiler 可部分缓解，但那不是当前系统已完成的能力。
- 运维限制：Interaction fingerprint 对重复和排列敏感，例如多次 deposit 加一次 withdraw 会产生多个新序列；Invariant Model 可能挖到 free parameter 上的 spurious restriction。GPT-4.1 筛选有帮助，但引入额外成本和模型依赖。
- 在真实 DeFi 场景中可能失效的条件：协议刚上线、关键函数低频、攻击者有耐心投毒模型、协议大规模升级、私有交易绕过 mempool、攻击只改变读取路径而不破坏当前 C1-C4 写入关系。

## My Takeaways

- 对 DeFi / 区块链安全研究的启发：实时防御不能只靠漏洞扫描或事后 forensic；协议运行时的 normal behavior model 是一条独立研究线，特别适合与 pause、circuit breaker、white-hat frontrun 机制结合。
- 可复用的方法：Protocol Interaction 的降噪方法很值得复用：只保留外部进入协议、导致 storage write 或关键 token 操作的调用。SVD 对 packed / dynamic storage 的解析也可以作为其他 DeFi trace analysis 的基础设施。
- 可能的后续问题：如何把 invariant classes 扩展到 storage reads、跨交易状态关系和经济公式，同时不把误报打爆？如何从测试、fuzzing 或 simulation 自动生成 bootstrap traces？如何让管理员反馈 false positive 时避免被攻击者投毒？

## Related Papers

- 前置阅读：SoK: Decentralized Finance (DeFi) Attacks；Daikon: Dynamically Discovering Likely Program Invariants；Demystifying Exploitable Bugs in Smart Contracts。
- 后续阅读：Demystifying Invariant Effectiveness for Securing Smart Contracts；InvCon / invariant learning 类工作；on-chain runtime monitoring 和 emergency pause / white-hat frontrun 系统。
- 可对比论文：DeFiRanger、BlockGPT、APE、TXSPECTOR、SODA、Evil Under the Sun、EtherShield。
- 最接近的相关工作：BlockGPT 和 APE 是最接近的 generalized anomaly / defensive detection 对比对象；DeFiRanger 是 DeFi price manipulation 专门检测对象；TXSPECTOR 是规则驱动 trace 分析对象。
- 关键差异：HOUSTON 的差异不是“用了 anomaly detection”这个标签，而是 anomaly signal 的语义化和可解释性：关键调用 fingerprint + likely invariant，且能在线更新。

## Open Questions

- HOUSTON 的 115 incident benchmark 是否会公开？如果不公开，后续工作很难公平复现 94.8% / 0.16% 这组核心数字。
- 如果攻击者先用低价值交易逐步扩展 interaction fingerprint 或放宽 invariant，再发起高价值攻击，实际投毒成本有多高？
- 对高度模块化、频繁升级、proxy-heavy 的协议，deployer-based protocol boundary 在第三方监控中会带来多少噪声？
- 能否把 storage read invariant、事件日志、token balance delta、oracle price dependency 纳入同一套可解释模型？
- 告警之后的最佳 response 是 pause、front-run、rate limit 还是人工确认？不同 response 的误报成本差异很大，论文只把它作为下游系统处理。
