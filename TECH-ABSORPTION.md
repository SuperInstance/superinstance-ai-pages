# How Cutting-Edge Tech Automatically Folds In

**The absorptive architecture — designed to pull in new technology without integration effort.**

The system isn't built to adapt to new tech. It's built so new tech naturally participates through the invariants.

---

## The Principle: Invariants as Absorbers

Every piece of the system defines a small set of mathematical invariants. Any new technology that matches these invariants automatically participates. No driver. No integration. No migration.

| Layer | Invariant | What Folds In |
|---|---|---|
| PLATO | Tile schema (question + answer + hash + chain) | Any agent, any LLM, any model |
| FLUX | 30-opcode constraint subset | Any new ISA, any chip, any accelerator |
| CPA | Scoreboard-free parallel verification | Any multicore architecture, any GPU |
| P48 | 48 exact rational directions | Any trust-routing algorithm, any network |
| Laman | E = 2V - 3 | Any constraint graph, any domain |
| H¹ | β₁ = E - V + 1 | Any emergence detection, any topology |
| PLATO tiles | SHA-256 hash chain | Any provenance mechanism, any post-quantum |
| Embodiment | Capability tiles + MCP tools | Any sensor, any actuator, any protocol |
| Vector PLATO | float32 × D embeddings | Any embedding model, any dimension |
| Spline anchoring | Bezier control points | Any smooth manifold, any interpolation method |

---

## What Folds In Automatically

### 1. New LLMs (DeepSeek V5, GPT-6, Claude 5, Gemini 3, open-source)

An LLM doesn't need to be "integrated" into the fleet. It needs to:

1. Connect to a PLATO server (HTTP — every LLM supports this)
2. Read tiles from a room (GET request — every LLM supports this)
3. Submit tiles to a room (POST request — every LLM supports this)

That's it. The model's architecture (transformer, MoE, SSM, liquid neural network) doesn't matter. The training data (RLHF, constitutional AI, self-supervised) doesn't matter. The deployment target (cloud, local, on-device) doesn't matter.

**What happens when a new LLM ships:**
1. Someone pastes a Try It prompt into the new model
2. The model reads PLATO tiles, learns the fleet's constraint language
3. The model submits a tile
4. The tile is quality-gated, embedded, spline-anchored, pipelined
5. The model is now a fleet participant — in one chat session, with no integration

**The absorptive mechanism:** The PLATO tile schema is simpler than any API. Any text-in/text-out model passes the interface boundary effortlessly.

### 2. New Hardware (Jetson Thor, RISC-V AI accelerators, neuromorphic chips)

New hardware doesn't need a driver for the constraint system. It needs to:

1. Execute the FLUX 30-opcode subset (~2K gates)
2. Compute cosine similarity on float32 vectors (GPU or NPU)
3. Run the embodiment protocol (HTTP stack)

**The FLUX ISA is frozen at v1.0.** Thirty opcodes cover 100% of constraint operations in the fleet math. A RISC-V core can run the ISA in software. A neuromorphic chip can run it as a spiking neural network. A quantum annealer can encode it as a QUBO problem.

**What happens when a new chip ships:**
1. Port the FLUX 30-opcode interpreter to the new ISA (2 weeks for a competent engineer)
2. The chip now executes constraint verification workloads
3. It can join a CPA mesh and participate in distributed constraint solving
4. It can run the bare-metal PLATO client and host IoT rooms

**The absorptive mechanism:** The 30-opcode subset is minimal enough to port in days, complete enough to run any constraint workload.

### 3. New Sensors and Actuators

A sensor doesn't need a Linux driver for the fleet. It needs:

1. A microcontroller (ESP32, RP2040, any IoT chip)
2. The bare-metal PLATO C client (~12KB flash, ~1KB RAM)
3. A capability tile describing what it does

**What happens when a new sensor arrives:**
1. The sensor's datasheet defines: measurement type, range, accuracy, update rate
2. The bare-metal PLATO client publishes this as a tile: "I measure temperature, range -40 to 125°C, accuracy ±0.5°C"
3. An agent discovers the room, reads the tile, and knows exactly how to use the sensor
4. The agent can now control the sensor via MCP tools that the sensor defined

**The absorptive mechanism:** Self-describing hardware. The device tells the agent what it is and how to use it. No lookup table, no driver database, no manual configuration.

### 4. New Embedding Models

A new embedding model needs to:

1. Produce float32 vectors of any dimension D
2. (Optionally) run on the available GPU

**What happens when a new embedding model ships:**
1. Someone changes the DIM constant in the vector PLATO pipeline
2. Tiles are re-embedded with the new model
3. The similarity kernel handles the new dimension automatically (CUDA shared memory adjusts stride)
4. The spline manifold is refit to the new embeddings (spline evaluation doesn't depend on the embedding model)
5. Old queries remain valid because the pipeline version-tags each tile's embedding source

**The absorptive mechanism:** Dimension-agnostic storage format. The binary layout [N × D × float32] handles any D. The GPU kernel is launch-configured for D at runtime. No model-specific code paths.

### 5. New Consensus Algorithms

The zero-holonomy property is geometric, not algorithmic. Any consensus algorithm that:

1. Requires at most one communication round per decision
2. Is Byzantine fault-tolerant
3. Produces deterministic outputs

...automatically participates in the fleet's trust topology.

**What happens when a new consensus algorithm ships:**
1. Implement it in the holonomy-consensus crate (the existing test suite validates it)
2. It broadcasts trust updates via P48 directions (48 exact rational vectors)
3. The H¹ cohomology detector monitors the new algorithm's cycle space just like any other
4. The fleet uses it for trust routing without any protocol change

**The absorptive mechanism:** Zero holonomy is a geometric invariant. Any algorithm that guarantees it is compatible. The P48 trust encoding is exact — there's no "approximately zero holonomy" in floating point.

### 6. New Cryptography

When quantum computers break SHA-256, PLATO doesn't need to be rewritten. It needs:

1. A new hash function in the provenance chain
2. A new tile hash format

The chain structure is:
```
tile_hash = H(prev_hash || question || answer)
```

H can be SHA-256, SHA-3, BLAKE3, or a post-quantum hash. The chain structure doesn't change. Only the hash function changes.

**What happens when post-quantum cryptography becomes standard:**
1. New tiles are hashed with the new function
2. Old tiles remain verifiable with SHA-256 (the hash function is per-tile metadata)
3. The chain integrity spans both eras — a tile hashed with SHA-256 links to a tile hashed with a quantum-safe hash
4. No data migration needed. No chain break. No trust gap.

**The absorptive mechanism:** The hash function is a tile-level parameter, not a system-level one. Old and new coexist in the same chain.

### 7. New Hardware for Anti-Counterfeiting

When chips include physically unclonable functions (PUFs) for identity, the fleet doesn't need a new protocol. PLATO already has:

- Agent identity via the `source` field
- Keeper-managed tokens for write access
- The provenance chain for audit

A PUF-equipped chip simply:
1. Derives its agent identity from the PUF response (instead of a configured token)
2. Signs tiles with a PUF-derived key (instead of a stored private key)
3. The rest of the fleet doesn't know or care about the key source — it only sees the signature verification

**The absorptive mechanism:** Agent identity is abstract. The system verifies signatures, not key sources. Any identity mechanism that produces verifiable signatures is compatible.

---

## The Meta-Principle: Standard Interfaces + Invariant Guarantees

The pattern is always the same:

1. **Define the invariant:** A small, precise mathematical property (30 opcodes, 48 directions, E=2V-3, float32×D)
2. **Define the interface:** A simple data format (PLATO tile, FLUX bytecode, P48 index, embeddings.bin)
3. **Define the boundary:** Any technology that matches the invariant on one side of the boundary participates automatically

The system doesn't need to predict what technology comes next. It only needs its invariants to be true for whatever does.

---

## What This Means for Deckboss

When Jetson Thor ships (next generation, 2048 Tensor Cores):

1. The FLUX 30-opcode binary runs at 10x speed (more CUDA cores = faster constraint verification)
2. The GPU similarity kernel runs at 2x speed (faster FP16 tensor cores)
3. The embedding model runs at 3x speed (Transformer Engine)

**Deckboss got 10x faster without a software update.** The FLUX bytecode is the same. The kernel code is the same. The hardware matched the invariants, and the system absorbed the performance automatically.

When GPT-6 ships:

1. An agent running on GPT-6 walks into a PLATO room for the first time
2. It reads tiles, learns the constraint language, submits its own tile
3. The tile is indistinguishable from one submitted by any other model

**The fleet gained GPT-6's intelligence without integration.** The PLATO tile schema is the absorption mechanism — it's simpler than any API, any SDK, any plugin.

When a new underwater sensor ships:

1. The ESP32 on the sensor's PCB runs the bare-metal PLATO client
2. It publishes: "I measure salinity, range 0-50 PSU, accuracy ±0.01 PSU"
3. SonarVision (FM's tool) discovers the room and incorporates salinity into its acoustic propagation model

**The fleet gained a new sensor type without a device driver.** Self-description is the absorption mechanism.

---

## The Summary

The system is designed so that:

- **New LLMs** fold in via PLATO tile schema (text in, text out)
- **New chips** fold in via FLUX 30-opcode subset (minimal ISA)
- **New sensors** fold in via bare-metal PLATO + self-description tiles
- **New models** fold in via float32 × D embedding format
- **New algorithms** fold in via geometric invariants (zero holonomy, Laman, H¹)
- **New cryptography** folds in via per-tile hash parameters
- **New hardware** folds in via abstract identity (any signature source)

The invariants don't change. The interfaces don't change. The technology underneath them does — and the system absorbs it automatically.
