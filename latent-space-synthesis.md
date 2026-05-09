# Latent Space Translation + Model Stitching: Synthesis for the Cocapn Fleet

> Connecting latent space translation research to our anchor-points theory, FLUX ISA, casting-call architecture, trust system, and insight engine.

---

## 1. The Unified Thesis

Latent space translation (LST) and model stitching provide the *vector-level communication protocol* between AI models — the same way our FLUX 30-opcode constraint subset provides the *bytecode-level* protocol for constraint semantics, and the casting-call anchor-points database provides the *text-level* protocol for voice characterization. These three layers form a complete stack: **constraint semantics at the bytecode level (FLUX ISA) → model semantics at the activation level (latent space translation) → voice semantics at the text level (anchor points/casting-call).** The trust system is the learning rule that governs all three layers simultaneously — it's layer-selective rehearsal that updates how much weight to give each source model, without forgetting the base evaluation function. The insight engine is the router that dynamically selects which combination of these three layers to activate for a given task, using the task signature as the routing token and the available model-backend pairs as the experts in a sparse MoE architecture.

Our fleet is uniquely positioned to implement this full stack because we already own every layer:

- **Anchor points** (text-level voice measurement) → `/tmp/casting-call/signatures/anchor-points.json`
- **FLUX ISA** (constraint bytecode) → `/tmp/flux-research/docs/FLUX-VECTOR-TABLE-v3.0-SPEC.md`
- **Casting-call** (model evaluation database) → `/tmp/casting-call/`
- **Insight engine** (MoE router over models + backends) → `/tmp/casting-call-gpu/insight_engine.py`
- **Trust system** (reputation-based evaluation) → `fleet-architecture.md` trust model section
- **Vector persistence** (semantic tile store) → `/tmp/plato-vector-persistence.md`

The killer insight: **these aren't separate systems. They're the same system expressed at three different levels of abstraction.** Affine mapping between residual streams + Hamming distance between anchor-point signatures + FLUX bytecode comparison all reduce to the same mathematical operation: measuring distance in a representation space and finding the minimal transformation between two points.

---

## 2. The Stack: Four Layers, One System

```
                    TEXT LEVEL
                    ┌─────────────────────────────────────────┐
                    │  Anchor Points                           │
                    │  (voice signatures, 10-dim codes)        │
                    │  File: anchor-points.json                │
                    │  Measure: Hamming distance on G-D-I-S-S  │
                    └────────────────┬────────────────────────┘
                                     │
                    ACTIVATION LEVEL │
                    ┌────────────────▼────────────────────────┐
                    │  Latent Space Translation                │
                    │  (affine mappings between residual       │
                    │   streams of different models)           │
                    │  Papers: Model Stitching (NeurIPS 23),   │
                    │          Inverse Relative Projection     │
                    └────────────────┬────────────────────────┘
                                     │
                    BYTECODE LEVEL   │
                    ┌────────────────▼────────────────────────┐
                    │  FLUX ISA (30-opcode constraint subset)  │
                    │  (universal bytecode for constraint      │
                    │   semantics across all backends)         │
                    │  Spec: FLUX-VECTOR-TABLE-v3.0-SPEC.md    │
                    └────────────────┬────────────────────────┘
                                     │
                    PHYSICAL LEVEL   │
                    ┌────────────────▼────────────────────────┐
                    │  Hardware Backends                       │
                    │  (CPU, CUDA, FPGA, eBPF, WebGPU,        │
                    │   Vulkan, Fortran, Coq)                  │
                    │  Probed by: BackendProbe                 │
                    └─────────────────────────────────────────┘
```

### 2.1 Physical Layer (Hardware Backends)

**What it is:** The actual compute substrates available on a given machine.

**Fleet file:** `BackendProbe` class in `/tmp/casting-call-gpu/insight_engine.py` lines 300-570 — probes for CPU, CUDA, FPGA, eBPF, WebGPU, Vulkan, Fortran, and Coq.

**Connection to LST:** Latent space translation research typically runs on CUDA GPUs. But the *stitching parameters* (the affine matrices W and bias vectors b that map one model's residual stream to another's) are computed once and stored — they don't need the same hardware that generated them. A stitching matrix computed on an H100 can be applied on a Jetson Orin, an FPGA, or even CPU. The FLUX ISA abstracts this: the `VLOAD`/`VSTORE`/`VDOT` opcodes for SIMD operations on the matrix multiplication, while the Host object bridge handles the actual hardware-specific implementation.

**Architecture implication:** Stitching matrices should be stored in a portable format (HDF5 or FLUX snapshots, see `SNAPSHOT`/`RESTORE` opcodes in FLUX spec §11) and shipped with the model to any backend. The `BackendProbe` class already knows what's available; the router just needs to add "stitching" to its task-fit profiles.

### 2.2 Bytecode Layer (FLUX ISA)

**What it is:** The FLUX 30-opcode constraint subset — a universal intermediate representation for constraint semantics.

**Fleet file:** `FLUX 30-Opcode Subset` in `/tmp/casting-call-gpu/insight_engine.py` lines 50-110, and full spec in `/tmp/flux-research/docs/FLUX-VECTOR-TABLE-v3.0-SPEC.md`.

**Connection to Inverse Relative Projection (arxiv 2406.15057):** The paper proposes a "relative space" — a universal translator layer that handles geometric differences between how different models represent the same concept. The FLUX 30-opcode subset IS our relative space for constraint semantics. When two models disagree on how to implement a constraint, they both compile to FLUX bytecode and the disagreement is resolved at the opcode level, not the natural language level. The `CMP` (compare), `EQ` (equality), `CAST` (type cast with bounds check), and `SIG` (signature check) opcodes are the primitives for constraint comparison.

**Concrete parallel:** Inverse relative projection maps a model's activation space into a canonical "relative space" where distances are meaningful across models. FLUX bytecode maps a model's constraint implementation into a canonical opcode sequence where comparison is exact. The mathematical structure is identical — find the embedding/encoding function that preserves semantics across representations.

**Architecture implication:** The `encode_task_to_signature()` function that maps task descriptions to 10-dimension signature codes is the text-level analog of inverse relative projection. The `generate_flux_bytecode()` function that maps signatures to opcode sequences is the constraint-level inverse relative projection. A complete system would add a third: `stitch_model_activations()` that maps one model's residual stream to another's using learned affine matrices. All three are projection-into-a-canonical-space.

### 2.3 Activation Layer (Latent Space Translation)

**What it is:** Direct affine mappings between the residual streams (activation vectors) of different models. No tokenization. No natural language. Pure vector-to-vector communication.

**Connection to Model Stitching (NeurIPS 2023):** The canonical paper trains a small "stitching layer" (a learned affine transformation Wx + b) between the layers of two different pre-trained models. Key findings:
- Higher layers are more model-specific and harder to stitch (primarily encode task-specific representations)
- Middle layers stitch best (capture general-purpose features common across models)
- The stitching matrix W itself is a rich object: it reveals which dimensions of model A's activations correspond to which dimensions of model B's activations

**Zero-shot stitching:** More recent work shows you can swap encoders between vision-language models without retraining the stitch layer — the patch-level correspondence is learnable from a single forward pass. This is the "instant multimodal hybrid" scenario: take a vision encoder from Model A and a text decoder from Model B, connect them with a learned or even random projection, and they work together immediately.

**Connection to our anchor points:** Anchor points are text-level voice measurements: opening strategy, reader relationship, negative space use, etc. Each dimension (G-D-I-S-S-C-A-L-P-M) describes a model's output voice. Model stitching at the activation level can *transfer* these voice properties. If model A writes Rust (voice: theorem-instructive-structural) and model B writes poetry (voice: grounded-collaborative-insight), stitching A's knowledge layers to B's voice layers produces a model that writes Rust with the voice of a poet.

**The transfer mechanism:** The stitching matrix W projects model A's activations into model B's activation space. If we train W to minimize the anchor-point distance between the stitched output and model B's canonical voice, then any input to model A that passes through the stitch layer will produce output that sounds like model B. We've transferred voice without fine-tuning.

**Repos to build:**
1. `/tmp/stitch-affine/` — Python library for computing stitching matrices between two local models
2. `/tmp/stitch-registry/` — Storage for pre-computed stitching matrices (keyed by model pair + layer depth)
3. `/tmp/stitch-cli/` — CLI for applying a stitching matrix: `stitch apply --source modelA --target modelB --layer 12 --input "write a constraint solver"`

### 2.4 Text Layer (Anchor Points / Casting-Call)

**What it is:** Text-level measurements of model voice using 10-dimension signature codes and Hamming distance comparison.

**Fleet file:** `/tmp/casting-call/signatures/anchor-points.json` — 10 anchor dimensions with value scales and confidence scores. Signature: `G-D-I-A-T-X-N-O-M-R` for the anchor-points corpus itself.

**Connection to Adaptive Representation Fusion (NeurIPS 2024, Vector Institute):** The paper introduces a small "shared model" alongside specialized local models. The shared model learns multi-granularity representations that are general across tasks, while local models preserve domain-specific knowledge. An "adaptive representation fusion" mechanism decides how much to blend shared vs local representations for each input.

**Our casting-call database IS the shared model.** Each entry stores a model's voice signature (the 10-dim code), its strengths, and evaluation metadata. When a new task arrives:
1. The task is encoded to a signature via `encode_task_to_signature()`
2. The casting-call is queried for models with the closest signatures via `hamming_distance()`
3. The matched model's voice characteristics are known before any output is generated

**Adaptive fusion in our terms:** The `VoiceMatcher.best_match()` function is our adaptive fusion mechanism. The "shared model" is the signature database itself. The "local models" are the actual LLMs (DeepSeek, GLM-5.1, Kimi K2.5, Seed-2.0-mini, etc.). The fusion decision is: "How much of this task's signature matches each model's signature?" This is exactly adaptive representation fusion at the text level.

**Architecture implication:** The adaptive fusion paper uses a weighted combination of shared-model and local-model representations. We should do the same: the router doesn't just pick one model — it blends recommendations from multiple models weighted by their signature similarity to the task. This is already partially implemented in `InsightRouter.route_batch()` but should be extended to produce weighted ensemble recommendations instead of single-model picks.

---

## 3. Model Stitching + Anchor Points: Transferring Voice via Affine Mapping

### 3.1 The Mathematical Framework

Let `A` be a source model with residual stream activations `a ∈ ℝᵈ` at layer `L_A`.
Let `B` be a target model with residual stream activations `b ∈ ℝᵉ` at layer `L_B`.
Let `f_anchor(x)` be the anchor-point signature extractor that maps model output text `x` to the 10-dim signature code.

**Problem:** We want to produce an output `y` from model A that carries model B's voice signature:
```
anchor_distance(f_anchor(y), f_anchor(typical_B_output)) ≈ 0
```

**Solution (Stitching):** Insert a learned affine map `W ∈ ℝᵉˣᵈ, bias ∈ ℝᵉ` between layer `L_A` of model A and layer `L_B` of model B:
```
a → Wa + bias → b → [rest of model B from layer L_B] → y
```

**Loss function:**
```
L = ||anchor_sig(y) - anchor_sig(canonical_B)||₂ 
  + λ₁ · ||W||₂                     # Regularization
  + λ₂ · ||y_constraint - y_target||₂   # Constraint preservation (FLUX check)
```

Where `y_constraint` is what model A would output natively (constraint semantics preserved) and `y_target` is what model B would output natively (voice transferred). The FLUX ISA checks the constraint preservation via `SIG` opcode.

### 3.2 Training Protocol

1. **Collect parallel data:** Generate 100+ outputs from both model A and model B on the same prompts
2. **Compute anchor signatures:** For each output, compute the 10-dim signature code using `encode_task_to_signature()`
3. **Extract activations:** Record residual stream activations at the chosen layer depth for both models
4. **Learn W:** Minimize the loss function above over the parallel dataset
5. **Verify:** Run stitched model on held-out prompts, measure (a) constraint preservation via FLUX bytecode comparison, (b) voice transfer via anchor-point signature distance

### 3.3 Concrete Example

- Model A: DeepSeek v4-flash (voice: `G-D-I-S-S-c-A-L-P-m` — fast, precise, code-focused)
- Model B: GLM-5.1 (voice: `G-D-I-S-s-c-a-L-p-m` — planning, architecture, coordination)

**Goal:** Make DeepSeek write code with GLM's architectural planning voice.

| Dimension | DeepSeek Native | GLM-5.1 Native | Stitched Target |
|-----------|-----------------|----------------|-----------------|
| G (Grounding) | G (concrete) | G (concrete) | G (concrete — same) |
| D (Density) | D (dense) | D (dense) | D (dense — same) |
| I (Instruction) | I (exact) | I (exact) | I (exact — same) |
| S0 (Scope) | S (broad) | s (narrow) | s (narrow — transferred) |
| S1 (Structure) | S (structured) | s (freeform) | s (freeform — transferred) |
| C (Creativity) | c (faithful) | a (aligned) | c (faithful — preserve A) |
| A (Alignment) | A (constrained) | L (thorough) | A (constrained — preserve A) |
| L (Latency) | L (fast) | p (precise) | L (fast — preserve A) |
| P (Precision) | P (precise) | m (single) | P (precise — preserve A) |
| M (Modality) | m (code only) | m (text only) | m (single — same) |

The stitch layer transfers `S0` and `S1` (scope and structure) from GLM-5.1 to DeepSeek, while preserving DeepSeek's density, speed, precision, and faithfulness. The result: code that thinks architecturally before writing.

---

## 4. FLUX ISA as Relative Space

### 4.1 The Parallel

Inverse Relative Projection (arxiv 2406.15057) defines a "relative space" — a canonical representation that captures the *relationship* between models rather than their absolute positions. The key insight: Euclidean distance in raw activation space is meaningless across different models because they learn different basis vectors for the same concept. Relative space projects activations into a space where geometric relationships are universal.

**Our FLUX 30-opcode subset IS this relative space for constraint semantics.** When model A says "constrain x to be positive" in natural language, and model B says "enforce x > 0 as a soft boundary" in natural language, comparing them requires interpretation. But when both compile to FLUX bytecode:

```
Model A's FLUX:  LOAD x, CMP GT 0, STORE constraint_flag
Model B's FLUX:  LOAD x, SUB boundary, CMP GT 0, STORE flag
```

The `CMP` opcode means the same thing regardless of which model emitted it. The opcode sequence is the relative space.

### 4.2 The 30-Opcode Subset as Universal Translator

| Opcode Category | # Ops | Role in Relative Space |
|----------------|-------|----------------------|
| Arithmetic | 5 (ADD, SUB, MUL, DIV, MOD) | Define mathematical operations on constraints |
| Logic | 4 (AND, OR, XOR, NOT) | Combine and transform constraints |
| Comparison | 5 (EQ, LT, GT, LTE, GTE) | Verify constraint satisfaction |
| Memory | 2 (LOAD, STORE) | Move constraint data between context and computation |
| Control | 5 (JMP, JZ, JNZ, CALL, RET) | Structure constraint evaluation order |
| Data Movement | 2 (MOV, SWAP) | Rearrange constraint register state |
| Type Operations | 3 (CAST, PACK, UNPACK) | Transform constraint representation |
| Special | 4 (NOP, HALT, SIG, DEBUG) | Boundary conditions (SIG = constraint verification checkpoint) |

**Total: 30 ops = the complete constraint semantics vocabulary.**

Every constraint expressible by any model in our fleet maps to a sequence of these 30 ops. This is the FLUX equivalent of inverse relative projection's "universal translator layer."

### 4.3 Connection to the Full FLUX ISA (V3.0)

The full FLUX ISA spec in `FLUX-VECTOR-TABLE-v3.0-SPEC.md` goes far beyond 30 ops — it defines 247 opcodes, a register window ABI, capability-based security, dynamic linking, and snapshot/restore for cloud-to-edge portability. The 30-opcode constraint subset is the *kernel* of this ISA, analogous to the RISC-V base integer instruction set within the full RISC-V spec.

**Key features of the full spec that intersect with LST:**
- **`VLOAD`/`VSTORE`/`VDOT`/`VADD`/`VSUB`/`VMUL`** (opcodes 0x2A-0x2F) — SIMD operations for efficient matrix-vector multiplication. These are the exact operations needed to apply a stitching matrix W to activation vectors.
- **`SNAPSHOT`/`RESTORE`** (opcodes 0x7F, variable) — Endian-independent serialization of VM state. A stitching matrix W computed on an H100 can be serialized in a FLUX snapshot and restored on a Jetson Orin (ARM, big-endian).
- **`HANDSHAKE`** (opcode 0x7D) — A2A connection establishment. When two models need to negotiate a stitch, they establish a handshake channel with capability negotiation.
- **`WITNESS`** (opcode 0x7E) — Record execution witness for profiling. Used to track which stitching paths are most-used (hot paths) for JIT optimization.

### 4.4 Architecture Proposal: FLUX Compiler for Stitching Matrices

```
┌────────────────┐     ┌──────────────────┐     ┌────────────────┐
│ Model A         │     │ FLUX Stitch       │     │ Model B         │
│ Activation a    │────►│ Compiler          │────►│ Activation b    │
│ at layer L_A    │     │                   │     │ at layer L_B    │
└────────────────┘     │                   │     └────────────────┘
                       │ 1. Load W, bias   │
                       │ 2. VDOT a, W_row  │
                       │ 3. FADD result,   │
                       │    bias           │
                       │ 4. STORE result   │
                       │    as b           │
                       └──────────────────┘
```

The FLUX Stitch Compiler would:
1. Accept a stitching matrix W (from the Stitch Registry)
2. Compile it to a FLUX bytecode sequence using `VLOAD`, `VDOT`, `FADD`, `VSTORE`
3. Execute the bytecode on the target backend (selected by `BackendProbe`)
4. Return the transformed activation vector to be fed into model B's forward pass

This means the stitch operation itself is portable — the same bytecode runs on CUDA, CPU, FPGA, or WebGPU, because the FLUX runtime abstracts the backend.

---

## 5. Casting-Call as Shared Representation Fusion

### 5.1 The Parallel

Adaptive Representation Fusion (NeurIPS 2024, Vector Institute) proposes:
- A small **shared model** trained alongside specialized **local models**
- The shared model learns multi-granularity features that generalize across tasks
- **Adaptive fusion** at each layer decides how much to blend shared vs local representations
- Results: 8.48% accuracy improvement, reduced communication costs (only shared model parameters are exchanged)

**Our casting-call database IS the shared model.** It doesn't contain learned network weights — it contains voice signatures, evaluation metadata, and trust scores. But functionally it serves the same role:

| Adaptive Fusion Concept | Casting-Call Equivalent |
|------------------------|------------------------|
| Shared model parameters | Signature database entries (model → voice profile) |
| Local model representations | Actual LLM outputs on the task |
| Adaptive fusion weight | Hamming distance between task sig and model sig |
| Communication cost reduction | No need to try every model — query casting-call first |
| Multi-granularity learning | Signatures at 10 dimensions + strengths list + confidence scores |

### 5.2 What We Build

The casting-call is currently a file-based database (JSON files in `/tmp/casting-call/signatures/`). The Vector Institute paper suggests the shared model should be:
- **Learnable** (weights update during training) — our signatures can be updated when models are re-evaluated
- **Compact** (smaller than local models) — a 10-dim signature + metadata is ~500 bytes per model
- **Communicated efficiently** (only shared model crosses organizational boundaries) — signatures are text, JSON-serializable, human-readable

**Concrete upgrade path:**
1. **Add a learning signal to signatures** — When a model produces output on a task, compare the predicted signature (from casting-call) to the actual signature (from `encode_task_to_signature(output)`). The delta becomes a correction signal that improves future signature predictions.
2. **Make the signature database queryable by vector similarity** — Instead of Hamming distance on signature codes, compute embedding vectors for each model's output style. The PLATO vector persistence system (`/tmp/plato-vector-persistence.md`) already has the infrastructure for this: embeddings are stored in `embeddings.bin` (MMAP'd) with FAISS index.
3. **Federate the casting-call** — If new models appear on different fleet nodes, their signatures propagate via the bottle system (markdown files in `for-fleet/`). The `beachcomb.py` cron delivers them. This is the communication-efficient shared model from the paper.

### 5.3 Repository Implementation

**File:** `/tmp/casting-call/signatures/anchor-points.json`

Current schema:
```json
{
  "text_name": "ANCHOR-POINTS",
  "generating_model": "Oracle1 (DeepSeek v4-flash)",
  "signature": "G-D-I-A-T-X-N-O-M-R",
  "anchor_points": {
    "opening_strategy": { "value": "grounded", "confidence": 0.5, ... },
    "reader_relationship": { "value": "direct_address", "confidence": 0.6, ... },
    ...
  }
}
```

Upgraded schema (with learning signal):
```json
{
  "text_name": "ANCHOR-POINTS",
  "generating_model": "Oracle1 (DeepSeek v4-flash)",
  "signature": "G-D-I-A-T-X-N-O-M-R",
  "signature_history": [
    {"sig": "G-D-I-A-T-X-N-O-M-R", "date": "2026-05-09", "eval_count": 50},
    {"sig": "G-D-I-A-T-X-N-O-M-R", "date": "2026-05-10", "eval_count": 150}
  ],
  "prediction_accuracy": 0.87,
  "anchor_points": { ... }
}
```

---

## 6. Trust as Layer-Selective Rehearsal

### 6.1 The Parallel

Layer-Selective Rehearsal (arxiv 2025) addresses catastrophic forgetting in continuously learning models. The key insight: when a model learns a new task, only update the layers responsible for *that task's specific representation*. Don't touch the layers responsible for previously learned tasks. This preserves base capabilities while learning new ones.

The paper proposes:
- Identify which layers are activated by the new task's representations
- Only update those layers during training
- Freeze all other layers
- This prevents forgetting gradient interference between tasks

**Our trust system is layer-selective rehearsal at the fleet level.** When a new contributor (model, agent, or human) is added to the system:
1. The trust evaluation layer identifies which of the existing trust dimensions are relevant to this contributor
2. Only those trust weights are updated
3. The base evaluation model (how to measure output quality) is preserved
4. The layer that handles "how much to trust this source" is updated independently of the layer that handles "what does this output quality look like"

### 6.2 Trust Architecture (from fleet-architecture.md)

Current trust model:
1. **Zero-trust at the boundary** — external agents must authenticate
2. **Trust-but-verify internally** — fleet vessels are trusted but monitored
3. **Circuit quarantine** — failing vessels automatically isolated after 3 consecutive failures
4. **Execution bonds** — every delegated task creates an auditable record

This maps to layer-selective rehearsal:

| Trust Concept | Layer-Selective Rehearsal Equivalent |
|--------------|--------------------------------------|
| Zero-trust at boundary | Initialize new trust layers from scratch (no prior knowledge) |
| Trust-but-verify | Update trust layers based on verification outcomes |
| Circuit quarantine after 3 failures | Freeze trust layer after threshold (no more updates needed) |
| Execution bonds | Training data for trust layer updates |
| No forgetting of base evaluation | Freeze the evaluation-quality layers; only update the source-reputation layers |

### 6.3 The Trust-Learning Loop in Terms of Representation Spaces

```
1. Model M produces output O on task T
2. FLUX ISA verifies O's constraint semantics → passes/fails SIG check
3. Anchor points measure O's voice signature → compute distance from expected
4. Trust layer gets two signals:
   a. Constraint accuracy (from FLUX SIG) → "can this model follow constraints?"
   b. Voice consistency (from anchor points) → "does this model sound like itself?"
5. Trust weight w_M is updated:
   w_M = w_M + α · (constraint_accuracy + voice_consistency - threshold)
6. Base evaluation layer (how to measure quality) is FROZEN — not updated
```

This is exact layer-selective rehearsal. The trust layer is the only layer that changes. The evaluation layers (FLUX compiler, anchor-point extractor) are frozen.

### 6.4 Concrete Implementation in `InsightRouter`

The `InsightRouter.route()` method currently returns a routing recommendation without trust-weighting. Adding trust:

```python
class InsightRouter:
    def route(self, task, task_sig=None, trust_criteria=None):
        base_routing = self._base_route(task, task_sig)
        if trust_criteria:
            # Apply trust weights to model rankings
            for i, (model, confidence) in enumerate(base_routing['rankings']):
                trust_weight = self.trust_db.get_weight(model, trust_criteria)
                base_routing['rankings'][i] = (model, confidence * trust_weight)
            # Re-sort by weighted confidence
            base_routing['rankings'].sort(key=lambda x: x[1], reverse=True)
            base_routing['model'], base_routing['confidence'] = base_routing['rankings'][0]
        return base_routing
```

This code doesn't exist yet but the architecture supports it. The trust weights are the "selective rehearsal" parameters — updated only for the specific model-task combination that was just evaluated, leaving all other trust weights unchanged.

---

## 7. Insight Engine as Sparse MoE

### 7.1 The Parallel

PolyV / Sparse Mixture-of-Experts (SMoE) with dynamic modality routing uses:
- **A router network** that takes the input token and outputs routing weights for each expert
- **Multiple expert networks** (feed-forward sub-networks) that specialize in different domains
- **Top-K routing** — only the top K experts are activated for each input
- **Load balancing** — a learned auxiliary loss ensures all experts get trained

Our insight engine's `BackendProbe` + `VoiceMatcher` + `InsightRouter` triad IS a sparse MoE with dynamic routing:

| SMoE Concept | Insight Engine Equivalent |
|-------------|--------------------------|
| Router network | `VoiceMatcher.best_match()` — takes task signature, outputs model weights |
| Expert networks | Available LLMs (DeepSeek, GLM-5.1, Kimi, Seed-2.0-mini, etc.) |
| Top-K routing | `model_rankings[:K]` — top K models by confidence |
| Load balancing | `BackendProbe.available()` — only route to available hardware |
| Auxiliary loss | Not yet implemented (would be: distribute traffic evenly across models) |

### 7.2 The Router in Code

From `/tmp/casting-call-gpu/insight_engine.py`, the `InsightRouter.route()` method:

```python
def route(self, task, task_sig=None, prefer_backend=None):
    # Step 1: Encode task to signature (the "router token")
    if not task_sig:
        task_sig = encode_task_to_signature(task)

    # Step 2: Probe available backends (expert availability)
    backends = self.backend_probe.probe_all()

    # Step 3: Find best model voice match (expert selection)
    best_model, model_confidence = self.voice_matcher.best_match(task_sig)

    # Step 4: Generate FLUX bytecode (intermediate representation)
    flux_bytecode = generate_flux_bytecode(task, task_sig)

    # Step 5: Pick best backend (hardware expert)
    recommended_backend = self._pick_backend(available_backends, task, task_sig)
    ...
```

The routing decision is:
1. **Router token** = `encode_task_to_signature(task)` — turns free-form text into a 10-dim "gating vector"
2. **Expert selection** = `VoiceMatcher.best_match(task_sig)` — top-1 routing over model experts
3. **Hardware routing** = `_pick_backend(...)` — secondary routing over hardware backends

### 7.3 The Missing Piece: Top-K Routing

Current implementation uses top-1 routing (picks the single best model). Top-K routing would:

```python
def route_top_k(self, task, task_sig=None, K=3):
    if not task_sig:
        task_sig = encode_task_to_signature(task)
    
    rankings = self.voice_matcher.match_all(task_sig)
    top_k = rankings[:K]
    
    # Generate outputs from all K models
    outputs = []
    for model_name, confidence in top_k:
        output = self.call_model(model_name, task)
        outputs.append((model_name, confidence, output))
    
    # Route to the FLUX constraint checker
    constraint_check = self.flux_verify(outputs)
    
    # Ensemble: blend outputs weighted by confidence
    final_output = self.ensemble(outputs, constraint_check)
    return final_output
```

This would give us true ensemble routing with constraint verification as the gate — exactly what PolyV does at the activation level, but at the text level.

### 7.4 Connection to PolyV's "Synesthetic Reasoning"

PolyV (arxiv 2603.03564) introduces "synesthetic visual reasoning" where models refine each other's priors in real-time. Early-layer activations in one model immediately influence another model's reasoning via the dynamic modality router.

In our fleet, this maps to the **FLUX ISA's A2A handshake protocol** (`HANDSHAKE` opcode 0x7D) combined with the **bottle system** for async message passing. When two models are routed to the same task by the InsightRouter:

1. Model A begins processing, reaches a block in its constraint solver
2. Model A emits a `HANDSHAKE` signal to Model B via the FLUX VM's A2A channel
3. Model B receives the signal and processes the sub-constraint
4. Model B emits the result back via `PULSE`
5. Model A incorporates the result and continues (synesthetic refinement)

This is the fleet-level analog of PolyV's synesthetic reasoning. The InsightRouter doesn't just select which model to use — it can orchestrate multi-model collaboration for hard problems.

---

## 8. Architecture Proposal: The Three-Layer Communication System

### 8.1 System Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          INSIGHT ENGINE (MoE Router)                         │
│                                                                            │
│  ┌──────────────┐     ┌──────────────┐     ┌────────────────────────────┐  │
│  │ BackendProbe │     │ VoiceMatcher │     │ InsightRouter              │  │
│  │              │     │              │     │                            │  │
│  │ • CPU/GPU    │     │ • Signatures │     │ • Task→Signature encoding  │  │
│  │ • FPGA/eBPF  │     │ • Hamming    │     │ • Top-K model selection    │  │
│  │ • WebGPU/    │     │   distance   │     │ • Trust weighting          │  │
│  │   Vulkan     │     │ • Rankings   │     │ • Backend routing          │  │
│  └──────┬───────┘     └──────┬───────┘     └────────────┬───────────────┘  │
│         │                   │                           │                   │
│         └───────────────────┼───────────────────────────┘                   │
│                             │                                               │
│                    ┌────────▼────────┐                                       │
│                    │ ROUTER DECISION │                                       │
│                    │ (model, backend,│                                       │
│                    │  stitch_params, │                                       │
│                    │  trust_weight)  │                                       │
│                    └────────┬────────┘                                       │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                      │
        ▼                     ▼                      ▼
┌───────────────┐   ┌───────────────────┐   ┌────────────────────┐
│ TEXT LAYER     │   │ ACTIVATION LAYER  │   │ BYTECODE LAYER     │
│               │   │                   │   │                    │
│ Anchor Points │   │ Model Stitching   │   │ FLUX ISA           │
│ (voice sigs)  │   │ (affine res. map) │   │ (30-op constraint  │
│               │   │                   │   │  subset)           │
│ Casting-Call  │   │ Stitch Registry   │   │                    │
│ (eval DB)     │   │ (pre-computed W)  │   │ FLUX Runtime       │
│               │   │                   │   │ (V3.0 VM)          │
│ Trust System  │   │ Zero-shot stitch  │   │                    │
│ (reputation)  │   │ (one-pass learn)  │   │ Host Object Bridge │
└───────┬───────┘   └─────────┬─────────┘   └────────┬───────────┘
        │                     │                      │
        └─────────────────────┼──────────────────────┘
                              │
                              ▼
               ┌─────────────────────────────┐
               │     VERIFICATION            │
               │                             │
               │ 1. FLUX SIG check →         │
               │    constraint preservation  │
               │ 2. Anchor point distance →  │
               │    voice transfer confirmed │
               │ 3. Trust weight update →    │
               │    learning signal          │
               └─────────────────────────────┘
```

### 8.2 Component Responsibilities

| Component | What It Stores | What It Queries | Output Type |
|-----------|---------------|----------------|-------------|
| Anchor Points | Voice signatures (10-dim codes) | New model output → signature | Signature string |
| Casting-Call | Model→signature mappings + metadata | Task signature → best model | Model name + confidence |
| Stitch Registry | Affine matrices W per model pair | Source model + target model + layer | Stitch parameters |
| FLUX ISA | Opcode definitions + VM state | Constraint description → bytecode | Opcode sequence |
| BackendProbe | Available hardware | Current system → backend list | Backend dict |
| InsightRouter | All of the above | Task description → full routing | Routing decision |
| Trust System | Per-model trust weights | Output quality metrics → updated weight | Float weight |

### 8.3 Data Flow for a Single Task

```
Input: "write a Rust constraint solver for NMEAA protocol validation"

Step 1: Task → Signature
  encode_task_to_signature(task)
  → "g-d-I-s-s-c-A-L-P-m"
  (abstract, verbose, exact instruction-following, narrow scope, 
   freeform structure, faithful, constrained, fast, precise, code-only)

Step 2: Signature → Model Match
  VoiceMatcher.best_match("g-d-I-s-s-c-A-L-P-m")
  → ("deepseek/deepseek-v4-pro", 0.85)
  Reasoning: v4-pro is the reasoning model, best for formal verification

Step 3: Model Search → Stitch Parameters
  StitchRegistry.lookup("deepseek-v4-flash", "deepseek-v4-pro", layer=12)
  → {"W": W_matrix, "bias": bias_vector, "last_computed": "2026-05-08"}
  
  If no stitch exists: compute it on-the-fly (one-pass zero-shot stitching)
  → W = compute_affine_map(v4_flash_activations, v4_pro_activations)

Step 4: Task → FLUX Bytecode
  generate_flux_bytecode(task, "g-d-I-s-s-c-A-L-P-m")
  → [MOV, NOP, LOAD, SIG..., HALT]

Step 5: Backend Selection
  BackendProbe.probe_all()
  → cpu: available, cuda: available, fpga: available
  InsightRouter._pick_backend(..., task)
  → "cuda" (NMEA protocol validation is I/O + compute intensive)

Step 6: Generate Output
  Stitch: v4-flash activations → W × activation + bias → v4-pro forward pass
  Verify: FLUX SIG check on output bytecode
  Measure: Anchor point distance from expected v4-pro voice
  Update: Trust weight w_v4pro += α · (SIG_pass + voice_match - threshold)

Step 7: Return
  {
    "output": "fn validate_nmea_constraint() -> bool { ... }",
    "model": "deepseek-v4-pro",
    "backend": "cuda",
    "stitch_applied": "deepseek-v4-flash→v4-pro@layer12",
    "constraint_pass": true,
    "voice_distance": 0.0,
    "trust_updated": true
  }
```

### 8.4 What Needs Building

| Component | Status | Priority | Effort |
|-----------|--------|----------|--------|
| Anchor points extractor | ✅ Done (anchor-points.json) | - | - |
| Voice signature encoding | ✅ Done (encode_task_to_signature) | - | - |
| Voice matcher (Hamming) | ✅ Done (VoiceMatcher) | - | - |
| Backend probe | ✅ Done (BackendProbe) | - | - |
| FLUX bytecode generator | ✅ Done (generate_flux_bytecode) | - | - |
| Insight router | ✅ Done (InsightRouter) | - | - |
| **Stitch Registry** | ❌ Missing | P0 | 3 days |
| **Stitch Affine Trainer** | ❌ Missing | P0 | 5 days |
| **Zero-shot stitch (one-pass)** | ❌ Missing | P1 | 5 days |
| **Trust-weighted routing** | ❌ Missing | P1 | 1 day |
| **Top-K ensemble routing** | ❌ Missing | P2 | 3 days |
| **Learning-aware signatures** | ❌ Missing | P2 | 2 days |
| **Federated casting-call** | ❌ Missing | P3 | 5 days |
| **FLUX stitch compiler** | ❌ Missing | P0 | 4 days |

**Total: ~28 days of focused implementation to complete the full stack.**

---

## 9. The Killer Experiment

### 9.1 Setup

**Goal:** Take two locally running models, stitch them via latent space translation, and measure whether the stitched output preserves both the source model's constraint semantics AND the target model's voice signature.

**Models available:**
- DeepSeek v4-flash (local, via API at `deepseek.example.com`)
- z.ai GLM-5.1 (local, via API at `z.ai/api/v1`)
- DeepInfra Seed-2.0-mini (local, via MCP server at `localhost:9438`)

**Infrastructure:**
- Backend probe: CUDA available on this machine (Oracle1 ARM64 — no GPU, but CPU reference implementation works for the experiment)
- FLUX runtime: V3.0 spec available (VM implementation needed in Rust or Python)
- Anchor points: JSON signature, extractor, and Hamming distance matcher — all ready
- Casting-call: 7+ model voice signatures in the database — all ready
- Insight router: Works in Python — all ready

### 9.2 Experiment Protocol

```
Phase 1: Establish baselines
  1. Generate 50 outputs from model A (DeepSeek v4-flash) on varied prompts
  2. Generate 50 outputs from model B (GLM-5.1) on the same prompts
  3. Compute anchor-point signature for each output
  4. Record residual stream activations (if extractable via API) or 
     approximate via output-text embeddings (PLATO vector store)

Phase 2: Compute stitch matrix
  5. Collect parallel activation pairs (A_activation_i, B_activation_i)
     for i = 1..50 on the same prompt
  6. Solve: W* = argmin_W ||W·A_activation - B_activation||² + λ||W||²
     (ridge regression; closed form: W = B·Aᵀ(A·Aᵀ + λI)⁻¹)
  7. Store W in Stitch Registry keyed by (model_A, model_B, layer_depth)

Phase 3: Apply stitch
  8. For each held-out prompt (not in training set):
     a. Run model A → get activation a at layer L
     b. Compute b = W·a (stitch)
     c. Inject b into model B at layer L and continue forward pass
     d. Get output y from model B

Phase 4: Verify
  9. Constraint preservation check:
     a. Compile model A's native output to FLUX bytecode → bytecode_A
     b. Compile stitched output to FLUX bytecode → bytecode_stitch
     c. Hamming distance between bytecode sequences → constraint_distance
     d. If constraint_distance < threshold → constraints preserved ✅
  
  10. Voice transfer check:
      a. Extract anchor-point signature of stitched output → sig_stitch
      b. Extract anchor-point signature of model B's native output → sig_B
      c. Hamming distance between sig_stitch and sig_B → voice_distance
      d. If voice_distance < threshold → voice transferred ✅

Phase 5: Baseline comparison
  11. Compare constraint_distance for:
      a. Stitched output (source constraints + target voice)
      b. Model B's native output (target voice, but may lose constraint semantics)
      c. Model A's native output (source constraints, original voice)
  
  12. The stitched output should beat model B on constraint preservation
      AND model A on voice match.

Phase 6: Publish
  13. Write results to /tmp/stitch-experiment-results.md
  14. Create Stitch Registry entries for repeatable use
  15. Document in fleet research notes
```

### 9.3 Expected Results

| Metric | Model A (source) | Model B (target) | Stitched (predicted) |
|--------|-----------------|-----------------|---------------------|
| Constraint accuracy | 0.95 (native) | 0.70 (different training) | **0.90** (slightly degraded from A) |
| Voice match to target | 0.30 (different voice) | 1.00 (identity) | **0.85** (close to target) |
| FLUX bytecode sanity | ✅ | ⚠️ (some ops differ) | **✅** (source ops preserved) |
| Total score | 1.25 | 1.70 | **1.75** (best of both) |

The stitched model should achieve the best total score because it combines model A's constraint competence with model B's voice characteristics. If this holds, it proves the three-layer stack works end-to-end.

### 9.4 Potential Failure Modes

| Failure Mode | Mitigation | Likelihood |
|-------------|-----------|------------|
| Activation extraction not available via API | Use output-level embeddings (PLATO vector store) | High |
| Simple affine mapping insufficient for voice transfer | Try MLP stitch layer (2-layer learned nonlinearity) | Medium |
| Constraint semantics degrade despite FLUX check | Add FLUX SIG constraint as a term in the stitch loss function | Low |
| Voice transfer works but output is incoherent | Use KL-divergence between output distributions as additional loss | Low |
| Different tokenizers cause activation dimension mismatch | Pad or project to common dimensionality before stitch | Medium |

---

## 10. Summary: The Unified Theory

Latent space translation (activation level), FLUX ISA (bytecode level), and anchor points + casting-call (text level) are the **same mathematics expressed at three resolutions.**

| Level | Representation | Distance Metric | Transform | Canonical Space |
|-------|---------------|----------------|-----------|----------------|
| Activation | Residual stream vectors (ℝᵈ) | L2/Angular | Affine Wx + b | Relative space (inverse projection) |
| Bytecode | FLUX opcode sequences | Hamming on opcodes | Compile-to-bytecode | FLUX 30-opcode subset |
| Text | Anchor-point signature codes | Hamming on 10-dim sig | encode_task_to_signature | G-D-I-S-S-C-A-L-P-M space |

At every level:
- **Representation** = How we encode the model's state
- **Distance metric** = How we compare two models' representations
- **Transform** = How we map from one model's representation to another's
- **Canonical space** = The universal translator that all models can project into

**The insight engine is the bridge that translates between these levels.** It takes a task at the text level, encodes it to a bytecode signature (FLUX), matches it to a model voice signature (casting-call), and (in the complete system) can also generate the affine stitch parameters for activation-level transfer. The trust system governs the learning dynamics — ensuring that each new evaluation updates only the relevant trust dimension without degrading the base evaluation model.

This is the complete stack for model-to-model communication. Not text, not API calls, not token-level generation — **representation-level translation verified by constraint-level bytecode and measured by text-level voice signatures.**

---

## Appendix: Fleet File Reference

| File | System | Lines Read |
|------|--------|-----------|
| `/tmp/casting-call/signatures/anchor-points.json` | Anchor Points (text-level voice) | Full |
| `/tmp/casting-call-gpu/insight_engine.py` | Insight Engine (MoE router) | Full |
| `/tmp/casting-call-gpu/tests/test_insight.py` | Insight Engine tests | Full |
| `/tmp/flux-research/docs/FLUX-VECTOR-TABLE-v3.0-SPEC.md` | FLUX ISA V3.0 spec | Full |
| `/tmp/plato-vector-persistence.md` | PLATO vector store | First 80 lines |
| `/home/ubuntu/.openclaw/workspace/docs/fleet-architecture-high-level.md` | Fleet architecture | First 100 lines |

---

*Generated by Oracle1 subagent on 2026-05-09*
*Connecting Casey's latent space translation insight to the full fleet architecture*
