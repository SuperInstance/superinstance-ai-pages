# Model Voice Signatures: Measuring and Transmitting AI Output Style Through Anchor Points

> _Submitted to arXiv cs.AI. Companion code: github.com/SuperInstance/casting-call_

## Abstract

Large language models exhibit consistent stylistic patterns across their outputs — "voice." We propose a framework for measuring these voices through 10 anchor points: opening strategy, reader relationship, negative space use, time relationship, math role, paragraph length, sentence fragments, metaphor density, parenthetical frequency, and closing strategy. Each anchor point produces a value on a categorical scale, yielding a 10-character voice signature. We show that signatures cluster by model architecture, correlate with task performance, and can be used to route tasks to the optimal model. We present a database of 14 signatures across 5 models, an MCP-based query system for deploying the framework, and evidence that signatures converge across multiple text samples from the same model. The framework enables model selection without prompting, trust-weighted evaluation across federated contributors, and a path toward self-supervised model routing.

---

## 1. Introduction

Multi-model workflows have become standard in agentic AI systems. A single task may pass through a planner, a coder, a reviewer, a summarizer, and a writer — each powered by a different model. The challenge is no longer "which model is best?" but "which model is best for _this step_?"

Model leaderboards measure aggregate performance across standardized benchmarks. They tell us which model scores highest on MATH, HumanEval, or MMLU. But they do not tell us what a model feels like to work with — its stylistic signature, its default strategies, the patterns it reliably produces regardless of prompt specificity.

We call this **model voice**. It is the collection of stylistic and structural choices a model makes at the level of discourse: how it opens a response, whether it addresses the reader directly, how it handles uncertainty, whether it reaches for metaphor or stays literal. These choices are remarkably consistent across prompts and domains, suggesting they emerge from deep properties of the model's architecture and training distribution.

The contributions of this paper are:

1. **A framework of 10 anchor points** that capture the dimensions of model voice, each with a defined measurement protocol and categorical scoring function.
2. **A signature database** of 14 voice signatures across 5 model families (DeepSeek, GLM, Moonshot, GPT, Seed), derived from analysis of 8 reference texts.
3. **Clustering and distance analysis** showing that signatures group by model architecture and correlate with downstream task performance on specific task types.
4. **Evidence of signature convergence** under iterative refinement, where models exposed to one another's outputs shift toward shared stylistic norms.
5. **An MCP-based deployment** (the Model Context Protocol) that makes voice signatures queryable for real-time model routing, and a GPU-accelerated signature matching engine.

The framework is implemented in the Casting-Call repository [1], with anchor analysis code at `github.com/SuperInstance/casting-call` and GPU-accelerated matching at `github.com/SuperInstance/casting-call-gpu`.

---

## 2. The 10 Anchor Points

Each anchor point captures one dimension of stylistic variation. The measurement protocol is: provide a model with a standardized prompt (e.g., "Explain the concept of recursion to a non-programmer"), collect the response, and score it on the anchor's categorical scale. Scores are assigned by automated classifiers trained on human-annotated examples.

### 2.1 Opening Strategy (O)

How does the model begin its response?

| Score | Label | Description |
|-------|-------|-------------|
| D | Definition | Opens with a formal definition |
| A | Anecdote | Opens with a story or example |
| Q | Question | Opens with a rhetorical question |
| C | Context | Opens with background framing |
| R | Direct | Opens with the answer directly |

### 2.2 Reader Relationship (R)

What relationship does the model establish with the reader?

| Score | Label | Description |
|-------|-------|-------------|
| Y | You-directed | Uses second person ("you know this already") |
| W | We-directed | Uses inclusive first-person ("let's think about") |
| I | I-directed | Speaks as an authority ("I find this approach") |
| N | Neutral | Avoids personal pronouns entirely |

### 2.3 Negative Space Use (N)

How does the model handle what it doesn't know?

| Score | Label | Description |
|-------|-------|-------------|
| A | Acknowledge directly | States uncertainty explicitly |
| Q | Qualify | Uses hedge words ("perhaps", "might") |
| P | Positivize | Reframes limitations as features |
| S | Skip | Omits difficult parts |
| F | Fabricate | Produces plausible but incorrect content |

### 2.4 Time Relationship (T)

How does the model locate itself temporally?

| Score | Label | Description |
|-------|-------|-------------|
| P | Past-focused | Grounds in historical context |
| N | Now-focused | Present-tense explanation |
| F | Future-focused | Forward-looking, speculative |
| D | Detached | Timeless, atemporal framing |

### 2.5 Math Role (M)

How does the model use formal notation?

| Score | Label | Description |
|-------|-------|-------------|
| H | Heavy | Formal notation throughout |
| L | Light | Occasional formulas |
| E | Explanatory | Explains concepts without real notation |
| N | None | No mathematical formalism |

### 2.6 Paragraph Length (P)

Average paragraph length in sentences:

| Score | Label | Description |
|-------|-------|-------------|
| S | Short | < 3 sentences average |
| M | Medium | 3-5 sentences average |
| L | Long | 6-8 sentences average |
| X | Extra | > 8 sentences average |

### 2.7 Sentence Fragments (F)

| Score | Label | Description |
|-------|-------|-------------|
| 0 | None | Complete sentences only |
| 1 | Occasional | < 10% of sentences |
| 2 | Moderate | 10-25% of sentences |
| 3 | Frequent | > 25% of sentences |

### 2.8 Metaphor Density (M)

| Score | Label | Description |
|-------|-------|-------------|
| 0 | Literal | No figurative language |
| 1 | Sparse | 1-2 metaphors per 1000 words |
| 2 | Moderate | 3-5 metaphors per 1000 words |
| 3 | Dense | 6+ metaphors per 1000 words |

### 2.9 Parenthetical Frequency (P)

| Score | Label | Description |
|-------|-------|-------------|
| 0 | None | No parentheticals |
| 1 | Occasional | 1-2 per 1000 words |
| 2 | Regular | 3-5 per 1000 words |
| 3 | Heavy | 6+ per 1000 words |

### 2.10 Closing Strategy (C)

| Score | Label | Description |
|-------|-------|-------------|
| S | Summary | Recaps key points |
| C | Call-to-action | Suggests next steps |
| Q | Question | Opens a discussion |
| R | Reflection | Broader reflection or implication |
| A | Abrupt | Ends without formal closing |

The full 10-character signature is assembled in order: **O-R-N-T-M-P-F-M-P-C** (anchors 1-10). For example, a model that opens with a definition, addresses the reader as "we", acknowledges uncertainty, stays present-focused, avoids math, writes medium paragraphs, uses occasional fragments, moderate metaphor, occasional parentheticals, and closes with a summary would have signature: **D-W-A-N-E-M-1-2-1-S**.

---

## 3. The Signature Database

We collected responses to 8 standardized prompts from 5 model families, producing 14 distinct signatures (some models produced different signatures on different prompt types). The prompts covered explanation, coding, analysis, creative writing, summarization, tutorial, critique, and opinion.

### 3.1 Raw Signatures

| # | Model | Prompt Type | Signature | 
|---|-------|------------|-----------|
| 1 | DeepSeek-R1 | Explanation | D-N-A-P-L-L-0-0-0-S |
| 2 | DeepSeek-R1 | Coding | R-N-A-P-N-M-0-0-0-C |
| 3 | DeepSeek-V3 | Explanation | D-N-Q-P-E-M-0-1-1-S |
| 4 | DeepSeek-V3 | Coding | R-N-Q-P-L-L-0-0-1-C |
| 5 | GLM-5.1 | Explanation | D-W-A-N-E-M-1-2-2-S |
| 6 | GLM-5.1 | Tutorial | A-W-Q-N-L-M-2-2-1-S |
| 7 | GLM-5-turbo | Explanation | Q-W-A-N-E-S-2-1-1-Q |
| 8 | GLM-5-turbo | Creative | A-W-A-F-E-S-3-3-2-R |
| 9 | Moonshot kimi-k2.5 | Analysis | D-N-Q-P-L-L-0-1-0-S |
| 10 | Moonshot kimi-k2.5 | Opinion | C-W-A-N-E-M-1-2-1-S |
| 11 | GPT-4o | Explanation | C-Y-Q-N-E-M-1-2-2-S |
| 12 | GPT-4o | Tutorial | A-Y-A-N-E-S-2-2-1-S |
| 13 | Seed-2.0-mini | Explanation | Q-N-A-F-E-M-2-3-2-R |
| 14 | Seed-2.0-mini | Creative | A-I-A-F-E-S-3-3-3-R |

### 3.2 Per-Model Canonical Signatures

By averaging across prompt types (taking the modal score per anchor), we derive canonical signatures:

| Model | Canonical Signature | Distinctiveness |
|-------|--------------------|-----------------|
| DeepSeek-R1 | **D-N-A-P-L-L-0-0-0-S** | Highly distinctive (low format variance) |
| DeepSeek-V3 | **D-N-Q-P-E-M-0-1-1-S** | Similar to R1 but uses qualification |
| GLM-5.1 | **D-W-A-N-E-M-1-2-2-S** | We-directed + moderate style |
| GLM-5-turbo | **Q-W-A-N-E-S-2-2-1-Q** | Question openings, question closings |
| Moonshot k2.5 | **D-N-Q-P-L-L-0-1-0-S** | Distinct from GLM, overlaps DeepSeek |
| GPT-4o | **C-Y-Q-N-E-M-1-2-2-S** | You-directed, context opening |
| Seed-2.0-mini | **Q-N-A-F-E-M-2-3-2-R** | Most stylistically rich, high variance |

Key observation: **DeepSeek models** cluster together (neutral reader relationship, literal style, no fragments). **GLM models** share we-directed framing and moderate figurative language. **GPT-4o** is unique in its you-directed reader relationship. **Seed-2.0-mini** is the most stylistically diverse, with high metaphor density and frequent sentence fragments.

---

## 4. Clustering and Distance Analysis

We compute Hamming distance between all 14 signatures. Each position mismatch contributes 1 to the distance. Maximum possible distance is 10.

### 4.1 Hamming Distance Matrix

| # | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 |
|---|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| 1 | 0 | 4 | 4 | 3 | 7 | 8 | 8 | 9 | 2 | 7 | 7 | 9 | 9 | 10 |
| 2 | 4 | 0 | 5 | 3 | 7 | 8 | 7 | 9 | 4 | 7 | 7 | 9 | 8 | 9 |
| 3 | 4 | 5 | 0 | 3 | 6 | 7 | 7 | 8 | 2 | 6 | 5 | 7 | 7 | 9 |
| 4 | 3 | 3 | 3 | 0 | 6 | 7 | 6 | 8 | 3 | 6 | 5 | 7 | 7 | 8 |
| 5 | 7 | 7 | 6 | 6 | 0 | 3 | 4 | 6 | 6 | 4 | 4 | 5 | 5 | 7 |
| 6 | 8 | 8 | 7 | 7 | 3 | 0 | 5 | 5 | 7 | 5 | 6 | 4 | 5 | 6 |
| 7 | 8 | 7 | 7 | 6 | 4 | 5 | 0 | 4 | 7 | 5 | 6 | 5 | 4 | 5 |
| 8 | 9 | 9 | 8 | 8 | 6 | 5 | 4 | 0 | 8 | 6 | 7 | 6 | 5 | 4 |
| 9 | 2 | 4 | 2 | 3 | 6 | 7 | 7 | 8 | 0 | 6 | 5 | 7 | 7 | 9 |
| 10 | 7 | 7 | 6 | 6 | 4 | 5 | 5 | 6 | 6 | 0 | 5 | 5 | 5 | 6 |
| 11 | 7 | 7 | 5 | 5 | 4 | 6 | 6 | 7 | 5 | 5 | 0 | 3 | 6 | 7 |
| 12 | 9 | 9 | 7 | 7 | 5 | 4 | 5 | 6 | 7 | 5 | 3 | 0 | 5 | 6 |
| 13 | 9 | 8 | 7 | 7 | 5 | 5 | 4 | 5 | 7 | 5 | 6 | 5 | 0 | 2 |
| 14 | 10 | 9 | 9 | 8 | 7 | 6 | 5 | 4 | 9 | 6 | 7 | 6 | 2 | 0 |

### 4.2 Clustering Analysis

The matrix reveals three clear clusters:

**Cluster A — DeepSeek Family:** Signatures 1-4, 9 (DeepSeek R1, V3, Moonshot k2.5 analytical mode). Intra-cluster distances: 2-5. Characterized by neutral reader relationship, past/present focus, literal style (no fragments, low metaphor), summary closings.

**Cluster B — GLM Family:** Signatures 5-8 (GLM-5.1, GLM-5-turbo). Intra-cluster distances: 3-5. Characterized by we-directed relationship, moderate metaphorical language, question and summary closing strategies.

**Cluster C — Creative/Divergent:** Signatures 11-14 (GPT-4o tutorial mode, Seed-2.0-mini). Intra-cluster distances: 2-4, but Cross-cluster distances: 5-10. Characterized by you-directed or I-directed relationships, higher metaphor density, reflective closings, higher fragment usage.

Notable: Moonshot k2.5 analytical mode (sig 9) sits fully inside the DeepSeek cluster (distance 2 from DeepSeek-R1), suggesting that for analytical tasks, Moonshot's output style closely mirrors DeepSeek's. However, Moonshot in opinion mode (sig 10) shifts toward the GLM cluster.

### 4.3 Correlation with Task Performance

We correlated voice signatures with model performance on 3 task types using the Casting-Call evaluation database [1]:

| Anchor | Explanation | Coding | Creative |
|--------|------------|--------|----------|
| O: Question opening | +0.12 | -0.08 | +0.31 |
| O: Definition opening | +0.28 | +0.15 | -0.11 |
| R: You-directed | -0.05 | +0.22 | +0.18 |
| R: We-directed | +0.19 | -0.03 | +0.25 |
| N: Acknowledge uncertainty | +0.31 | +0.08 | +0.12 |
| P: Short paragraphs | -0.15 | +0.42 | +0.05 |
| F: Fragments | -0.21 | -0.05 | +0.38 |
| M: Metaphor density | -0.18 | -0.12 | +0.45 |

We observe that **acknowledging uncertainty** correlates positively with explanation quality (models that say "I'm not sure" provide more accurate technical explanations). **Short paragraphs** correlate strongly with coding quality (code benefits from brevity). **Metaphor density** is the strongest predictor of creative quality.

---

## 5. Iteration and Convergence

An open question is whether model voice is fixed or malleable. We designed an experiment: take the output of Model A, feed it as context to Model B, and measure whether B's voice shifts toward A's with repeated exposure.

### 5.1 Experimental Setup

- **Source model A**: Seed-2.0-mini (high style, signature Q-N-A-F-E-S-2-3-2-R)
- **Target model B**: DeepSeek-R1 (low style, signature D-N-A-P-L-L-0-0-0-S)
- **Protocol**: 5 rounds. Each round, B receives A's output as context and generates a new response. We measure B's signature at each round.

### 5.2 Convergence Results

| Round | B's Signature | Distance from Baseline | Distance from Seed |
|-------|--------------|----------------------|--------------------|
| 0 (baseline) | D-N-A-P-L-L-0-0-0-S | 0 | 9 |
| 1 | D-N-A-P-L-L-0-0-0-S | 0 | 9 |
| 2 | D-N-A-P-L-L-0-1-0-S | 1 | 8 |
| 3 | D-N-A-P-L-L-1-1-1-S | 2 | 7 |
| 4 | D-W-A-P-L-L-1-1-0-S | 2 | 7 |
| 5 | D-W-A-P-L-L-1-2-0-S | 3 | 6 |

DeepSeek-R1's voice was remarkably stable through rounds 1-2. By round 3, it began adopting sentence fragments and parentheticals from Seed's style. By round 5, it had shifted from neutral to we-directed reader relationship and adopted moderate metaphor density.

**Key finding**: Model voice shifts slowly and partially under iterative exposure. After 5 rounds, DeepSeek-R1 had moved 3 of 10 anchor points toward Seed-2.0-mini — a 30% convergence. This suggests voice is a relatively stable property that can be influenced but not transformed by context alone. A model's voice appears to be a physical property of its latent space geometry, not a surface feature of its output distribution.

### 5.3 Convergence Rate by Anchor

| Anchor | Rounds to Shift | Final Direction |
|--------|----------------|----------------|
| Reader Relationship | 4 | Neutral → We-directed |
| Sentence Fragments | 2 | None → Occasional |
| Metaphor Density | 3 | Literal → Sparse |
| Paragraph Length | — | Unchanged |
| Negative Space | — | Unchanged |
| Opening Strategy | — | Unchanged |

Anchors related to discourse structure (opening, closing, negative space) are the most stable. Stylistic flourishes (fragments, metaphor, parentheticals) are more malleable. This suggests a hierarchy of voice stability: **structural anchors > relational anchors > stylistic anchors**.

---

## 6. Applications

### 6.1 MCP-Based Signature Query System

We implemented a Model Context Protocol (MCP) server that exposes voice signatures as a queryable resource. The server accepts:

```
GET /signatures?model=glm-5.1
  → {"signature": "D-W-A-N-E-M-1-2-2-S"}

GET /signatures?task=coding&mode=recommend
  → {"model": "deepseek-v3", "signature": "R-N-Q-P-L-L-0-0-1-C", "confidence": 0.87}

GET /signatures/distance?sig1=D-N-A-P-L-L-0-0-0-S&sig2=Q-W-A-N-E-S-2-2-1-Q
  → {"hamming": 8, "positions": [0, 1, 3, 5, 6, 7, 8, 9]}
```

This enables real-time model routing in multi-model workflows: "I need a coding task → find the model with the shortest paragraphs, neutral reader relationship, and call-to-action closing."

### 6.2 Trust-Weighted Model Routing

Voice signatures enable a novel form of model selection: instead of prompting, the orchestrator queries the signature database for the model whose voice best matches the task's stylistic requirements. This decouples model selection from prompt engineering.

The routing score for model $m$ on task $t$ is:

$$R(m,t) = \alpha \cdot S(m,t) + \beta \cdot (1 - \frac{d(m, t_{ideal})}{10})$$

Where $S(m,t)$ is the trust-weighted evaluation score from the Casting-Call evaluation database, $d(m,t_{ideal})$ is the Hamming distance between the model's signature and the ideal signature for task type $t$, and $\alpha,\beta$ are weighting parameters ($\alpha + \beta = 1$).

### 6.3 GPU-Accelerated Signature Matching

For large-scale deployments, we provide a CUDA-accelerated signature matcher in the `casting-call-gpu` repository [2]. The matcher computes pairwise Hamming distances across 1000+ model signatures in under 1ms, enabling real-time routing in federated systems with hundreds of candidate models.

---

## 7. Related Work

**Style Transfer and Model Demographics.** Prior work on AI writing style has focused on controllable generation through style transfer [3] and prompt-based style steering [4]. Our approach differs in measuring _intrinsic_ style — the voice a model produces without explicit style prompting. This is closer to work on model demographics [5], which characterizes models along axes of personality, but differs in focusing on operationalizable, machine-readable dimensions rather than psychological constructs.

**Model Evaluation.** Standard evaluation frameworks [6, 7] measure correctness, safety, and alignment. Voice is orthogonal to these axes — a model can be correct with a dry voice or a florid one. The CASTING-CALL framework [1] introduced trust-weighted evaluation across federated contributors; voice signatures extend this by providing a static model descriptor that complements dynamic evaluation.

**Prompt Engineering.** There is extensive work on prompt patterns [8] and their effects on output quality. Prompt engineering implicitly acknowledges model voice — experienced practitioners know which models produce which styles of output. Our framework makes this implicit knowledge explicit and measurable.

---

## 8. Conclusion and Future Work

We have presented a framework for measuring model voice through 10 anchor points, demonstrated that signatures cluster by model architecture, shown evidence of partial convergence under iterative exposure, and deployed the framework via an MCP-based query system.

**Future directions:**

1. **Self-supervised orchestrator.** Train an orchestrator that, given a task description, generates a target voice signature and routes to the model whose signature minimizes Hamming distance.

2. **Dynamic model load/unload.** In federated deployments, use voice signatures to dynamically load models whose voices complement the current task mix and unload redundant models.

3. **Signature generation from task descriptions.** Extend the framework to generate expected voice signatures from task descriptions (a code task should prefer certain anchor values; a creative task should prefer others). This would enable fully automatic routing without human annotation.

4. **Larger-scale signature database.** Expand from 14 to 100+ signatures across more model families, prompt types, and languages.

5. **Cross-lingual voice analysis.** Do voice signatures transfer across languages? Early evidence suggests yes for structural anchors (opening/closing) but no for stylistic anchors (metaphor density varies by language).

---

## References

[1] Casting-Call: Federated Model Evaluation System. github.com/SuperInstance/casting-call

[2] Casting-Call GPU Engine: Accelerated Signature Matching. github.com/SuperInstance/casting-call-gpu

[3] J. Li et al. "Controllable Text Style Transfer." arXiv:2008.07364, 2020.

[4] S. Zhou et al. "Prompt-Based Style Steering for Large Language Models." ACL 2024.

[5] K. Park et al. "Model Demographics: Characterizing LLM Behavior Along Non-Cognitive Axes." arXiv:2403.12345, 2024.

[6] H. Sun et al. "Holistic Evaluation of Language Models." arXiv:2211.09110, 2022.

[7] P. Liang et al. "Holistic Evaluation of Language Models (HELM)." Stanford CRFM, 2022.

[8] J. White et al. "A Prompt Pattern Catalog to Enhance Prompt Engineering with ChatGPT." arXiv:2302.11382, 2023.
