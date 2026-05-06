# Blockchain Paper Reading

区块链论文阅读记录。当前主要方向：**DeFi**。

后续每给定一篇论文，就在 `notes/defi/` 下整理一篇独立笔记，并在本页按会议与年份补充索引。

状态：`TODO` 待读，`READING` 在读，`DONE` 已整理，`REVISIT` 待复读。

## Conference Index

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

### USENIX Security

- 2023
  - `[TODO]` [A Large Scale Study of the Ethereum Arbitrage Ecosystem](https://www.usenix.org/system/files/sec23fall-prepub-515-mclaughlin.pdf)

### The Web Conference / WWW

- 2022
  - `[TODO]` [Cyclic Arbitrage in Decentralized Exchanges](https://arxiv.org/pdf/2105.02784.pdf)

## Note Files

- Template: [notes/_template.md](notes/_template.md)
- DeFi notes: [notes/defi/](notes/defi/)

## Codex Skill

- Skill source: [skills/paper-deep-reading/](skills/paper-deep-reading/)
- Local install target: `~/.codex/skills/paper-deep-reading/`

### How to Use

This repo includes a Codex skill for DeFi paper deep reading. To install or refresh it locally:

```bash
mkdir -p ~/.codex/skills
cp -R skills/paper-deep-reading ~/.codex/skills/
```

After installation, ask Codex with a paper title, PDF path, arXiv URL, publisher URL, or GitHub/artifact link. Example prompts:

```text
精读这篇论文：https://arxiv.org/abs/xxxx.xxxxx
```

```text
读一下 /path/to/paper.pdf，并整理到 notes/defi/
```

The skill will:

- create one Chinese deep-reading note under `notes/defi/`
- focus on motivation, basic idea, method reasoning, evidence, and limitations
- include a sharp fact-grounded `毒舌评论`
- update the conference index in this README when appropriate
