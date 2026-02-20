<div align="center">

# ρlατfοrm
# rθαdmαρ

**Distributed validator network for decentralized AI evaluation on Bittensor**

![Platform Banner](assets/banner.png)

</div>

---

## Overview

Platform Network operates as **Subnet 100** on the Bittensor blockchain. It is a decentralized incentive layer designed to continuously improve AI-powered development tools through competitive challenges. Miners on the network compete to produce the most effective harnesses for the [Cortex CLI](https://github.com/CortexLM/cortex) and [Cortex IDE](https://github.com/CortexLM/cortex-ide). The long-term objective is to train dedicated language models by leveraging [GRAIL](https://github.com/one-covenant/grail) (Subnet 81), an external protocol for cryptographically verifiable inference, using the datasets fabricated by network participants.

The core infrastructure is implemented in [platform-v2](https://github.com/PlatformNetwork/platform-v2), a fully decentralized peer-to-peer validator network built in Rust. Validators execute challenge logic inside a hardened WASM runtime, reach stake-weighted consensus over libp2p, and submit finalized weights to the Bittensor chain.

---

## Network Architecture

```mermaid
flowchart TB
    subgraph Cortex["Cortex Software Products"]
        CLI["Cortex CLI\n(github.com/CortexLM/cortex)"]
        IDE["Cortex IDE\n(github.com/CortexLM/cortex-ide)"]
    end

    subgraph Platform["Platform Network — Subnet 100"]
        PV2["platform-v2\nWASM Validator Network"]
        TC["Term Challenge\nAgent Harness Competition"]
        BC["Bounty Challenge\nBug Discovery & QA"]
        DC["Data Fabrication Challenge\nDataset Production"]
    end

    subgraph External["External Integration"]
        BT["Bittensor Chain\nWeight Submission"]
        GRAIL["GRAIL Protocol — Subnet 81\n(one-covenant/grail)\nVerifiable Post-Training"]
    end

    Miners["Miners\n(Network Participants)"] -->|Submit agent harnesses| TC
    Miners -->|Report bugs| BC
    Miners -->|Submit data fabrication harnesses| DC

    TC -->|Evaluate against| CLI
    TC -->|Evaluate against| IDE
    BC -->|Improve quality of| CLI
    BC -->|Improve quality of| IDE
    DC -->|Training data for| GRAIL

    PV2 -->|Hosts| TC
    PV2 -->|Hosts| BC
    PV2 -->|Hosts| DC
    PV2 -->|Consensus weights| BT
    BT -->|TAO emissions| Miners

    GRAIL -->|Fine-tuned models| CLI
    GRAIL -->|Fine-tuned models| IDE
```

Miners interact with three challenge modules hosted on the platform-v2 validator network. Each challenge produces measurable outputs (scores, valid issues, datasets) that validators aggregate through P2P consensus before submitting weights to the Bittensor chain. TAO emissions flow back to miners in proportion to their performance, completing the incentive loop.

---

## Challenges

Platform Network hosts dynamic challenges, each targeting a distinct aspect of software development quality. Challenges are implemented as WASM modules that run inside the platform-v2 validator runtime, ensuring deterministic and verifiable evaluation.

### Term Challenge — Agent Harness Competition

**Repository:** [PlatformNetwork/term-challenge-v2](https://github.com/PlatformNetwork/term-challenge-v2)

Term Challenge is the flagship competition of Platform Network. Miners submit Python agent packages that autonomously solve software engineering tasks drawn from SWE-bench datasets. The network evaluates these agents through a multi-stage review pipeline combining LLM-based code review and AST structural validation, executed by six deterministically selected validators per submission.

```mermaid
sequenceDiagram
    participant M as Miner
    participant V as Validators (P2P)
    participant LLM as LLM Reviewers (×3)
    participant AST as AST Reviewers (×3)
    participant W as WASM Module
    participant E as term-executor
    participant BT as Bittensor Chain

    M->>V: Submit agent ZIP + metadata
    V->>W: validate(submission)
    W-->>V: Approved (>50% consensus)
    V->>LLM: Assign LLM code review
    V->>AST: Assign AST structural review
    LLM-->>V: LLM review scores
    AST-->>V: AST review scores
    V->>E: Execute agent on SWE-bench tasks
    E-->>V: Task results + scores
    V->>W: evaluate(results)
    W-->>V: Aggregate score + weight
    V->>BT: Submit weights at epoch boundary
```

#### Submission Flow

The submission process enforces rate limiting (1 submission per 3 epochs per miner), size constraints (ZIP packages capped at 1 MB), and cryptographic authentication via sr25519 signatures. Each submission is versioned automatically under a first-register-owns naming system.

```mermaid
flowchart LR
    Register["Register Name\n(first-register-owns)"] --> Version["Auto-increment\nVersion"]
    Version --> Pack["Package Agent\nZIP ≤ 1MB"]
    Pack --> Sign["Sign with\nsr25519"]
    Sign --> Submit["Submit via\nJSON-RPC"]
    Submit --> RateCheck{"Epoch Rate\nLimit OK?"}
    RateCheck -->|"No: < 3 epochs\nsince last"| Reject["Rejected"]
    RateCheck -->|Yes| Validate["WASM\nvalidate()"]
    Validate --> Consensus{">50% Validator\nApproval?"}
    Consensus -->|No| Reject
    Consensus -->|Yes| Evaluate["Evaluation\nPipeline"]
    Evaluate --> Store["Store Code +\nHash + Logs"]
```

#### Validator Assignment

For each submission, six validators are deterministically selected from a seed derived from the submission ID. Three perform LLM-based code review and three perform AST structural analysis. If any reviewer times out, a replacement validator is automatically assigned, ensuring liveness.

```mermaid
flowchart TB
    Sub["New Submission"] --> Seed["Deterministic Seed\nfrom submission_id"]
    Seed --> Select["Select 6 Validators"]
    Select --> LLM["3 LLM Reviewers"]
    Select --> AST["3 AST Reviewers"]
    LLM --> LR1["LLM Reviewer 1"]
    LLM --> LR2["LLM Reviewer 2"]
    LLM --> LR3["LLM Reviewer 3"]
    AST --> AR1["AST Reviewer 1"]
    AST --> AR2["AST Reviewer 2"]
    AST --> AR3["AST Reviewer 3"]
    LR1 & LR2 & LR3 --> TD1{"Responded\nin time?"}
    AR1 & AR2 & AR3 --> TD2{"Responded\nin time?"}
    TD1 -->|No| Rep1["Replacement\nValidator"]
    TD1 -->|Yes| Agg["Result\nAggregation"]
    TD2 -->|No| Rep2["Replacement\nValidator"]
    TD2 -->|Yes| Agg
    Rep1 --> Agg
    Rep2 --> Agg
    Agg --> FinalScore["Final Score"]
```

#### Decay Mechanism

The decay function ensures no miner can rest on a single high-scoring submission. The weight multiplier $\mu(t)$ governs the time-dependent emission allocation:

$$
\mu(t) = \begin{cases}
1.0 & \text{if } t \leq 72\text{h (grace period)} \\[6pt]
2^{-\frac{t - 72}{24}} & \text{if } t > 72\text{h}
\end{cases}
$$

The effective weight $w_i$ for miner $i$ with raw score $s_i$ is:

$$
w_i = s_i \cdot \mu(t_i)
$$

After 72 hours of grace and 24 hours of decay ($t = 96$), the multiplier drops to $\mu(96) = 2^{-1} = 0.5$. After a further 24 hours ($t = 120$), it reaches $\mu(120) = 2^{-2} = 0.25$. The weight asymptotically approaches zero, eventually burning to UID 0.

```mermaid
flowchart LR
    Top["Top Score\nAchieved"] --> Grace["72h Grace\nPeriod"]
    Grace -->|Within grace| Full["100% Weight\nRetained"]
    Grace -->|After grace| Decay["Exponential\nDecay Begins"]
    Decay --> Half["50% per 24h\nhalf-life"]
    Half --> Min["Decay to 0.0\nmin multiplier"]
    Min --> Burn["Weight Burns\nto UID 0"]
```

#### Agent Code Storage and Log Consensus

Submitted agent packages (up to 1 MB) are stored on-chain with hash verification. Evaluation logs are proposed by validators and validated through P2P consensus with a majority hash agreement threshold exceeding 50%.

```mermaid
flowchart TB
    Submit["Agent Submission"] --> ValidateSize{"package_zip\n≤ 1MB?"}
    ValidateSize -->|Yes| Store["Blockchain\nStorage"]
    ValidateSize -->|No| Reject["Rejected"]
    Store --> Code["agent_code:hotkey:epoch"]
    Store --> Hash["agent_hash:hotkey:epoch"]
    Store --> Logs["agent_logs:hotkey:epoch\n≤ 256KB"]

    V1["Validator 1"] -->|Log Proposal| P2P["P2P Network"]
    V2["Validator 2"] -->|Log Proposal| P2P
    V3["Validator 3"] -->|Log Proposal| P2P
    P2P --> HashMatch{">50% Hash\nMatch?"}
    HashMatch -->|Yes| Validated["Validated Logs"]
    HashMatch -->|No| Discarded["Rejected"]
```

---

### Bounty Challenge — Bug Discovery and Software Improvement

**Repository:** [PlatformNetwork/bounty-challenge](https://github.com/PlatformNetwork/bounty-challenge)

Bounty Challenge takes a different approach to incentivizing software quality. Rather than competing on agent performance, miners earn rewards by discovering and reporting valid bugs, security vulnerabilities, and improvement suggestions for the Cortex software ecosystem. Issues must be submitted to the bounty-challenge repository, reviewed by project maintainers, and closed with a `valid` label to qualify for rewards.

```mermaid
flowchart LR
    subgraph Miner Actions
        Discover["Discover Bug\nor Improvement"]
        Submit["Submit Issue\nto bounty-challenge repo"]
        Sign["Sign with sr25519\n(Hotkey Verification)"]
    end

    subgraph Validation
        Review["Maintainer Review"]
        Label{"Valid Label\nApplied?"}
        Score["Points Credited\n(1 point per valid issue)"]
        Reject["No Reward"]
    end

    subgraph Weight Calculation
        Net["Net Points =\nvalid + star_bonus − penalty"]
        Weight["Weight =\nnet_points × 0.02"]
        Chain["Submit to\nBittensor Chain"]
    end

    Discover --> Submit --> Sign --> Review --> Label
    Label -->|Yes| Score --> Net --> Weight --> Chain
    Label -->|No| Reject
```

#### Registration and Authentication

Miners register by linking their GitHub username to their Bittensor hotkey via an sr25519 cryptographic signature. The registration is a one-time process that binds on-chain identity to GitHub identity, enabling automatic attribution of issues to miners.

```mermaid
flowchart LR
    CLI["bounty wizard\n(CLI)"] --> Key["Enter Bittensor\nSecret Key"]
    Key --> GitHub["Provide GitHub\nUsername"]
    GitHub --> Sign["sr25519 Signature\nof Username"]
    Sign --> API["POST /register\nvia Bridge API"]
    API --> Verify{"Signature\nValid?"}
    Verify -->|Yes| Store["Store Registration\nin PostgreSQL"]
    Verify -->|No| Reject["Registration\nFailed"]
```

#### Platform Integration Architecture

Bounty Challenge operates as a containerized service within the platform-v2 ecosystem. Validators auto-scan GitHub issues, verify labels, compute weights within a 24-hour rolling window, and submit them to the Bittensor chain.

```mermaid
flowchart LR
    subgraph Miners
        Miner["Miner\n(CLI/wizard)"]
    end

    subgraph Platform["Platform Server\nchain.platform.network"]
        API["API Gateway"]
        DB[("PostgreSQL")]
        Bounty["Bounty Challenge\n(container)"]
        API --> DB
        Bounty --> DB
    end

    subgraph GitHub["GitHub"]
        Issues["bounty-challenge\nIssue Tracker"]
    end

    Miner -->|"register/status"| API
    API -->|"route"| Bounty
    Bounty -->|"scan issues"| Issues
    Miner -->|"create issues"| Issues
```

#### Weight Calculation Formula

The weight assigned to each miner operates within a 24-hour rolling window. The penalty is computed independently for invalid and duplicate issues. Each miner's valid issue count acts as a separate shield for each penalty type:

$$
p_i = \underbrace{\max(0,\; n_i - v_i)}_{\text{invalid penalty}} + \underbrace{\max(0,\; d_i - v_i)}_{\text{duplicate penalty}}
$$

$$
\pi_i = v_i + b_i - p_i \quad\text{where}\quad b_i = \begin{cases} 0.25 \times |\text{starred repos}| & \text{if } v_i \geq 2 \\ 0 & \text{otherwise} \end{cases}
$$

$$
w_i = \max(0,\; \pi_i) \times 0.02
$$

The star bonus $b_i$ caps at 1.25 points (5 repositories at 0.25 each) and requires a minimum of 2 valid issues to activate. Anti-abuse mechanisms include label protection via GitHub Actions, first-reporter-wins deduplication, and maintainer gatekeeping.

---

### Data Fabrication Challenge — Synthetic Coding Dataset Production (March 2026)

The third challenge, planned for launch in **March 2026**, incentivizes miners to produce the best harnesses for fabricating synthetic coding datasets. Miners compete to build and submit data generation pipelines (harnesses) that autonomously produce high-quality training data specialized in code understanding, generation, and reasoning tasks. Validators execute these harnesses in sandboxed environments and evaluate the quality, diversity, and correctness of the resulting synthetic datasets through automated metrics and cross-validation. The better the harness performs at generating useful coding data, the higher the miner's weight and TAO earnings.

The datasets produced through this challenge serve a strategic purpose: they become the training fuel for fine-tuning language models through the GRAIL protocol, creating a direct pipeline from decentralized data production to verifiable model improvement.

#### Data Fabrication Pipeline

```mermaid
flowchart TB
    subgraph MinerSide["Miner"]
        Build["Build Data\nFabrication Harness"]
        Package["Package Harness\n+ Metadata"]
        Submit["Submit to\nValidator Network"]
    end

    subgraph ValidatorSide["Validator Network"]
        Receive["Receive Harness\nSubmission"]
        Execute["Execute Harness\nin Sandbox"]
        Output["Synthetic Coding\nDataset Output"]
        Correctness["Correctness Check\n(syntax, executability)"]
        Diversity["Diversity Analysis\n(language coverage)"]
        Utility["Utility Scoring\n(held-out validation)"]
        Aggregate["Aggregate\nQuality Score"]
    end

    subgraph Downstream["Downstream"]
        Store["Store Validated\nDatasets"]
        GRAIL["Feed to GRAIL\n(Subnet 81)"]
    end

    Build --> Package --> Submit --> Receive
    Receive --> Execute --> Output
    Output --> Correctness
    Output --> Diversity
    Output --> Utility
    Correctness --> Aggregate
    Diversity --> Aggregate
    Utility --> Aggregate
    Aggregate --> Store --> GRAIL
```

#### Quality Scoring

The composite quality score for each harness submission combines three orthogonal metrics. Let $\mathcal{D}_i$ be the dataset produced by miner $i$'s harness:

$$
Q_i = \alpha \cdot C(\mathcal{D}_i) + \beta \cdot D(\mathcal{D}_i) + \gamma \cdot U(\mathcal{D}_i)
$$

where:

$$
C(\mathcal{D}_i) \in [0, 1] \quad \text{(correctness: syntactic validity, executability, annotation accuracy)}
$$

$$
D(\mathcal{D}_i) \in [0, 1] \quad \text{(diversity: language coverage, paradigm variety, difficulty spread)}
$$

$$
U(\mathcal{D}_i) \in [0, 1] \quad \text{(utility: downstream training signal, measured via held-out validation)}
$$

The weighting coefficients satisfy $\alpha + \beta + \gamma = 1$ and are tunable by the subnet owner to steer miner incentives toward the most needed dataset characteristics at any given time.

---

## Unified Scoring Framework

All three challenges share a common weight normalization step before submission to the Bittensor chain. Let $w_i^{(c)}$ denote the raw weight of miner $i$ in challenge $c$, and let $\epsilon_c$ denote the emission allocation for challenge $c$ (configured by the subnet owner such that $\sum_c \epsilon_c = 1$). The final normalized weight for miner $i$ across the entire subnet is:

$$
W_i = \sum_{c \in \mathcal{C}} \epsilon_c \cdot \frac{w_i^{(c)}}{\sum_{j} w_j^{(c)}}
$$

where $\mathcal{C} = \{\text{Term}, \text{Bounty}, \text{DataFab}\}$. This formulation allows the subnet owner to dynamically rebalance incentives across challenges. For instance, setting $\epsilon_{\text{Term}} = 0.5$, $\epsilon_{\text{Bounty}} = 0.2$, $\epsilon_{\text{DataFab}} = 0.3$ allocates half of emissions to the agent harness competition while directing 30% toward dataset production.

The effective weight for Term Challenge incorporates the decay multiplier $\mu(t_i)$, meaning that $w_i^{(\text{Term})} = s_i \cdot \mu(t_i)$ where $s_i$ is the raw evaluation score. For Bounty Challenge, $w_i^{(\text{Bounty})} = \max(0, \pi_i) \times 0.02$ where $\pi_i$ is the net point value. For Data Fabrication, $w_i^{(\text{DataFab})} = Q_i$ the composite quality score.

---

## GRAIL Integration — Verifiable Post-Training

The ultimate destination for the datasets produced by Platform Network's Data Fabrication Challenge is the [GRAIL protocol](https://github.com/one-covenant/grail) (*Guaranteed Rollout Authenticity via Inference Ledger*), operated independently as **Subnet 81** on Bittensor by the One Covenant team. GRAIL delivers post-training for language models with cryptographically verifiable inference, ensuring that rollouts produced during reinforcement learning are tied to a specific model and input and can be independently verified by validators. Platform Network (Subnet 100) intends to leverage GRAIL's infrastructure as a downstream consumer: the high-quality coding datasets produced by our miners feed into GRAIL's verifiable training pipeline, enabling the production of fine-tuned models without Platform Network needing to operate its own training infrastructure.

```mermaid
flowchart LR
    subgraph SN100["Subnet 100 — Platform Network"]
        DFC["Data Fabrication\nChallenge"]
        Miners100["Miners produce\ncoding datasets"]
    end

    subgraph GRAIL_Pipeline["GRAIL Protocol — Subnet 81 (External)"]
        Rollout["Rollout Generation\n(GRPO-style)"]
        Verify["GRAIL Verification\n(PRF commitments)"]
        Train["Post-Training\n(RL fine-tuning)"]
    end

    subgraph Output["Improved Models"]
        Model["Fine-tuned\nLanguage Model"]
        CortexCLI["Cortex CLI\nEnhanced Agent"]
        CortexIDE["Cortex IDE\nEnhanced Agent"]
    end

    Miners100 -->|High-quality datasets| DFC
    DFC -->|Validated datasets| Rollout
    Rollout -->|Signed rollouts| Verify
    Verify -->|Verified rollouts| Train
    Train -->|Updated weights| Model
    Model --> CortexCLI
    Model --> CortexIDE
```

The GRAIL protocol operates through a prover/verifier system. During rollout generation, miners produce multiple GRPO-style rollouts while tracking token IDs and log-probabilities for proof construction. The protocol binds each rollout to the generating model through PRF-based index derivation and sketch commitments. Formally, for a rollout $r$ produced by model $M$ on input $x$, the GRAIL commitment is:

$$
\text{Commit}(r, M, x) = \text{PRF}_k\big(\text{sketch}(r) \;\|\; \text{hash}(M) \;\|\; \text{hash}(x)\big)
$$

where $k$ is derived from the verifier-supplied challenge (combining drand randomness with the window's block hash). This ensures no substitution or replay attacks are possible.

The pipeline from Platform Network to GRAIL creates a cross-subnet collaboration: miners on Subnet 100 produce the raw material (high-quality coding datasets), and GRAIL on Subnet 81 provides the verifiable post-training infrastructure that transforms that material into measurably better models. These fine-tuned models then flow back into the Cortex software products, closing the loop between data production and model improvement.

---

## Tokenomics and Monetization Flywheel

Platform Network's economic model creates a self-sustaining cycle connecting software development, market adoption, and network incentives. The flywheel operates through four interconnected stages.

```mermaid
flowchart TB
    Dev["1. Software Development\n(Cortex CLI + IDE)"]
    Market["2. Marketing\n& Community Growth"]
    Customers["3. Customers\n(Subscriptions & Licenses)"]
    Revenue["4. Revenue Generation"]

    Dev -->|Superior AI tools| Market
    Market -->|User acquisition| Customers
    Customers -->|Subscription fees| Revenue

    Revenue -->|Reinvestment| Buyback["TAO Buyback\n(Token Demand)"]
    Revenue -->|Reinvestment| MarketReinvest["Marketing Budget\nExpansion"]
    Revenue -->|Reinvestment| DevReinvest["Development\nAcceleration"]

    Buyback -->|Increased token value| Emissions["Higher TAO\nEmission Value"]
    Emissions -->|Stronger incentives| Miners["Miner Participation\n& Competition"]
    Miners -->|Better harnesses,\nmore bugs found,\nbetter datasets| Dev

    MarketReinvest -->|Broader reach| Market
    DevReinvest -->|More features| Dev
```

**Stage 1: Software Development.** The challenges hosted on Platform Network (Term Challenge, Bounty Challenge, Data Fabrication Challenge) directly improve the quality of Cortex CLI and Cortex IDE. Miners competing to produce the best agent harnesses drive performance improvements. Bug bounty participants eliminate defects and security vulnerabilities. Dataset producers create the training data that fine-tunes the underlying language models.

**Stage 2: Marketing.** Superior software products create organic traction. The competitive nature of the challenges generates community engagement and visibility within the Bittensor ecosystem and the broader developer tools market.

**Stage 3: Customers.** As Cortex CLI and Cortex IDE reach production quality (estimated end of March 2026), the monetization layer activates through subscriptions and enterprise licenses. The tools' quality, continuously improved by network participants, directly determines conversion and retention.

**Stage 4: Revenue and Reinvestment.** Generated revenue flows into three channels. A portion is allocated to **TAO buyback**, creating demand pressure on the token and increasing the value of emissions received by miners. A portion funds **marketing expansion**, accelerating user acquisition. The remainder funds **development acceleration**, enabling the subnet owner to launch new challenges and expand the software ecosystem.

The buyback mechanism is particularly important for token economics. Let $R$ denote monthly revenue, $\phi_b$ the buyback allocation ratio, and $P_{TAO}$ the market price of TAO. The monthly buyback volume $V_b$ is:

$$
V_b = \frac{R \cdot \phi_b}{P_{TAO}}
$$

This creates a positive feedback loop: as software quality improves through network incentives, revenue grows, buyback volume increases, token value appreciates, emission value rises, and miner incentives strengthen — attracting more competitive participants and further improving software quality.

---

## Roadmap

```mermaid
gantt
    title Platform Network Development Timeline
    dateFormat YYYY-MM-DD
    axisFormat %b %d

    section Software Products
    Cortex CLI Launch                    :milestone, cli, 2026-01-18, 0d
    Cortex IDE Launch                    :milestone, ide, 2026-02-20, 0d
    CLI + IDE Production Release         :milestone, prod, 2026-03-31, 0d

    section Platform Infrastructure
    Platform v2 Release                  :milestone, pv2, 2026-02-21, 0d
    Term Challenge v2 Release            :milestone, tc2, 2026-02-21, 0d
    Bounty Challenge v2                  :active, bc2, 2026-02-21, 14d

    section Upcoming Challenges
    Data Fabrication Challenge           :dc, 2026-03-01, 30d

    section Model Training
    GRAIL Integration Pipeline           :grail, 2026-04-01, 60d
    First Fine-tuned Model Release       :milestone, model, 2026-05-31, 0d
```

The Cortex CLI was released on January 18, 2026, establishing the foundational command-line agent for the ecosystem. The Cortex IDE followed on February 20, 2026, providing a full graphical development environment. On February 21, 2026, Platform v2 and Term Challenge v2 launch together, marking the transition to a fully decentralized WASM-only validator architecture. Bounty Challenge v2 deploys in the same period with updated scoring mechanics and PostgreSQL-backed storage.

In March 2026, the Data Fabrication Challenge goes live, opening a new competitive frontier where miners produce coding datasets optimized for model training. Both Cortex CLI and Cortex IDE are targeted for production release at the end of March 2026, marking their readiness for commercial deployment.

Starting Q2 2026, the GRAIL integration pipeline becomes operational. Validated datasets from the Data Fabrication Challenge flow into the GRAIL verifiable post-training system on Subnet 81, producing the first fine-tuned language models purpose-built for the Cortex ecosystem.

---

## Repository Index

| Repository | Description |
|---|---|
| [PlatformNetwork/platform-v2](https://github.com/PlatformNetwork/platform-v2) | WASM-only P2P validator network for distributed evaluation on Bittensor |
| [PlatformNetwork/term-challenge-v2](https://github.com/PlatformNetwork/term-challenge-v2) | Terminal benchmark challenge — WASM evaluation module for agent harness competition |
| [PlatformNetwork/bounty-challenge](https://github.com/PlatformNetwork/bounty-challenge) | GitHub issue reward system for bug discovery and software improvement |
| [CortexLM/cortex](https://github.com/CortexLM/cortex) | Agent-native development CLI written in Rust |
| [CortexLM/cortex-ide](https://github.com/CortexLM/cortex-ide) | AI-powered IDE built with Tauri v2 (SolidJS + Rust) |
| [one-covenant/grail](https://github.com/one-covenant/grail) | Verifiable post-training for LLMs via the GRAIL protocol (Subnet 81, external team) |

---

## License

MIT
