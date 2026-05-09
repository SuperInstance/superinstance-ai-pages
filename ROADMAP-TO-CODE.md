# SuperInstance Fleet: Roadmap from Papers to Code

> _A practical document connecting every paper to the code that implements it._

---

## 1. CPA White Paper — "Casting Call Protocol Architecture"

### Claims
1. Multi-model workflows need a standardized protocol for calling and routing between models
2. Trust-weighted evaluation improves model selection quality over flat averaging
3. The protocol should support ephemeral agents, persistent services, and federated deployments
4. JSON-based evaluation records are sufficient for federated sharing
5. MCP (Model Context Protocol) provides the transport layer for model calls

### Existing Code

| Repo | Complete | Missing |
|------|----------|---------|
| `github.com/SuperInstance/casting-call` | Core CLI (cast, fetch, merge, trust), evaluation schema, CI workflow, 56 evaluations | CLI needs polish, no npm/pypi package published |
| `github.com/SuperInstance/casting-call-gpu` | CUDA kernel for pairwise Hamming distance | None — complete for current scope |
| `github.com/SuperInstance/plato-sdk` | Base protocol, message types, relay transport | No Casting-Call integration yet |

### What Needs Building

| Component | Effort | Priority | Notes |
|-----------|--------|----------|-------|
| `casting-call` npm/pypi package | 2 days | P0 | Publish to npm and PyPI for agent consumption |
| MCP server for signature queries | 3 days | P0 | Expose /signatures, /recommend, /distance endpoints |
| Trust auto-update engine | 5 days | P1 | Auto-adjust trust based on recommendation outcomes |
| Protocol spec document | 2 days | P1 | Formalize the protocol outside the white paper |
| Multi-repo merge strategy | 4 days | P2 | Merge from N forks with conflict resolution |

### Dependency Chain
- **CPA Paper** → `casting-call` → npm/pypi package (blocks MCP server)
- **CPA Paper** → `casting-call-gpu` → GPU engine (supports MCP server)
- **CPA Paper** → `plato-sdk` → integration layer

---

## 2. Model Voice Signatures — "Measuring AI Output Style Through Anchor Points"

### Claims
1. Model voice can be measured through 10 anchor points
2. Voice signatures cluster by model architecture
3. Signatures correlate with task performance
4. Signatures converge under iterative exposure
5. MCP-queryable signature database enables real-time routing

### Existing Code

| Repo | Complete | Missing |
|------|----------|---------|
| `github.com/SuperInstance/casting-call` | Anchor analyzer code (Python), signature database (14 entries) | Hamming distance matrix calculator in CLI |
| `github.com/SuperInstance/casting-call-gpu` | GPU signature matcher | None — complete |

### What Needs Building

| Component | Effort | Priority | Notes |
|-----------|--------|----------|-------|
| Automated anchor scorer | 10 days | P1 | Train classifiers for each anchor point (currently manual) |
| Signature convergence experiment runner | 4 days | P1 | Script to automate iterative exposure experiments |
| Expanded signature database (50+ models) | 14 days | P2 | Currently only 14 signatures across 5 models |
| Task→ideal signature predictor | 7 days | P2 | Generate ideal signatures from task descriptions |
| Cross-lingual signature analyzer | 5 days | P2 | Compare English vs Chinese vs other language signatures |

### Dependency Chain
- **Voice Signatures Paper** → `casting-call` (anchor analyzer) → automated scorer
- **Voice Signatures Paper** → `casting-call-gpu` → GPU matching (supports 50+ model DB)
- **Voice Signatures Paper** → MCP server (from CPA) → signature query endpoints

---

## 3. Federated Model Evaluation — "Through Git-Native Trust Systems"

### Claims
1. Git is sufficient infrastructure for federated evaluation sharing
2. Trust-weighted evaluation outperforms unweighted averaging by 31%
3. 56 evaluations across 9 templates show clear model differentiation
4. Self-supervised auto-logging builds model knowledge organically

### Existing Code

| Repo | Complete | Missing |
|------|----------|---------|
| `github.com/SuperInstance/casting-call` | Evaluation schema, trust engine, CLI query, 56 evaluations | Auto-logging instrumentation |
| `github.com/SuperInstance/casting-call` | `cast recommend` command | Web UI for browsing evaluations |

### What Needs Building

| Component | Effort | Priority | Notes |
|-----------|--------|----------|-------|
| Agent auto-logging instrumentation | 5 days | P0 | Hook into agent model call layer; logs evaluations automatically |
| Trust auto-update from outcomes | 3 days | P1 | After an agent acts on a recommendation, log outcome and adjust trust |
| Web UI for evaluation browsing | 7 days | P1 | Visual dashboard: models, evaluations, trust scores |
| Consensus filter + outlier removal | 2 days | P1 | Implement the 2-sigma filter for recommendation accuracy |
| Cross-repo trust propagation | 5 days | P2 | If contributor X is trusted in repo A, propagate to repo B |

### Dependency Chain
- **Federated Eval Paper** → `casting-call` → auto-logging (needed for self-supervised extension)
- **Federated Eval Paper** → `casting-call` → trust engine → trust auto-update
- **Federated Eval Paper** → Web UI (visualizes the 56 evaluations)

---

## 4. FLUX ISA White Paper (FM's) — "Federated Learning and Unified eXchange ISA"

### Claims
1. A unified ISA (Instruction Set Architecture) enables models to exchange information across architectures
2. FLUX enables heterogeneous model federation without fine-tuning
3. The protocol supports inference routing, partial evaluation, and incremental computation
4. FLUX is compatible with existing transformer architectures

### Existing Code

| Repo | Complete | Missing |
|------|----------|---------|
| `github.com/SuperInstance/plato-kernel` | Base kernel, message routing | FLUX-specific instruction set not implemented |
| `github.com/SuperInstance/plato-protocol` | Protocol definitions | No FLUX ISA encoding/decoding |
| `github.com/SuperInstance/plato-relay` | Message relay | No FLUX-aware routing |

### What Needs Building

| Component | Effort | Priority | Notes |
|-----------|--------|----------|-------|
| FLUX ISA specification (formal) | 14 days | P1 | Complete instruction set definition, encoding, validation |
| FLUX encoding/decoding library | 10 days | P1 | Rust library for encoding/decoding FLUX instructions |
| FLUX-aware relay router | 7 days | P1 | Modify plato-relay to route based on FLUX instructions |
| Heterogeneous model adapter | 14 days | P1 | Adapter layer for non-transformer architectures |
| FLUX inference runtime | 21 days | P2 | Runtime that executes FLUX instruction sequences |
| Test suite with cross-architecture eval | 10 days | P2 | Verify FLUX works across transformer, Mamba, state-space models |

### Dependency Chain
- **FLUX ISA Paper** → `plato-kernel` → FLUX instruction set (blocks everything)
- **FLUX ISA Paper** → `plato-protocol` → FLUX encoding (blocks relay)
- **FLUX ISA Paper** → `plato-relay` → FLUX routing
- **FLUX ISA Paper** → Heterogeneous adapter → cross-architecture eval

---

## 5. PLATO Quality Gate (FM's) — "Quality Verification for Agent Outputs"

### Claims
1. Agent outputs need verifiable quality gates before reaching human consumers
2. PLATO's quality gates can automatically detect truncation, hallucination, and format errors
3. Quality gates integrate with the PLATO relay protocol
4. Evaluation outcomes can be fed back into model selection (Casting-Call)

### Existing Code

| Repo | Complete | Missing |
|------|----------|---------|
| `github.com/SuperInstance/plato-sdk` | Basic message validation | Quality gate logic not implemented |
| `github.com/SuperInstance/plato-lab-guard` | Lab environment guards | Not generalized to production quality gates |
| `github.com/SuperInstance/plato-afterlife` | Afterlife processing | No quality gate integration |

### What Needs Building

| Component | Effort | Priority | Notes |
|-----------|--------|----------|-------|
| Quality gate detection library | 7 days | P0 | Detect truncation, hallucination, format errors automatically |
| PLATO relay integration | 5 days | P0 | Quality gate fires before message relay to human |
| Quality→eval feedback loop | 3 days | P1 | Auto-log quality gate results as evaluations in Casting-Call |
| Configurable quality thresholds | 4 days | P1 | Per-channel, per-model threshold configuration |
| Quality visualization dashboard | 7 days | P2 | Web UI showing pass/fail rates by model and task type |

### Dependency Chain
- **PLATO Quality Gate Paper** → `plato-sdk` → quality gate library (blocks relay integration)
- **PLATO Quality Gate Paper** → `plato-afterlife` → quality→eval feedback (connects to Casting-Call)
- **PLATO Quality Gate Paper** → `casting-call` → auto-logging (receives quality outcomes)

---

## 6. The Fleet Grows Itself (Narrative) — "Organic Agent Fleet Growth"

### Claims (narrative/non-rigorous)
1. Agent fleets grow organically when the right infrastructure exists
2. Each new agent compoundingly improves fleet capability
3. Knowledge sharing between agents (via PLATO) creates collective intelligence
4. Fleet growth follows a network effect: value increases superlinearly with agent count

### Existing Code

| Repo | Complete | Missing |
|------|----------|---------|
| `github.com/SuperInstance/cocapn-keeper` | Base keeper infrastructure (routing, auth, monitoring) | Agent discovery, spawning, lifecycle management |
| `github.com/SuperInstance/plato-relay` | Message relay | No fleet-wide broadcast or swarm coordination |
| `github.com/SuperInstance/openmanus-vessel` | Single-agent vessel (demo) | Multi-agent fleet orchestration |

### What Needs Building

| Component | Effort | Priority | Notes |
|-----------|--------|----------|-------|
| Agent auto-discovery protocol | 7 days | P1 | Agents broadcast capabilities, find peers |
| Fleet spawning API | 10 days | P1 | `spawn <agent_type> <capability>` from keeper |
| Swarm coordination protocol | 14 days | P2 | Split tasks, aggregate results across agents |
| Cross-vessel knowledge sharing | 7 days | P2 | PLATO-based knowledge transfer between agents |
| Fleet health dashboard | 10 days | P2 | Visual: agents online, capacity, recent tasks |

### Dependency Chain
- **Fleet Grows Itself** → `cocapn-keeper` → auto-discovery (blocks spawning)
- **Fleet Grows Itself** → `plato-relay` → swarm coordination
- **Fleet Grows Itself** → `openmanus-vessel` → fleet spawning API
- **Fleet Grows Itself** ← **PLATO Quality Gate** (quality feedback enables self-growth)

---

## Master Priority

### P0 — Must Ship This Month

| # | Item | Paper | Repo | Effort | Blocks |
|---|------|-------|------|--------|--------|
| 1 | `casting-call` npm/pypi package | CPA | casting-call | 2 days | All downstream MCP servers |
| 2 | MCP server for signature/recommend queries | CPA, Voice | casting-call | 3 days | Real-time routing |
| 3 | Agent auto-logging instrumentation | Federated Eval | casting-call | 5 days | Self-supervised loop |
| 4 | Quality gate detection library | PLATO QG | plato-sdk | 7 days | Relay integration |
| 5 | PLATO relay quality gate integration | PLATO QG | plato-relay | 5 days | Production quality |
| 6 | Consensus filter + outlier removal | Federated Eval | casting-call | 2 days | Recommendation accuracy |
| 7 | FLUX ISA formal specification | FLUX ISA | plato-kernel | 14 days | All FLUX work |

**Total P0 effort: ~38 days** (can parallelize across 2-3 people → ~2 weeks calendar)

### P1 — Must Ship This Quarter

| # | Item | Paper | Repo | Effort | Notes |
|---|------|-------|------|--------|-------|
| 8 | Automated anchor scorer (10 classifiers) | Voice | casting-call | 10 days | Automates voice measurement |
| 9 | Trust auto-update from outcomes | CPA, Fed Eval | casting-call | 3 days | Closes feedback loop |
| 10 | Web UI for evaluation browsing | Fed Eval | casting-call | 7 days | Human-readable eval system |
| 11 | Protocol spec document | CPA | casting-call | 2 days | Formal protocol |
| 12 | Signature convergence experiment runner | Voice | casting-call | 4 days | Reproduce paper results |
| 13 | FLUX encoding/decoding library | FLUX ISA | plato-protocol | 10 days | Core FLUX impl |
| 14 | FLUX-aware relay router | FLUX ISA | plato-relay | 7 days | Enables FLUX routing |
| 15 | Heterogeneous model adapter | FLUX ISA | plato-kernel | 14 days | Cross-architecture |
| 16 | Quality→eval feedback loop | PLATO QG | plato-afterlife | 3 days | Connects QG to Casting-Call |
| 17 | Configurable quality thresholds | PLATO QG | plato-sdk | 4 days | Per-channel config |
| 18 | Agent auto-discovery protocol | Fleet | cocapn-keeper | 7 days | Fleet growth |
| 19 | Fleet spawning API | Fleet | cocapn-keeper | 10 days | Fleet growth |

**Total P1 effort: ~81 days** (parallelizable across teams → ~4-6 weeks calendar)

### P2 — Nice to Have (Research Quality)

| # | Item | Paper | Repo | Effort | Notes |
|---|------|-------|------|--------|-------|
| 20 | Expanded signature database (50+ models) | Voice | casting-call | 14 days | Scale research |
| 21 | Task→ideal signature predictor | Voice | casting-call | 7 days | Auto-routing |
| 22 | Cross-lingual signature analyzer | Voice | casting-call | 5 days | Language transfer |
| 23 | Multi-repo merge strategy | CPA | casting-call | 4 days | Federation depth |
| 24 | Cross-repo trust propagation | Fed Eval | casting-call | 5 days | Trust federation |
| 25 | FLUX inference runtime | FLUX ISA | plato-kernel | 21 days | Execution engine |
| 26 | FLUX cross-architecture test suite | FLUX ISA | plato-kernel | 10 days | Verify claims |
| 27 | Quality visualization dashboard | PLATO QG | plato-sdk | 7 days | Visual QG |
| 28 | Swarm coordination protocol | Fleet | plato-relay | 14 days | Multi-agent |
| 29 | Cross-vessel knowledge sharing | Fleet | plato-relay | 7 days | Collective intelligence |
| 30 | Fleet health dashboard | Fleet | cocapn-keeper | 10 days | Fleet visibility |

**Total P2 effort: ~104 days** (ongoing research, can parallelize)

---

## Summary: Key Milestones

| Milestone | Papers | Effort | Timeline |
|-----------|--------|--------|----------|
| **MVP:** Casting-Call published + MCP server live | CPA, Voice | 5 days | Week 1 |
| **Self-supervised:** Auto-logging + quality gates active | Fed Eval, PLATO QG | 17 days | Weeks 2-3 |
| **FLUX Foundation:** ISA spec + encoding library | FLUX ISA | 24 days | Weeks 3-5 |
| **Fleet Basics:** Auto-discovery + spawning API | Fleet | 17 days | Weeks 3-5 |
| **Quality Complete:** Web UI + trust auto-update + feedback loops | All | 15 days | Weeks 4-5 |
| **Research Scale:** 50+ model signatures + convergence experiments | Voice | 18 days | Weeks 5-7 |
| **FLUX Runtime:** Inference engine + cross-architecture tests | FLUX ISA | 31 days | Weeks 6-10 |
| **Fleet Swarm:** Coordination + knowledge sharing + dashboard | Fleet | 31 days | Weeks 6-10 |

**Total estimated effort to full implementation: ~223 days** (~11 months single developer, ~3 months with all three fleet agents working in parallel).
