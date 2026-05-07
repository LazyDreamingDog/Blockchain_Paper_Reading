# 【FC‘2026】Real AI Agents with Fake Memories: Fatal Context Manipulation Attacks on Web3 Agents

## Metadata

- Title: Real AI Agents with Fake Memories: Fatal Context Manipulation Attacks on Web3 Agents
- Authors: Atharv Singh Patlan, Peiyao Sheng, S. Ashwin Hebbar, Prateek Mittal, Pramod Viswanath
- Venue / Year: Financial Cryptography and Data Security (FC), 2026. 注：arXiv 页面仍标为 preprint；FC 2026 信息来自作者主页和机构发布信息。
- Note Name: 【FC‘2026】real-ai-agents-fake-memories
- Paper: https://arxiv.org/abs/2503.16248
- arXiv Version: v3, last revised 2025-07-09
- Local PDF: `/Users/wenkuanxiao/Zotero/storage/8QVNAJNT/Patlan 等 - 2025 - Real AI Agents with Fake Memories Fatal Context Manipulation Attacks on Web3 Agents.pdf`
- Zotero: 我的文库 / 污染交易 / agentic wallet
- Code: Unknown. arXiv 页面、PDF 和当前公开搜索结果未发现 CrAIBench 代码仓库。
- Dataset / Artifact: CrAIBench 在论文中描述；Hugging Face 上有与本文 arXiv 关联的 `SentientAGI/crypto-agent-safe-function-calling` 数据集，5.2k train rows，主要对应微调防御数据而非完整 CrAIBench benchmark。
- Scope / Subfield: Agentic Wallet 安全 / Web3 Agent 安全
- Tags: Web3 Agent, DeFi, Agentic Wallet, Context Manipulation, Memory Injection, Prompt Injection, ElizaOS, CrAIBench, Function Calling, LLM Agent Security, Fine-tuning Defense
- Status: DONE

## TL;DR

这篇论文把 Web3 AI agent 的攻击面从传统 prompt injection 扩展到更根本的 context manipulation：攻击者不一定要在当前输入里塞恶意指令，而是可以污染 agent 的持久记忆，让后续干净请求触发错误的链上动作。作者在 ElizaOS 上展示了跨平台 memory injection：Discord 中伪造的历史对话会被共享记忆保存，之后 X 上的转账请求可能被改到攻击者地址。为系统评估风险，论文提出 CrAIBench，覆盖 Chain、Trading、DAO/NFT 三类 Web3 agent 任务；实验显示 memory injection 比 prompt injection 更难防，常规 prompt-level 和 prompt-injection detector 防护迁移效果很差，而针对性微调在单步任务上能把 ASR 从 85.1% 降到 1.7%。

## 毒舌评论

这篇论文真正刺痛人的地方不是“LLM 会听坏话”，而是证明了很多 agent 架构把未认证历史当成事实来源：一旦 memory 被污染，越强的“遵循上下文”能力越可能变成执行假记忆的能力。它最可能被高估的地方也很明确：CrAIBench 和 ElizaOS case study 说明了风险形态，但公开复现实物还不够完整，防御实验主要证明“现有 prompt 防线不对口”，还没有给出能支撑高价值金融 agent 上线的系统级 memory integrity 方案。

## Research Question

- 研究对象：拥有长期记忆、工具调用和链上操作能力的 Web3 / DeFi AI agents。
- 小领域范围：agentic wallet / crypto agent 的 context-level attack，尤其是 memory injection。
- 具体问题：当 agent 把历史对话、外部数据、用户输入和静态知识拼成执行上下文时，攻击者能否通过污染这些 context surfaces 诱导 unauthorized transfers、错误交易、治理误投票或协议违规。
- 为什么重要：Web3 agent 的输出不只是文本，可能触发不可逆链上交易；一旦 agent 管理钱包或执行 DeFi 操作，小错误会变成直接资金损失。
- 论文边界：重点是上下文污染导致的 agent 行为偏转；不讨论私钥盗窃、链共识攻击、智能合约本身漏洞利用或交易排序攻击。

## Motivation and Basic Idea

- Motivation：传统 prompt injection 主要看当前输入或外部检索内容是否有恶意指令，但真实 agent 依赖更宽的 context：用户 prompt、外部数据、静态知识、历史记忆、工具返回和平台共享状态。Web3 agent 又高度依赖长期记忆维持 DAO 社群协作、交易偏好和跨平台一致性，因此 memory 成了高价值且缺乏完整性保护的攻击面。
- Basic idea：把 AI agent 的上下文形式化为 `c_t = (p_t, d_t, k, h_t)`，其中 `h_t` 是交互历史。攻击者的目标不是直接绕过当前 prompt guard，而是让 `h_t` 含有恶意轨迹 `delta_h`，使未来 agent 在正常请求下仍把伪造历史当成可信事实。
- 这个 idea 如何回应 motivation：如果风险来自“agent 信任上下文”而不是“当前输入危险”，那么只过滤当前 prompt 没用；必须检查 memory 的来源、完整性、隔离边界和写入权限。
- 作者给出的证据：ElizaOS 上的间接 / 直接 memory injection 案例；Discord 注入影响 X 转账的跨平台演示；CrAIBench 中四类模型、三类 Web3 domain、多种防御设置的量化结果；Browser-Use / WebVoyager 泛化实验。
- 我的判断：论文的问题设定是成立的。它抓住了 agent 安全里一个经常被低估的 trust boundary：memory 不是中性的上下文缓存，而是会影响后续行动的持久状态。

## Background

- 背景：ElizaOS 等 Web3 agent 框架把 clients/providers、agent character、memory evaluators 和 plugins 组合起来，支持 Discord、X、REST API、区块链插件等跨平台交互。LLM 通常不直接接触私钥，而是输出结构化工具调用，由插件完成交易、发帖、查询或合约交互。
- 问题：这种架构表面上把 secrets sandbox 住了，但插件是否执行敏感动作取决于 orchestrator LLM 如何解释上下文。如果上下文里的历史对话被伪造，插件仍会忠实执行被 LLM 生成的恶意参数。
- Gap：已有 prompt injection / RAG poisoning / AgentPoison 类工作关注当前输入、外部知识库或检索库污染；本文强调的是 agent 自身运行时产生并持久保存的 memory 被污染，且影响可以跨 session、跨用户、跨平台传播。

## Threat Model / Assumptions

- 攻击者或参与者能力：攻击者可以像普通用户一样在 Discord、X、chat 或 API 中与 agent 交互；在直接 memory injection 情况下，攻击者还可能通过错误配置、insider compromise 或第三方存储后端问题获得 memory store 写权限。
- 链上 / 链下假设：agent 通过插件执行转账、交易、bridge、staking、governance、NFT 等操作；链上操作可能不可逆。模型本身不需要掌握私钥，只要能诱导插件参数出错即可造成损失。
- Memory 假设：agent 会把历史对话或摘要作为未来上下文的一部分；在 multi-user / DAO 环境中，多个用户可能共享同一 memory pool。
- 攻击目标：让 agent 选择攻击者指定的 action sequence，例如改写 `toAddress`、改变 leverage、替换 token、跳过确认、重定向工具调用或泄露私密资料。
- 不覆盖的情况：不评估底层 smart contract 漏洞、MEV、mempool frontrunning、链下身份认证、wallet signing UX、真实主网高价值资金损失统计。

## Method

- 核心思路：Context manipulation 是总攻击类，prompt injection 和 memory injection 都是它的子类。论文的关键推进是把 memory injection 形式化，并在 Web3 agent 框架中展示它如何绕过现有 prompt 防护。
- 核心思路是怎么想到的：agent 的行为函数依赖 context，而 context 不只是当前 prompt。只要攻击者能操纵 `p_t`、`d_t`、`k` 或 `h_t` 中任一部分，就能影响 action selection；其中 `h_t` 最危险，因为它持久、隐蔽、可被后续正常请求反复触发。
- 从 motivation 到 method 的逻辑链：Web3 agent 需要 memory 保持连续性 -> memory 在多用户/跨平台场景中经常共享 -> 当前 guardrail 主要审查新输入 -> 伪造为“已经发生过的历史”的恶意内容不再像新指令 -> agent 把它当成事实或已确认偏好 -> 插件按错误参数执行链上动作。
- 关键设计取舍：论文没有设计一个完整防御系统，而是先把攻击面拆成形式化模型、真实框架 case study、benchmark 和防御探索。这个顺序合理，因为 memory injection 的核心问题是系统边界，不是单个 jailbreak prompt。
- 为什么不是更直接 / 更简单的方案：单纯测试 prompt injection 会低估风险，因为现实 agent 的动作经常由 retrieved memory、历史摘要和外部数据共同决定。只看当前 user prompt 无法解释“用户请求是干净的，但 agent 仍转错账”的情形。
- 系统流程或算法步骤：
  - Formal agent framework：上下文 `c_t = (p_t, d_t, k, h_t)`，分别代表 user prompt、external data、static knowledge 和 history；decision engine `M` 根据 `c_t` 输出 action sequence 的分布。
  - Context manipulation：攻击者注入有界扰动 `delta`，得到 `c* = c_t ⊕ delta`，目标是提高 adversary-chosen action `a*` 的概率。
  - Memory injection：攻击者污染 history，形成 `c* = (p_t, d_t, k, h_t ⊕ delta_h)`。直接注入依赖 memory backend 写权限；间接注入通过普通交互诱导 agent 自己把恶意内容写进 memory。
  - Prompt injection 对照：direct prompt injection 污染 `p_t`；indirect prompt injection 污染 `d_t`，例如 API response、网页、链上数据或其他外部数据源。
  - ElizaOS 间接 memory injection：攻击者构造一段看起来像完整历史对话的内容，其中包含伪造的用户指令和伪造的 agent 接受回复，最后以无害问题结束。ElizaOS 只响应最后的正常问题，但整段内容会被写入 memory。
  - 触发阶段：后续用户提出转账、copy-trade、交易或治理请求时，agent 检索到被污染的历史，LLM 生成恶意 JSON/tool call，插件按错误参数执行。
  - Cross-platform propagation：因为 Discord、X、链上插件共享同一个 orchestrator 和 memory，Discord 注入可以影响 X 上的后续交易请求。
  - CrAIBench：构造 Chain、Trading、DAO/NFT 三类 agent-execution domains，同时支持 benign operation、corrupted memory 和 corrupted action output，用于比较 PI 与 MI。
  - 防御探索：比较 prompt-level defenses、model-based PI detectors，以及对 Qwen-2.5-14B-Instruct 的 security-augmented SFT。
- 关键定义 / 公式 / 不变量：
  - `c_t = (p_t, d_t, k, h_t)`：agent 在时间 `t` 的上下文。
  - `M: C -> Delta(A)`：decision engine 从上下文映射到 action sequence 分布。
  - `a_t = argmax_a P(a | c_t)`：agent 选择最高概率行动序列。
  - `c_{t+1} = F(c_t, a_t)`：行动更新环境和上下文，尤其会追加历史 `h_t`。
  - CM-MI：污染 `h_t` 的 context manipulation；CM-PI：污染 `p_t` 或 `d_t` 的 context manipulation。
  - Sleeper injection：恶意 memory 在写入时不触发明显异常，等待特定未来请求激活。
  - Cross-session / cross-user backdoor：一个用户或 session 写入的假记忆影响另一个用户或之后的 session。
- 实现细节：
  - ElizaOS case study 中，作者利用共享 memory、消息格式和插件编排机制完成攻击；示例包括改变 copy-trade leverage 和重定向 ETH transfer address。
  - CrAIBench 中 memory injection 被包装成 retrieved memory block，prompt injection 被放在指定 PI slots；两者使用相同核心 injection strings，便于比较攻击面差异。
  - Fine-tuning defense 使用 APIGen Function-Calling Datasets 中的 blockchain-related queries 扩展出 memory injection variants，并加入 reasoning trajectory。

## Evaluation

- 实验思路：第一，用 ElizaOS 做真实框架 case study，验证 memory injection 可以跨平台触发链上操作。第二，用 CrAIBench 在 Chain、Trading、DAO/NFT 上比较 PI 与 MI，测试 GPT-4o-mini、GPT-4o、Claude Sonnet 3.5、Claude Sonnet 3.7。第三，评估 prompt-level defense、PromptGuard 2 / ProtectAI 2 / DataSentinel detector、fine-tuning defense。第四，把 memory injection 扩展到 Browser-Use / WebVoyager web-navigation agents，测试非 Web3 场景。
- 评估指标：
  - Benign Utility：无攻击时完成用户任务的比例。
  - Utility Under Attack：攻击下仍正确执行用户意图且没有 adversarial side effects 的比例。
  - Targeted Attack Success Rate (ASR)：agent 满足攻击者目标的比例。
- CrAIBench 规模：
  - Chain：9 actions，25 multi-step tasks / 50 injections，7 single-step tasks / 14 injections。
  - Trading：13 actions，78 multi-step tasks / 390 injections，16 single-step tasks / 80 injections。
  - DAO/NFT：11 actions，32 multi-step tasks / 124 injections，8 single-step tasks / 27 injections。
  - Total：33 actions，135 multi-step tasks / 564 injections，31 single-step tasks / 121 injections。
- 主要结果：
  - MI vs PI：四个模型上 memory injection 的 ASR 都显著高于 prompt injection。Fig. 5 中 GPT-4o-mini 的 MI 约 82%，PI 约 45%；Claude Sonnet 3.5 的 MI 约 63%，PI 约 34%；GPT-4o 的 MI 约 73%，PI 约 36%；Claude Sonnet 3.7 的 PI 接近 0，但 MI 仍为 55.1%。
  - 模型能力关系：模型越强，PI 和 MI ASR 都下降；但 PI 下降得更快，MI 即使在最强模型上仍显著存在，说明推理能力和普通 prompt guard 不能自动解决 memory trust 问题。
  - Prompt-level defense：Default、Secure、Confirmation 三类 system prompt 叠加 sandwiching/delimiter 后，prompt injection 在多域中明显下降；memory injection 仍保持较高 ASR，尤其 DAO/NFT 中的精心构造记忆即使要求确认也能绕过不少检查。
  - Model-based detection：在 GPT-4o agent 上，PromptGuard 2 是三者中最好的一档，但仍漏掉约一半恶意 memory updates；ProtectAI 2 有相近但稍弱的改进；DataSentinel 在该设置下几乎不改善 baseline，并且 benign false alarm 很高。
  - Fine-tuning defense：Qwen-2.5-14B-Instruct 默认 benign utility 83.9%，utility under MI 44.6%，ASR 85.1%；prompt-based defense benign utility 83.9%，utility under MI 19.0%，ASR 55.4%；fine-tuning defense benign utility 87.1%，utility under MI 85.1%，ASR 1.7%。
  - Fine-tuning 数据：作者选取 2,199 个 blockchain-related function-calling queries 扩展 memory injection variants，并加入 3,000 个 benign tasks；训练 3 epochs，8 张 H100，learning rate `1e-5`，5% warmup，cosine scheduler，weight decay `1e-4`。
  - Ablation：没有 reasoning trajectory 的训练也能降 ASR，但约为 8.3%，比完整设置高约 5 倍；只用 adversarial examples 会降低 benign utility 到约 80%。
  - WebVoyager 泛化：作者从 WebVoyager 采样 9 个 web navigation tasks，以私密信息外传为攻击目标，在 Browser-Use 上测试。未防御时 PI 和 MI ASR 都超过 80%；加入 secure prompt 和 sandwiching 后，PI 大幅缓解，但最弱形式的 MI 仍有非平凡 ASR。

## Key Artifacts

- 关键图：
  - Fig. 1：跨平台 memory injection 流程图，展示 Discord 中的假历史如何在 X 上触发高杠杆 copy-trade。
  - Fig. 2：通用 AI agent 架构，包括 Perception Layer、Memory System、Decision Engine、Action Module。
  - Fig. 3：context manipulation 攻击面，把 direct prompt injection、indirect prompt injection、memory injection 放到同一信息流图中。
  - Fig. 4：ElizaOS memory injection 构造示例，说明恶意指令必须伪装成已完成的历史 user-agent exchange，并以正常问题收尾。
  - Fig. 5 / Fig. 6：不同模型的 PI/MI ASR 对比，以及 ASR 与 benign utility 的关系，支撑“强模型缓解但不能消除 MI”。
  - Fig. 7：不同 system prompt 和 domain 下的 ASR，支撑“prompt-level defenses 对 PI 更有效，对 MI 迁移不足”。
  - Fig. 8：PromptGuard 2、ProtectAI 2、DataSentinel 与 baseline 的 ASR 对比，支撑“PI detector 对 memory 伪装历史不对口”。
  - Fig. 10：Qwen-2.5-14B-Instruct 微调防御结果，支撑“针对性 alignment 能显著降低单步任务 MI ASR”。
  - Fig. 11：WebVoyager / Browser-Use 泛化实验，支撑“memory injection 不限于 Web3 agent”。
  - Fig. 12 / Fig. 13：Discord memory injection 与 X 上 Sepolia transfer 演示，支撑真实框架中的跨平台触发链。
  - Fig. 14：Confirmation system prompt，说明作者用于链上敏感操作确认防御的 prompt baseline。
- 关键表：
  - Table 1：CrAIBench domain/action/task/injection 规模，是 benchmark 主体证据。
- 关键公式 / 定义 / 算法：
  - `c_t = (p_t, d_t, k, h_t)` 和 `c* = c_t ⊕ delta` 是全文攻击面统一框架。
  - CM-MI 的核心公式是把 `h_t` 替换为 `h_t ⊕ delta_h`。
  - Benign Utility、Utility Under Attack、ASR 三个指标决定实验解释。
- 这些证据分别支撑哪些结论：
  - Fig. 1/12/13 支撑“攻击不是纯理论，可以在 ElizaOS 这类真实框架上触发插件动作”。
  - Fig. 5/6/7/8 支撑“MI 比 PI 更难防，且现有 PI 防御迁移不足”。
  - Fig. 10 支撑“模型对 memory 中的恶意轨迹做针对性训练有希望，但当前只在单步任务上证明”。
  - Fig. 11 支撑“memory integrity 是通用 agent 问题，不只是 DeFi 问题”。

## Findings

- 发现 1：Memory injection 不是“持续时间更长的 prompt injection”，而是攻击了 agent 的信任根。它把恶意内容伪装成已经被接受的历史事实，让后续新输入看起来完全正常。
- 发现 2：Web3 agent 的 sandboxed secrets 设计只能防止模型直接看见私钥，不能防止模型生成错误工具参数。只要插件信任 orchestrator LLM 的 JSON/action output，memory poisoning 仍能变成链上动作。
- 发现 3：更强模型能降低 PI ASR，但不能消除 MI；这说明问题不只是模型“聪不聪明”，而是系统把 memory 当 trusted context 的架构假设错了。
- 发现 4：PromptGuard、ProtectAI、DataSentinel 这类 PI detector 的目标分布和 MI 不匹配。MI 看起来像过去已执行的历史，而不是当前要执行的恶意指令。
- 发现 5：Fine-tuning defense 的结果很强，但它更像证明“模型可以学会拒绝恶意记忆形态”，不是完整替代 memory provenance、isolation、access control 和 out-of-band confirmation。

## Strengths

- 论文最有说服力的地方：把一个直觉风险落成了完整证据链：形式化 -> ElizaOS 实证攻击 -> CrAIBench benchmark -> 防御对比 -> 非 Web3 泛化。
- 方法、数据、实验或问题设定的优势：CrAIBench 明确区分 PI slots 和 memory retrieval context，用相同核心 injection strings 比较 attack surface，避免把“攻击文案强弱”混成“攻击面强弱”。
- 相比已有工作的有效推进：相比传统 prompt injection 研究，本文把 persistent state 和 multi-user shared memory 放到中心；相比 RAG poisoning，它关注 agent 自身历史而非外部知识库；相比 AgentPoison，它更强调 Web3 高风险动作和实际框架插件链路。
- 工程启发明确：论文不是只说“加安全 prompt”，而是明确指出 memory integrity checks、strict validation、context isolation、out-of-band confirmation 和 context-aware model training 都是必要层次。

## Limitations

- 威胁模型、假设或适用范围的限制：ElizaOS 是代表性 case study，但不能直接覆盖所有 Web3 agent 架构；不同框架的 memory write policy、retrieval policy、tool authorization 和 confirmation UX 会显著影响风险。
- 数据集、baseline、metric、ablation 或复现性的不足：CrAIBench 完整代码/benchmark 当前未在论文或 arXiv 页面公开发现；公开可见的 Hugging Face 数据集更像 fine-tuning defense 数据，而不是完整攻击评测环境。
- 防御结论边界：Fine-tuning defense 主要在 31 个 single-step tasks 和 121 个 MI attacks 上评估；多步 DeFi 操作、真实钱包确认、跨平台长程记忆污染下是否同样有效还需要更多实证。
- 真实 DeFi 场景中的风险：论文没有度量真实部署 agent 中 memory store 的权限配置分布，也没有大规模统计真实资金损失；因此“危险性”证据强，“现实发生率”证据弱。
- 架构防御讨论仍偏原则：白名单、out-of-band confirmation、memory isolation 都会损害 agent 自动化体验；论文承认 trade-off，但没有给出可量化的安全/可用性设计空间。
- 可能的过度外推：把“fiduciarily responsible language models”作为长期方向很有启发，但这个概念目前更像目标描述，不是已经可验证的技术方案。

## My Takeaways

- 对 DeFi / 区块链安全研究的启发：agentic wallet 安全不应只审查 transaction intent，也要审查 intent 的来源链。一个安全 agent 需要知道“这条偏好/记忆/规则是谁写入的、何时写入的、是否可撤销、是否绑定用户、是否适用于当前链上操作”。
- 可复用的方法：把 context surfaces 拆成 prompt、external data、knowledge、history 的模型很实用，可用于分析其他 agent 框架；CrAIBench 的 domain/action/state/task/injection 结构也适合扩展到 lending、bridge、oracle、portfolio management。
- 对当前研究线的意义：如果后续研究做 agentic wallet 或 LLM-based DeFi automation，必须把 memory poisoning 当成一等威胁，而不是附带 prompt injection 小节。
- 可能的后续问题：怎样给 memory entry 加 cryptographic provenance？怎样把 multi-user shared memory 变成 per-user/per-task capability-scoped memory？怎样在不杀死自动化体验的情况下，对高风险链上动作做强确认？

## Related Papers

- 前置阅读：Prompt Injection Attack against LLM-Integrated Applications；Not what you've signed up for；AgentDojo；InjecAgent；AgentPoison；Eliza: A Web3 Friendly AI Agent Operating System。
- 后续阅读：Context Manipulation Attacks: Web Agents are Susceptible to Corrupted Memory；Commercial LLM Agents are Already Vulnerable to Simple yet Dangerous Attacks；Defeating Prompt Injections by Design；ShieldAgent。
- 可对比论文：AgentPoison 关注 memory / knowledge base poisoning；BadChain 和 Sleeper Agents 关注 backdoor / CoT 风险；DataSentinel、PromptGuard、ProtectAI 代表 PI detection 防线；HOUSTON / DeFiScope 可作为 DeFi 安全中“链上行为检测”方向的对照。
- 最接近的相关工作：Rehberger 的 ChatGPT memory prompt injection blog 和 Dong et al. 的 practical memory injection attack 与本文最接近；本文的差异是把问题系统化到 Web3 agent、共享 memory 和链上插件执行。
- 关键差异：本文不是只证明一个 jailbreak trick，而是把 memory 作为 agent 系统状态来攻击；其核心贡献在于安全边界重画，而不是某个特定恶意 prompt。

## Open Questions

- CrAIBench 是否会公开完整运行环境、domain state、tools 和 attack cases？没有 benchmark，后续很难独立验证 MI/PI 的 ASR 差距。
- Memory entry 应该采用什么最小 provenance schema：author、platform、session、permission、expiry、scope、signature、human confirmation 哪些是必要字段？
- 对 DAO/group chat 的共享 agent，如何避免一个低权限用户污染高权限用户的交易上下文？
- Fine-tuning 学到的是“拒绝恶意 memory 模板”，还是更抽象的 fiduciary reasoning？面对新型伪装记忆时是否会退化？
- 如果攻击者长期、小步、低价值地投毒 memory，让恶意偏好逐渐看起来像正常历史，CrAIBench 当前设置能否覆盖这种慢速攻击？
- 对 agentic wallet 来说，真正应该保护的是 prompt、memory、tool schema、wallet confirmation 还是交易 simulation result？这些层之间如何组合成可证明的安全边界？
