# 【USENIX‘2024】GuideEnricher: Protecting the Anonymity of Ethereum Mixing Service Users with Deep Reinforcement Learning

## Metadata

- Title: GuideEnricher: Protecting the Anonymity of Ethereum Mixing Service Users with Deep Reinforcement Learning
- Authors: Ravindu De Silva, Wenbo Guo, Nicola Ruaro, Ilya Grishchenko, Christopher Kruegel, Giovanni Vigna
- Venue / Year: 33rd USENIX Security Symposium (USENIX Security 24), 2024
- Note Name: 【USENIX‘2024】guideenricher
- Paper: https://www.usenix.org/conference/usenixsecurity24/presentation/de-silva
- PDF: https://www.usenix.org/system/files/usenixsecurity24-de-silva.pdf
- Local PDF: `/Users/wenkuanxiao/Zotero/storage/INI6PKGH/Silva 等 - GuideEnricher Protecting the Anonymity of Ethereum Mixing Service Users with Deep Reinforcement Lea.pdf`
- Slides: https://www.usenix.org/system/files/usenixsecurity24_slides-de-silva.pdf
- Code: https://github.com/ucsb-seclab/GUIDE-ENRICHER
- Dataset / Artifact: 代码仓库公开了 GuideEnricher 代码、模型、实验说明和扩展附录；论文未给出独立链上数据集下载链接。
- Scope / Subfield: 混币服务匿名性 / 隐私风险发现
- Tags: Ethereum, Mixing Service, Tornado Cash, Anonymity, Privacy, De-anonymization, Deep Reinforcement Learning, PPO, Guidebook, Detector, Simulation
- Status: DONE

## TL;DR

GuideEnricher 解决的问题是：混币服务通常给用户一份“不要这样操作”的匿名性 guidebook，但这些规则靠人工和事后链上分析总结，部署前很难发现未知的 anonymity-compromising patterns。论文把这个问题转成 DRL 探索：构造一个带 Tornado Cash、crowd users、规则 detector 的仿真环境，让 evader agent 完成转账任务，同时尽量绕过已知规则。作者用 PPO 训练 agent，再用聚类和人工检查从 evader traces 中提炼新规则；实验展示其能在 TC、TC Nova、Railgun 等服务上训练有效 agent，并发现若干 gas price / linked address 相关的新型风险模式。

## 毒舌评论

这篇论文最聪明的地方不是“DRL 很强”，而是把一个没有标签、没有 ground truth 的隐私规则发现问题，硬拗成了一个可以给 reward 的游戏。但它的发现质量最后仍然靠人来判定，且“新 pattern 会不会真实出现、出现后到底能降低多少匿名集”没有量化闭环；所以 GuideEnricher 更像是一个高产的 guidebook fuzzing 工具，不是一个已经证明能自动度量或保证混币匿名性的系统。

## Research Question

- 研究对象：Ethereum 混币服务用户在 deposit / withdraw / transfer 过程中的链上行为模式，以及这些模式如何导致交易或地址被重新关联。
- 小领域范围：混币服务匿名性保护、链上隐私风险发现、guidebook 自动扩展。
- 具体问题：能否在混币服务部署前，不依赖真实历史受害案例，主动发现 guidebook 中尚未覆盖的 anonymity-compromising patterns。
- 为什么重要：混币服务的密码学机制可以隐藏 deposit 和 withdraw 的直接链接，但用户操作方式仍可能暴露关联。现有 Tutela、Wang et al. 等工作主要从历史交易中事后总结规则，无法提前保护未来用户。
- 论文边界：不攻击 zk-SNARK 或合约实现本身，不做真实用户 deanonymization，不证明混币协议的形式化匿名性；目标是发现“用户应该避免的行为模式”。

## Motivation and Basic Idea

- Motivation：混币服务开发者通常会给用户 guidebook，例如不要复用 deposit/withdraw 地址、不要使用明显可链接的 gas price、等待足够长时间再 withdraw。但真实链上研究已经说明这些规则不完整；如果只能等用户在真实链上踩坑后再总结，就永远是 postmortem。
- Basic idea：把“发现未知匿名性破坏模式”类比成 fuzzing。先把已知 guidebook 规则实现成 detector，再训练一个 evader agent 在仿真链上完成转账任务并规避 detector。若 agent 能高成功率完成任务但仍产生可被人解释为可链接的行为，这些行为就可能是 guidebook 的候选增补规则。
- 这个 idea 如何回应 motivation：论文没有直接训练 agent 去“发现未知规则”，因为未知规则没有标签。它改为设计一个容易诱发匿名性风险的任务，让 detector 负责排除已知风险，用 reward 把 agent 推向“已知规则之外但仍像异常操作”的区域。
- 作者给出的证据：仿真与真实 Tornado Cash 10 ETH 合约 replay 后余额一致；simulated crowd 的 address reuse rate 为 2.6%，接近真实交易的 2.54%；PPO agent 在 Exp-1 中 task reward 0.999、invalid rate 0.129%、evading rate 99.871%；二次分析从约 10,000 个 episodes 中人工看 300 个，找到 3 个 gas price 相关模式，迭代后又发现 1 个 linked address 绕过模式。
- 我的判断：论文的问题设定很有价值，且“detector + evader + clustering + expert validation”这条路线比纯人工测试更可扩展。但它真正输出的仍是候选规则，不是自动证明；每个 pattern 的实际匿名性损害需要后续统计或攻击模型量化。

## Background

- 背景：Tornado Cash 这类 mixing service 把一次转账拆成 deposit 和 withdraw。用户先从一个地址向 mixer deposit，随后用 Note / zk-SNARK proof 从另一个地址 withdraw 到接收方，从而切断公开链上的直接交易链接。
- 问题：链上所有交易公开，攻击者可以按地址、时间、gas price、金额、交易顺序等特征重建关联。即使密码学链接被隐藏，用户若操作粗心，仍可能把 deposit 和 withdraw 暴露成同一用户行为。
- Gap：已有匿名性分析能从历史交易中找出 known patterns，但开发者在服务部署前并不知道未来会出现哪些行为组合。纯人工枚举也很难覆盖复杂的交互空间。

## Threat Model / Assumptions

- 攻击者或参与者能力：攻击者观察公开链上交易，尝试把 deposit、withdraw、外部 transfer 或地址集合关联起来；论文不考虑攻破 zk-SNARK、偷 Note、破坏合约实现或链下身份数据库泄露。
- 链上 / 链下假设：GuideEnricher 在仿真环境中运行，不直接分析真实用户隐私；simulator 需要足够逼真地复现 mixer 的关键状态变化、余额更新、deposit/withdraw 语义和 background crowd 行为。
- Guidebook 假设：初始 guidebook 来自 Tornado Cash 文档和已有研究，主要包含 Address Match、Unique Gas Prices、Linked ETH Addresses、TORN Mining 等已知规则。
- 人工判定假设：候选 pattern 是否真能 compromise anonymity，需要熟悉 Ethereum 和 mixing service 的专家给出解释；论文暂未提供可自动判定的定量标准。
- 不覆盖的情况：混币服务密码学安全性、合约漏洞、真实攻击者成本、guidebook 规则对用户可用性的量化影响、不同链和不同 mixer 的完整泛化。

## Method

- 核心思路：GuideEnricher = blockchain simulator + mixer contract model + crowd users + original guidebook + rule-based detector + DRL evader + clustering/manual analysis + iterative detector update。
- 核心思路是怎么想到的：匿名性破坏模式本质是交易序列中的可链接行为。手写规则不完整，历史分析又滞后；因此让 agent 在“完成转账但不要触发已知规则”的压力下探索行为空间，类似让 fuzzer 在反馈引导下寻找程序漏洞。
- 从 motivation 到 method 的逻辑链：guidebook 不完整 -> 先把已有 guidebook 编成 detector -> 构造一个容易诱发匿名性泄漏的转账任务 -> 用 reward 惩罚 invalid actions 和被 detector 抓到的 known patterns -> agent 学会绕过 detector -> 聚类 evader traces，人工从代表样本中提取新 patterns -> 把新 patterns 加回 detector / crowd rules，继续迭代。
- 关键设计取舍：论文选择单 agent evader，而不是同时训练 detector agent。这样系统更简单，但 detector 更新仍需要人把新 pattern 写成规则。作者讨论了双 agent 自动更新 detector 的可能性，但认为系统设计成本较高，留作未来工作。
- 为什么不是更直接 / 更简单的方案：直接从真实链上挖 pattern 会回到 postmortem；直接人工枚举规则覆盖不了大行为空间；直接让 agent 最大化“未知 pattern”不可行，因为未知 pattern 没有标签。GuideEnricher 用 detector 作为负反馈，把 agent 推到已知规则的边界之外。
- 系统流程或算法步骤：
  - Simulator：复现 Ethereum 上 token transfer、wallet balance、block/transaction、Tornado Cash deposit/withdraw、Note 验证和交易合法性；为效率，Note 用明文代替真实 zk proof。
  - Crowd users：生成背景交易，遵守原始 guidebook，并与 TC、其他 simulated contracts 交互，用来模拟真实链上噪声和等待期间的交易流。
  - Original guidebook / detector：把已知 anonymity-compromising patterns 编成 rule-based detector，用来判断 evader 动作是否违反 guidebook。
  - Evader initialization：给 agent 一个 challenge table，例如 3 个钱包各 3 tokens，另有 247 个空钱包；任务是把 tokens 转到空钱包，分配方式不限制。
  - State / observation：11 维向量，包含上一步 action、wait time、deposit/withdraw address、deposit 总数、相关地址余额、TC 合约当前余额、evader 当前总余额、deposit/withdraw 地址是否来自 challenge table 等。
  - Action：`a = [a1, a2, a3, a4]`。`a1` 表示 deposit/withdraw，`a2` 是等待多少个 crowd transactions，`a3` 是 deposit wallet，`a4` 是 withdraw wallet。TC 固定每次 1 token，因此 action 不包含金额。
  - Reward：invalid action 得 `R = -10`；valid 但被 detector 抓到得 `R = -1`；valid 且绕过 detector 得 `R = 1`。作者没有使用最终 sparse reward，因为 dense reward 已足以训练 agent 完成任务并规避规则。
  - Policy learning：用 PPO 训练 policy network，环境由 OpenAI Gym 实现，DRL agent 基于 stable-baselines，分布式训练用 Ray。默认 policy network 是 2 层、每层 32 个 hidden perceptrons，ADAM learning rate 0.001，episode horizon 10,000。
  - Second analysis：筛掉 evading rate 低于 90% 的 episodes；用 DBSCAN 和 K-means 聚类；选 silhouette coefficient 较高的聚类结果；每个 cluster 取靠近 centroid 的 10 个代表 episodes，由人工检查并总结 pattern。
  - Iterative update：把发现的新 pattern 加入 detector 和 crowd guidebook，再重新训练 evader，寻找下一批 pattern。
- 关键定义 / 公式 / 不变量：
  - Anonymity-compromising pattern：一段会让攻击者更容易链接交易或地址的交易序列，而不是直接的密码学漏洞。
  - MDP：`<S, A, T, R>`，其中 `S` 是状态、`A` 是动作、`T` 是状态转移、`R` 是 reward。
  - Value function：论文用 `V_pi(s)` 表示从状态 `s` 出发按 policy `pi` 行动的期望总 reward。
  - PPO objective：使用 clipped surrogate objective 限制新旧 policy 的差异，以提高训练稳定性。
- 实现细节：论文说平均训练约 48 小时；代码、模型、scaled plot diagrams 和实验说明在 GitHub 仓库公开。Cyclone 被视为 TC copy；TC Nova 和 Railgun 需要扩展 action/observation space，因为它们支持任意金额或跨链式语义。

## Evaluation

- 实验思路：论文先验证 simulator realism，再测试 agent 是否能完成任务和绕过 detector；之后做 task/wallet 泛化、challenge table scalability、reward ablation、PPO 与 A2C/A3C 对比、TC Nova/Railgun 泛化，最后用聚类和人工分析验证是否真的能产出新 guidebook patterns。
- 评估指标：normalized reward、task finishing rate、invalid action rate、evading rate、address reuse rate、聚类后人工检查工作量、发现的 pattern 数量和解释性。
- 主要结果：
  - Exp-0 realism：重放真实 Tornado Cash 10 ETH 历史交易后，simulator balance 与真实合约余额一致；crowd address reuse rate 为 2.6%，真实交易为 2.54%。实验每轮收集约 1M-1.5M transactions，明显高于真实 TC 历史约 45,000 笔，因此更偏向探索而不是还原真实频率。
  - Exp-1 effectiveness：默认任务下 reward 0.999，invalid rate 0.129%，evading rate 99.871%，说明 agent 很快学到合法交易和绕过已知 detector 的策略。
  - Exp-2 generalizability：改变 token 数和空钱包数后，多数组合 evading rate 仍接近 99.8%-99.9%；但 6 tokens / 247 wallets 场景降到 91.102%，作者解释为任务更简单，agent 更容易选中 guidebook 内已知风险动作。
  - Exp-3 scalability：在 `[500] -> 999` 和 `[50 * 10 wallets] -> 1000 wallets` 这类更大 challenge tables 上，reward 仍能收敛到较高值，说明方法可以为较大任务训练有效 evader。
  - Exp-4 reward ablation：把 invalid penalty 从 -10 改到 -1 后，任务完成率仍为 100%，evading rate 99.884%；增加 final reward 后 evading rate 反而降到 94.214%，说明逐步 dense reward 比终局奖励更有效。
  - Exp-5 algorithm comparison：PPO 收敛到最优 reward，而 A3C / A2C 没有达到同等效果，支撑作者选择 PPO。
  - Exp-6 mixer generalization：在 TC Nova 和 Railgun 上，agent reward 也能收敛到高值；Cyclone 因与 TC 功能近似，无需额外改动。
  - Pattern discovery：从约 10,000 episodes 中，聚类后只需人工检查 300 episodes，耗时约 12 小时，发现 3 个 gas price 相关 patterns：多个 deposit 后 withdraw、一个 deposit 后多个 withdraw、先完成多个 deposit 再 withdraw。它们都绕过了“单对 deposit/withdraw 使用相同 gas price”的规则，但仍可能暴露高频同源行为。
  - Pattern prevalence：三个 pattern 在 cluster 中的占比分别为 0.15%、0.11%、0.08%；作者说它们还未在真实历史交易中出现，预期是因为仿真交易量远大于真实 TC 历史。
  - Iterative discovery：把 3 个新 patterns 加入 detector 后重新训练，Table 3 中各配置任务完成率均为 100%，invalid rate 约 0.039%-0.065%，evading rate 约 99.348%-99.961%。再次人工分析发现一个绕过 Linked Address 规则的新 pattern：先把 origin wallet 的 token 转给 fresh wallets，再由 fresh wallets 与 TC 交互，从而避开唯一 deposit-withdraw 地址对规则。

## Key Artifacts

- 关键图：
  - Figure 1：Tornado Cash deposit/withdraw 流程，说明用户用不同 wallet 和 Note 完成一次 mixer 转账后，链上只看到相互独立的 deposit 与 withdraw。
  - Figure 2：等待时间示例。无等待最容易链接；有足够 TC 相关交易夹在中间更难链接；仅等待 wall-clock time 但期间没有其他 TC 交易仍可能暴露链接。
  - Figure 3：GuideEnricher 架构图，显示 evader agent、simulator with TC、detector、guidebook、crowd、state/action/reward 闭环。
  - Figure 4：Exp-1 和 Exp-3 reward 收敛图，支撑 agent 在默认和大 challenge table 下都能学到高 reward policy。
  - Figure 5：PPO vs. A3C/A2C，支撑 PPO 在该任务上更稳定有效。
  - Figures 6 / 7：TC Nova 和 Railgun 上的 reward 曲线，支撑方法可迁移到其他 mixing services。
  - Figure 8：三个 gas price 相关 anonymity-compromising patterns，是论文最核心的 pattern discovery 证据。
  - Figure 9：迭代后发现的 linked address 绕过模式，说明 detector 更新后 evader 还能继续探索新模式。
  - Figures 10 / 11：reward penalty 和 policy network 敏感性实验，支撑 reward / hyperparameter 选择不完全脆弱。
- 关键表：
  - Table 1：Exp-1 agent 在不同 token / wallet 设置下的 reward、invalid rate、evading rate。
  - Table 2：reward ablation，显示 dense reward 已足够，额外 final reward 反而削弱 evasion。
  - Table 3：更新 detector 后的 retrained agents 表现；论文正文称 six settings，但表里可见 5 个配置，这是一个小的不一致。
  - Table 4：11 维 observation 解释，是复现 agent state design 的关键表。
  - Table 5：跨 challenge table 泛化很差，V-1/V-2/V-3 invalid rate 分别达到 47.717%、25.073%、87.905%，说明 agent 不是一次训练即可通吃所有任务。
  - Table 6：初始 guidebook，包含 Address Match、Unique Gas Prices、Linked ETH Addresses、TORN Mining；其来源对应 Tutela 和 Wang et al. 的混币隐私研究。
- 关键公式 / 定义 / 算法：MDP 定义、`V_pi(s)` value function、PPO clipped surrogate objective、`a = [a1, a2, a3, a4]` action encoding、`R = {-10, -1, 1}` reward design。
- 这些证据分别支撑哪些结论：Figure 3 / Table 4 支撑系统建模；Figures 4-7 和 Tables 1-3 支撑训练有效性、泛化和迭代能力；Figure 8 / Figure 9 支撑“能发现新 pattern”的主张；Table 5 反过来说明泛化有限。

## Findings

- 发现 1：GuideEnricher 的核心贡献是把 guidebook 扩展问题做成 feedback-guided exploration。detector 不只是评估器，而是定义了“已知风险边界”，让 agent 往边界外探索。
- 发现 2：论文发现的 patterns 不依赖复杂合约漏洞，而是利用用户操作习惯和链上元数据，例如 gas price、交易相邻性、fresh wallet 分发路径。这说明混币匿名性很大一部分是 operational privacy，而不只是密码学。
- 发现 3：仿真规模是双刃剑。更大的 episode/transaction 数能让罕见 pattern 出现，但也意味着 pattern 在真实世界中的发生概率和攻击价值并未被直接验证。
- 发现 4：agent 泛化能力不是论文最强点。作者自己在 Table 5 说明跨 challenge table 迁移会显著退化；因此 GuideEnricher 更适合“为一个任务训练一个探索器”，而不是训练一个通用混币策略模型。
- 发现 5：人工仍然是闭环核心。聚类把 10,000 episodes 压到 300 个，但最终 pattern 是否成立、能否写进 guidebook，仍靠专家解释。

## Strengths

- 论文最有说服力的地方：问题设定很干净，把“部署前主动发现 guidebook 缺口”与“事后 deanonymization”区分开来，避免了直接依赖真实用户隐私泄露案例。
- 方法、数据、实验或问题设定的优势：simulator realism 做了差分验证；实验覆盖 effectiveness、generalizability、scalability、ablation、algorithm comparison、mixer generalization 和 iterative update；不仅报告 reward，还报告 invalid / evading rate。
- 相比已有工作的有效推进：Tutela、Wang et al. 等工作从历史交易中总结规则，GuideEnricher 则能在没有真实历史 pattern 的情况下主动生成候选风险模式。它更像隐私 guidebook 的 fuzzing harness，而不是又一个链上聚类工具。
- 工程可复用性：agent state/action/reward 设计清晰，代码公开，适合后续研究把 simulator 扩展到其他 mixer、桥、隐私钱包或 privacy pool。

## Limitations

- 威胁模型、假设或适用范围的限制：论文不证明 pattern 对匿名集大小的实际影响，只要求专家能解释其可链接性；“可解释”不等价于“高概率真实攻击成功”。
- 数据集、baseline、metric、ablation 或复现性的不足：缺少与系统化 pattern generation / fuzzing baseline 的直接比较；pattern discovery 的核心指标偏人工工作量和发现案例，缺少 precision/recall 或攻击成功率。
- Simulator 外部有效性：TC replay balance 一致只能说明基础状态迁移正确，不代表真实用户策略、钱包软件、gas price 行为、mempool 条件和市场环境都被充分建模。
- 真实世界频率问题：论文承认三个新 gas price patterns 尚未在真实历史交易中出现。若真实用户几乎不会这么操作，guidebook 增补可能增加认知负担但收益有限。
- Human-in-the-loop 成本：300 episodes / 12 小时是合理但不小的人工成本；扩展到多个协议、多轮迭代、多种链时，专家审核仍可能成为瓶颈。
- 可用性风险：guidebook 越扩越厚，用户越难遵守。论文提到这会损害 mixing service utility，但没有定量评估。
- 泛化限制：Table 5 显示 agent 跨 challenge table 效果差；TC Nova / Railgun 只是训练曲线收敛，并不等同于发现同等质量的新 patterns。
- 伦理和双用风险：工具可用于保护用户，也可能帮助攻击者寻找新的链接策略；论文以类比 fuzzing 回应，但实际发布和使用仍需要治理。

## My Takeaways

- 对 DeFi / 区块链安全研究的启发：隐私系统的安全性不能只看密码学原语，还要看用户操作、钱包默认行为、链上公开元数据和协议 guidebook 是否足够具体。
- 可复用的方法：把已有安全规则实现成 detector，再训练 evader 绕过 detector，是一种很通用的“规则补全”范式。它可以迁移到 MEV 防御规则、DeFi 风控规则、钱包安全提示、跨链桥操作安全等场景。
- 可能的后续问题：如何把 pattern 的可链接性变成定量指标，例如 anonymity set reduction、linking confidence、攻击成本或 false guidance cost？如何用 LLM / program synthesis 自动把人工总结的 pattern 转为 detector rule？如何在不泄露真实用户隐私的情况下验证候选 pattern 的现实发生率？

## Related Papers

- 前置阅读：Tutela: An Open-Source Tool for Assessing User-Privacy on Ethereum and Tornado Cash；On How Zero-Knowledge Proof Blockchain Mixers Improve, and Worsen User Privacy；Blockchain is Watching You: Profiling and Deanonymizing Ethereum Users。
- 后续阅读：SquirRL: Automating Attack Analysis on Blockchain Incentive Mechanisms with Deep Reinforcement Learning；privacy pool / mixer guidebook 相关研究；DRL explainability for security。
- 可对比论文：BERT4ETH 的 Tornado de-anonymization 任务、TTAGN / graph-based identity inference、MonLAD、Tornado Cash / TC Nova / Railgun 隐私分析。
- 最接近的相关工作：Tutela 和 Wang et al. 是最接近的混币服务隐私规则来源；SquirRL 是最接近的区块链 DRL 安全探索工作。
- 关键差异：Tutela / Wang et al. 从真实历史中总结已经发生的关联模式；GuideEnricher 在仿真环境中主动生成候选行为模式，并通过 detector evasion 把探索集中在 guidebook 缺口上。

## Open Questions

- 三个 gas price patterns 和一个 linked address pattern 如果用真实链上攻击模型评估，能让 anonymity set 平均缩小多少？
- 如何避免 guidebook 规则过多后普通用户无法执行，最终反而放弃隐私最佳实践？
- 能否把人工解释环节转成半自动规则生成和验证，例如从 cluster trace 自动生成 detector predicate？
- 如果 attacker 知道 GuideEnricher 的 detector 和 reward，是否可以反过来训练更强的 linking strategy？
- 对 Privacy Pools、Railgun、跨链 mixer 或账户抽象钱包，state/action/reward 需要怎样改，才能发现真正与这些系统相关的新 pattern，而不是只复用 TC 逻辑？
