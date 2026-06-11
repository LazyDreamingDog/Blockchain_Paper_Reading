# Blockchain Paper Reading

区块链论文阅读记录。当前主要方向：**DeFi**。

后续每给定一篇论文，就在 `notes/defi/` 下整理一篇独立笔记，并在本页按会议与年份补充索引。

状态：`TODO` 待读，`READING` 在读，`DONE` 已整理，`REVISIT` 待复读。

## Conference Index

### IEEE Transactions on Dependable and Secure Computing (TDSC)

- 2026
  - `[DONE]` [【TDSC‘2026】Unveiling Ethereum Mixing Services Using Enhanced Graph Structure Learning](notes/defi/【TDSC‘2026】lasc.md) ([DOI](https://doi.org/10.1109/TDSC.2025.3640941))
- 2024
  - `[DONE]` [【TDSC‘2024】DeFiRanger: Detecting DeFi Price Manipulation Attacks](notes/defi/【TDSC‘2024】defiranger.md) ([DOI](https://doi.org/10.1109/TDSC.2023.3346888))

### IEEE Transactions on Information Forensics and Security (TIFS)

- 2024
  - `[DONE]` [【TIFS‘2024】Toward Understanding Asset Flows in Crypto Money Laundering Through the Lenses of Ethereum Heists](notes/defi/【TIFS‘2024】xblockflow.md) ([DOI](https://doi.org/10.1109/TIFS.2023.3346276), [artifact](https://github.com/lindan113/EthereumHeist))
- 2023
  - `[DONE]` [【TIFS‘2023】TRacer: Scalable Graph-Based Transaction Tracing for Account-Based Blockchain Trading Systems](notes/defi/【TIFS‘2023】tracer.md) ([DOI](https://doi.org/10.1109/TIFS.2023.3266162), [code](https://github.com/wuzhy1ng/BlockchainSpider))

### IEEE/ACM ASE

- 2025
  - `[DONE]` [【ASE‘2025】Detecting Various DeFi Price Manipulations with LLM Reasoning](notes/defi/【ASE‘2025】defiscope.md) ([arXiv](https://arxiv.org/abs/2502.11521), [code](https://github.com/AIS2Lab/DeFiScope))

### ACM SIGMETRICS / PACM Measurement and Analysis of Computing Systems

- 2026
  - `[DONE]` [【SIGMETRICS‘2026】Shedding Light on Shadows: Automatically Tracing Illicit Money Flows on EVM-Compatible Blockchains](notes/defi/【SIGMETRICS‘2026】mftracer.md) ([DOI](https://doi.org/10.1145/3771578), [code](https://github.com/blocksecteam/MFTracer/tree/main/codes), [dataset](https://github.com/blocksecteam/MFTracer/tree/main/LaunderNetEvm41))

### NDSS

- 2026
  - `[DONE]` [【NDSS‘2026】HOUSTON: Real-Time Anomaly Detection of Attacks against Ethereum DeFi Protocols](notes/defi/【NDSS‘2026】houston.md) ([DOI](https://doi.org/10.14722/ndss.2026.241534))

### Financial Cryptography and Data Security (FC)

- 2026
  - `[DONE]` [【FC‘2026】Real AI Agents with Fake Memories: Fatal Context Manipulation Attacks on Web3 Agents](notes/defi/【FC‘2026】real-ai-agents-fake-memories.md) ([arXiv](https://arxiv.org/abs/2503.16248), [dataset](https://huggingface.co/datasets/SentientAGI/crypto-agent-safe-function-calling))

### IEEE S&P

- 2024
  - `[TODO]` [POMABuster: Detecting Price Oracle Manipulation Attacks in Decentralized Finance](https://www.dropbox.com/scl/fi/ye9ij7m782bbr2lckt20e/IEEES_P24-camera-ready.pdf)
  - `[TODO]` [Non-Atomic Arbitrage in Decentralized Finance](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C5&q=Non-Atomic+Arbitrage+in+Decentralized+Finance&btnG=)
- 2023
  - `[TODO]` [SoK: Decentralized Finance (DeFi) Attacks](https://arxiv.org/pdf/2208.13035.pdf)
  - `[TODO]` [Clockwork Finance: Automated Analysis of Economic Security in Smart Contracts](https://eprint.iacr.org/2021/1147.pdf)
- 2022
  - `[TODO]` [Quantifying Blockchain Extractable Value: How dark is the forest?](https://arxiv.org/pdf/2101.05511.pdf)
- 2021
  - `[TODO]` [On the Just-In-Time Discovery of Profit-Generating Transactions in DeFi Protocols](https://arxiv.org/pdf/2103.02228)
  - `[TODO]` [High-Frequency Trading on Decentralized On-Chain Exchanges](https://arxiv.org/pdf/2009.14021.pdf)
- 2020
  - `[TODO]` [Flash Boys 2.0: Frontrunning in Decentralized Exchanges, Miner Extractable Value, and Consensus Instability](https://par.nsf.gov/servlets/purl/10159474)

### ACM CCS

- 2024
  - `[TODO]` [FORAY: Towards Effective Attack Synthesis against Deep Logical Vulnerabilities in DeFi Protocols](https://www.arxiv.org/pdf/2407.06348)
  - `[TODO]` [Rolling in the Shadows: Analyzing the Extraction of MEV Across Layer-2 Rollups](https://ben-weintraub.com/files/rolling-in-the-shadows.pdf)
- 2023
  - `[TODO]` [Demystifying DeFi MEV Activities in Flashbots Bundle](https://zzzihao-li.github.io/papers/CCS23_Bundle_MEV_full_version.pdf)

### ACM/SIGAPP Symposium on Applied Computing (SAC)

- 2025
  - `[DONE]` [【SAC‘2025】Attacking Anonymity Set in Tornado Cash via Wallet Fingerprints](notes/defi/【SAC‘2025】wallet-fingerprints-tornado-cash.md) ([DOI](https://doi.org/10.1145/3672608.3707896))

### USENIX Security

- 2024
  - `[DONE]` [【USENIX‘2024】GuideEnricher: Protecting the Anonymity of Ethereum Mixing Service Users with Deep Reinforcement Learning](notes/defi/【USENIX‘2024】guideenricher.md) ([paper](https://www.usenix.org/conference/usenixsecurity24/presentation/de-silva), [code](https://github.com/ucsb-seclab/GUIDE-ENRICHER))
- 2023
  - `[TODO]` [A Large Scale Study of the Ethereum Arbitrage Ecosystem](https://www.usenix.org/system/files/sec23fall-prepub-515-mclaughlin.pdf)

### The Web Conference / WWW

- 2023
  - `[DONE]` [【WWW‘2023】BERT4ETH: A Pre-trained Transformer for Ethereum Fraud Detection](notes/defi/【WWW‘2023】bert4eth.md) ([DOI](https://doi.org/10.1145/3543507.3583345), [code](https://github.com/git-disl/BERT4ETH))
- 2022
  - `[TODO]` [Cyclic Arbitrage in Decentralized Exchanges](https://arxiv.org/pdf/2105.02784.pdf)

## Note Files

- Template: [notes/_template.md](notes/_template.md)
- DeFi notes: [notes/defi/](notes/defi/)

## Workflow Tool

The reusable Codex skill and agent workflow have been moved to:

[paper-reading-workflow](https://github.com/LazyDreamingDog/paper-reading-workflow)
