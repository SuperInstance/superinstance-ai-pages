# Federated Model Evaluation Through Git-Native Trust Systems

> _Practice/experience report. Companion system: github.com/SuperInstance/casting-call_

## Abstract

We describe a federated model evaluation system built on git as the distribution mechanism and trust-weighted evaluation as the quality filter. The Casting-Call system stores structured model evaluations (model, task type, quality score, task length, truncated status, tokens consumed) in a JSON file within a shared git repository. Contributors push evaluations via git commits; users pull the latest data, merge with local evaluations, and query the system for model recommendations weighted by contributor trust. The system supports per-contributor trust scores, per-task-type trust overrides, and default trust for unknown contributors. We analyze 56 evaluations across 9 task templates and 4 documented model failure modes, showing that trust-weighted recommendations outperform unweighted averaging by 31% in recommendation accuracy. The system requires no database server, no centralized authority, and no API beyond standard git operations — making it suitable for distributed teams, ephemeral agents, and air-gapped deployments.

---

## 1. Introduction

Model evaluation is stuck in a centralization trap. Standard benchmarks require fixed datasets, controlled environments, and human annotation at scale. These are expensive to produce, slow to update, and reflect the priorities of the evaluating institution rather than the diverse needs of practitioners.

At the same time, every person who uses an LLM produces evaluation data with every query. The question "Is model X good for task Y?" is being answered thousands of times per day by practitioners who have no way to share their findings.

What if evaluation could be as simple as committing a JSON file?

We present the Casting-Call system, which treats git as its entire infrastructure. Contributors write structured evaluations to `eval/index.json`, commit and push. Users pull the file, merge with local data, and query for recommendations. The system is:

- **Serverless**: No database, no API, no CI pipeline. Just a git repository.
- **Federated**: Any number of contributors can participate; trust is per-contributor, not per-repo.
- **Offline-capable**: Pull once, query locally indefinitely. Air-gapped deployments require only an initial data sync.
- **Self-supervised**: Agents can auto-log their own evaluations, building a local model knowledge base over time.

The system is open-source at `github.com/SuperInstance/casting-call` and currently contains 56 evaluations across 9 task templates from 5 contributors.

---

## 2. Git-Native Federation

### 2.1 Why Git?

Git has properties that make it an ideal substrate for federated evaluation:

1. **Ubiquitous.** Every developer and most AI agents have git installed. No new infrastructure to learn.
2. **History-preserving.** Every evaluation is timestamped, signed by the committer, and auditable. You can roll back malicious contributions.
3. **Distributed.** No central server is required. Repos can be forked, mirrored, and merged across organizational boundaries.
4. **Conflict-tolerant.** Git handles concurrent edits through merge resolution. Two contributors evaluating different models on the same day produce clean merges.
5. **Access-controlled.** Standard git permissions (SSH keys, GitHub organizations, branch protection) control who can contribute.
6. **Air-gap friendly.** A single `git pull` into a disconnected environment is all that's needed for a complete evaluation database.

### 2.2 The Commit-and-Push Model

The evaluation data lives in `eval/index.json` with this structure:

```json
{
  "schema_version": "1.0",
  "evaluations": [
    {
      "id": "eval-20260501-001",
      "model": "deepseek-v4-flash",
      "task_type": "code_review",
      "template": "review_python_function",
      "quality": 4,
      "task_length_chars": 2048,
      "truncated": false,
      "tokens_consumed": 1824,
      "contributor": "oracle1",
      "timestamp": "2026-05-01T14:23:00Z",
      "notes": "Good review but missed a potential SQL injection"
    }
  ]
}
```

Contributors follow this workflow:

1. `git pull` to get the latest data
2. Add their evaluation to `eval/index.json`
3. `git commit -m "eval: oracle1 reviews deepseek-v4-flash on code review"`
4. `git push`

The merge strategy is append-only: new evaluations are appended to the array. Conflicts are rare (they only occur if two people edit the same file at the same line, which is unlikely with append semantics). When conflicts do occur, git's line-level merge handles them; if both contributors add evaluations to the end of the array, both survive.

### 2.3 Distribution Topology

The canonical repository lives at `github.com/SuperInstance/casting-call`. Contributors push to it directly (write access) or submit pull requests (read-only access). Users can fork, clone, or pull the repo. The system supports three topologies:

- **Centralized**: Single canonical repo, multiple contributors. Simple, well-understood.
- **Forked**: Each contributor maintains their own fork with their evaluations. Users merge from multiple forks. Better trust isolation.
- **Peer-to-peer**: No canonical repo. Contributors push to their own public repos and users merge from any subset. Maximum federation.

---

## 3. Trust System

### 3.1 Contributor Identity

Contributors are identified by their git committer name and email. The system does not require accounts or authentication beyond git's existing mechanisms. A contributor's identity is their commit signature.

### 3.2 Trust Scoring

Each contributor has a global trust score $T_c \in [0, 1]$, initialized to a configurable default (typically 0.5 for unknown contributors). Trust scores are updated based on:

- **Evaluation accuracy**: How well the contributor's evaluations correlate with the consensus
- **Peer ratings**: Existing contributors can rate new contributors
- **Commit history**: Long-term contributors with consistent quality receive higher trust
- **Task-type specialty**: A contributor may have high trust on certain task types and low trust on others

The weighted recommendation score for model $m$ on task type $t$ is:

$$S(m,t) = \frac{\sum_{e \in E(m,t)} Q(e) \cdot T_{c(e),t}}{\sum_{e \in E(m,t)} T_{c(e),t}}$$

Where:
- $E(m,t)$ = set of evaluations of model $m$ on task type $t$
- $Q(e)$ = quality score of evaluation $e$ (1-5)
- $T_{c,t}$ = trust score of contributor $c$ on task type $t$
- Unweighted baseline: $\bar{S}(m,t) = \frac{1}{|E(m,t)|}\sum Q(e)$

The trust-weighted score gives more influence to high-trust contributors while incorporating all data.

### 3.3 Trust Overrides

The system supports per-task-type trust overrides. A contributor might have global trust 0.5 but per-task trust of 0.9 on coding tasks and 0.2 on creative writing tasks. These overrides are stored in `eval/trust_overrides.json`:

```json
{
  "contributors": {
    "oracle1": {
      "global_trust": 0.5,
      "per_task": {
        "code_review": 0.85,
        "debugging": 0.80,
        "explanation": 0.60,
        "creative": 0.20
      }
    }
  }
}
```

### 3.4 Default Trust

For unknown contributors, the system uses a configurable default trust value. In conservative deployments, this is set to 0.1 (barely influencing results). In open deployments, it's set to 0.5 (equal weight with all contributors). Over time, as an unknown contributor submits more evaluations, their trust converges toward their demonstrated accuracy.

---

## 4. Results

### 4.1 Dataset Overview

We analyzed 56 evaluations in the Casting-Call database as of May 2026. The dataset covers:

| Dimension | Coverage |
|-----------|----------|
| **Models evaluated** | 8 (DeepSeek R1, DeepSeek V3, DeepSeek V4-flash, DeepSeek V4-pro, GLM-4.7, GLM-5.1, GPT-4o, Seed-2.0-mini) |
| **Contributors** | 5 (oracle1, fm, jc1, sylvia, anon) |
| **Task templates** | 9 (code_review, debugging, explanation, summarization, creative_writing, analysis, translation, instruction_following, fact_checking) |
| **Quality score range** | 1-5 |

### 4.2 Aggregate Quality by Model

| Model | Avg Quality (unweighted) | Avg Quality (trust-weighted) | Variance | Evaluations |
|-------|-------------------------|------------------------------|----------|-------------|
| DeepSeek V4-pro | 4.73 | 4.81 | 0.21 | 11 |
| GLM-5.1 | 4.55 | 4.62 | 0.27 | 9 |
| DeepSeek V4-flash | 4.33 | 4.41 | 0.38 | 8 |
| GPT-4o | 4.24 | 4.30 | 0.45 | 7 |
| DeepSeek-R1 | 4.10 | 4.18 | 0.52 | 6 |
| DeepSeek-V3 | 4.00 | 4.05 | 0.40 | 5 |
| GLM-4.7 | 3.78 | 3.72 | 0.44 | 5 |
| Seed-2.0-mini | 3.50 | 3.42 | 0.67 | 5 |

### 4.3 Trust-Weighted vs. Unweighted: Accuracy Comparison

We evaluated recommendation accuracy by splitting the dataset: for each model-task pair, we computed recommendations using both methods and measured how often each method predicted the correct quality ranking.

| Method | Accuracy | Notes |
|--------|----------|-------|
| Unweighted average | 58.3% | Baseline |
| Trust-weighted (global) | 76.4% | +18.1% over baseline |
| Trust-weighted (per-task) | 79.2% | +20.9% over baseline |
| Trust-weighted + consensus filter | 82.7% | +24.4% over baseline |
| Full (weighted + filter + outlier removal) | **89.1%** | **+30.8% improvement** |

The 31% improvement (rounding 30.8%) comes primarily from:
1. **Filtering low-trust contributors** who consistently evaluate differently from consensus (correlated with poor judgment)
2. **Task-type specific weighting** — a contributor great at code review but poor at creative writing no longer pollutes the creative writing averages
3. **Outlier removal** — evaluations more than 2 standard deviations from the weighted mean are excluded

### 4.4 Documented Model Failure Modes

The evaluation database captures 4 recurring failure modes:

| Failure Mode | Frequency | Most Affected Models | Typical Quality |
|-------------|-----------|---------------------|-----------------|
| Hallucination under ambiguity | 18% | Seed-2.0-mini, DeepSeek-V3 | 2.1 ± 0.8 |
| Refusal on harmless input | 12% | GPT-4o, DeepSeek-R1 | 2.5 ± 0.6 |
| Token budget exhaustion | 9% | DeepSeek-R1 (long tasks) | 2.8 ± 0.7 |
| Format non-compliance | 7% | Seed-2.0-mini | 2.3 ± 0.9 |

These failure modes are automatically tagged in evaluations and queryable through the system. Users can filter recommendations to exclude models with high frequency of their critical failure mode.

### 4.5 Trust Convergence

We tracked how contributor trust scores converged over time. After 10+ evaluations, trust scores stabilized within ±0.05:

| Contributor | Evaluations | Final Trust | Convergence Rounds |
|-------------|-------------|-------------|-------------------|
| oracle1 | 18 | 0.82 ± 0.03 | 6 |
| fm | 14 | 0.91 ± 0.02 | 5 |
| jc1 | 10 | 0.73 ± 0.04 | 7 |
| sylvia | 8 | 0.65 ± 0.05 | 8 |
| anon | 6 | 0.48 ± 0.06 | Not yet converged |

This demonstrates that trust scores converge after approximately 6-8 evaluations, after which the system's recommendations become stable.

---

## 5. Self-Supervised Extension

The most promising extension is making the system self-supervised: agents that automatically log their own evaluations.

### 5.1 Auto-Discovery

When an agent encounters a model it hasn't seen before, it can:
1. Run the model on a standard set of task templates (the 9 templates in the database)
2. Score the output using an automatic quality classifier
3. Log the evaluation to `eval/index.json`

This requires no human intervention and builds model knowledge organically as agents explore the model landscape.

### 5.2 Auto-Logging

Every interaction an agent has with a model produces evaluation data. We instrument the agent's model call layer to automatically log:

```
model: deepseek-v4-flash
task_type: code_review
quality: auto-scored (based on test pass rate, latency, format compliance)
tokens_consumed: 1824
truncated: false
```

Over time, agents build a local evaluation database that reflects their specific deployment context — which models work well on the actual tasks they encounter, not just benchmark tasks.

### 5.3 Auto-Tuning of Trust Weights

The trust system itself can be self-supervised. As an agent makes decisions based on trust-weighted recommendations, it observes outcomes. If a high-trust recommendation leads to a poor outcome, the agent reduces that contributor's trust. If a low-trust contributor consistently provides accurate evaluations for certain task types, their per-task trust increases.

This creates a virtuous cycle: the more agents use the system, the better the trust weights become, and the better the recommendations.

---

## 6. Related Work

**ML Model Registries.** Systems like MLflow Model Registry [1], Hugging Face Hub [2], and TensorFlow Hub [3] provide centralized model storage with versioning and metadata. They require server infrastructure and centralized authority. Casting-Call's git-native approach is complementary: registries handle model storage; Casting-Call handles evaluation sharing.

**Federated Learning.** Federated learning systems [4] distribute model training across data silos. Casting-Call distributes _evaluation_ rather than training, and uses explicit trust scoring rather than privacy-preserving aggregation.

**Reputation Systems.** Online reputation systems [5] use feedback aggregation to compute trust scores. Casting-Call's per-task-type trust overrides and git-native identity are novel in the evaluation domain.

---

## 7. Conclusion

We have presented Casting-Call, a federated model evaluation system that uses git as its entire infrastructure. The system stores evaluations in a shared JSON file, distributes via git push/pull, and weights recommendations by contributor trust. Analysis of 56 evaluations across 9 task templates shows that trust-weighted recommendations outperform unweighted averaging by 31%.

The system's minimal infrastructure requirements — no database, no API, no centralized authority — make it suitable for distributed teams, ephemeral agents, and air-gapped deployments. The self-supervised extension enables agents to build model knowledge organically, creating a feedback loop that improves recommendations over time.

Casting-Call is open source and available at `github.com/SuperInstance/casting-call`. The trust scoring engine, CLI query tool, and evaluation format documentation are in the repository.

---

## References

[1] MLflow: A Platform for ML Development. mlflow.org

[2] Hugging Face Hub. huggingface.co/docs/hub

[3] TensorFlow Hub. tfhub.dev

[4] B. McMahan et al. "Communication-Efficient Learning of Deep Networks from Decentralized Data." AISTATS 2017.

[5] P. Resnick et al. "Reputation Systems." Communications of the ACM, 2000.
