# Engineering Philosophy: Lessons from Grig's Code

**Source:** github.com/vdmo (podiumjs-rocks, chess, primeintegerrelations)
**Analysis:** 6 code files, ~600 lines across 3 systems
**Key insight:** Different lived experiences produce different engineering languages. Casey's background is commercial fishing (mechanical constraints, crew management). Grig's background is WebGPU/TypeScript/chess (allocation constraints, error propagation, cache discipline). Both encode their domain's constraints into their code.

---

## Lesson 1: Defensive Initialization with Progressive Enhancement

**Grig's pattern** (WebGPUContext.ts):
```typescript
async initialize(): Promise<boolean> {
    if (!navigator.gpu) { return false; }           // Check feature availability
    this.adapter = await navigator.gpu.requestAdapter(
        { powerPreference: 'high-performance',       // Request best first
          forceFallbackAdapter: false });            
    if (!this.adapter) { return false; }             // Fall through on failure
    
    this.device = await this.adapter.requestDevice();
    if (!this.device) { return false; }
    
    this.context = this.canvas.getContext('webgpu');
    if (!this.context) { return false; }             // Every step checked
    
    return true;  // Caller knows what to do with false
}
```

Every step checks its precondition. No exceptions. No silent failures. The caller gets a clean `boolean` and decides what to do.

**Our current pattern** (PLATO server):
```python
signed_tiles[signed.tile_id] = signed  # Could fail silently
tile_store.save_tile(...)  # Could fail, we don't check
```

**What to apply:**
- PLATO submission should return `(success: bool, reason: string)` at every step
- Pipeline retry should have the same structure as WebGPU's `powerPreference: 'high-performance' | 'low-power'` — a hierarchy of fallback strategies
- `plato_client.c` should follow this exact pattern: `plato_init()` → check → `plato_publish()` → check → return result code

---

## Lesson 2: Cache Everything, Fail Informatively

**Grig's pattern** (ShaderManager.ts):
```typescript
private shaderModules = new Map<string, GPUShaderModule>();
private pipelines = new Map<string, GPURenderPipeline>();
private bindGroupLayouts = new Map<string, GPUBindGroupLayout>();
```

Three separate caches, each keyed by a string label. When compilation fails, the error message includes the label, the line number, and the message text:
```
Shader compilation messages for "terrain_plane_wgsl":
  error: 'var' declaration must have a type
    Line 12: @group(0) @binding(0) var<uniform> uniforms;
```

**Our current pattern:**
- `cfp.py` FlattenQueue decodes CFP bytecode from scratch every query — no caching
- `plato-pipeline.py` trust_score loop recomputes from scratch every hour (reprocessing all tiles)
- `solver.rs` AC-3 propagation runs from scratch on every backtrack — no transposition table

**What to apply:**
- Add transposition tables to guard2mask CSP solver (grid/chess pattern)
- Add bytecode decompilation cache to CFP library (keyed by tile_hash → decoded program)
- Add incremental trust score updates to pipeline (only process tiles since last run)

---

## Lesson 3: Explicit Lifecycle, No Hidden State

**Grig's pattern** (Plane.ts):
```typescript
class Plane {
    createGeometry() → PlaneGeometry    // Pure data
    createBuffers()                     // Side effect on GPU
    bind(renderPass)                    // Use in context
    draw(renderPass)                    // Execute
    destroy()                           // Cleanup
}
```

Each method is independent. `createGeometry()` returns data without side effects. `createBuffers()` has side effects but is separate from geometry calculation. `destroy()` is explicit, not a destructor.

**Our current pattern:**
- PLATO server starts rooms implicitly on first tile submission
- Pipeline creates training directories on first run but doesn't clean them up
- Ambient briefing creates state file on first idle detection, never explicitly clears it
- Keeper stores tokens in memory with no flush mechanism

**What to apply:**
- PLATO should have explicit `create_room()` and `destroy_room()` methods (currently implicit)
- The data pipeline should have `open()` / `close()` semantics for its JSONL files
- Every script should have a documented lifecycle: `init()` → `run()` → `cleanup()` → `destroy()`

---

## Lesson 4: Static Factory Methods for Common Patterns

**Grig's pattern** (ShaderManager.ts):
```typescript
static getBasicTextureShader(): ShaderSource { /* vertex + fragment WGSL */ }
static getUniformShader(): ShaderSource { /* with MVP matrix, time uniform */ }
```

Pre-built templates for common use cases. No need to write WGSL boilerplate for a textured quad — just call `getBasicTextureShader()` and it works. The templates are documented, tested, and optimized.

**Our current pattern:**
- Every FLUX constraint program is written from scratch
- guard2mask has no built-in constraint templates
- CFP protocol has no pre-built opcode sequences for common patterns (like "check range then assert")

**What to apply:**
- Create `ConstraintTemplate` module in guard2mask:
  - `check_range(variable, min, max)` → pre-built AC-3 compatible constraint
  - `check_implication(antecedent, consequent)` → pre-built implication
  - `check_equality(variable, target)` → pre-built equality
  - `check_rigidity(E, V)` → pre-built Laman count check
- These are the "basic texture shaders" of constraint programming

---

## Lesson 5: Interleaved Data for Hardware Efficiency

**Grig's pattern** (Plane.ts):
```typescript
// Interleaved vertex buffer: [position.x, position.y, position.z, uv.u, uv.v]
const vertexData = new Float32Array(vertexCount * 5);
for (let i = 0; i < vertexCount; i++) {
    vertexData[i * 5 + 0] = positions[i * 3 + 0];  // x
    vertexData[i * 5 + 1] = positions[i * 3 + 1];  // y
    vertexData[i * 5 + 2] = positions[i * 3 + 2];  // z
    vertexData[i * 5 + 3] = uvs[i * 2 + 0];        // u
    vertexData[i * 5 + 4] = uvs[i * 2 + 1];        // v
}
```

Interleaving positions and UVs means one memory read fetches all the data for a vertex. Separate buffers would require two memory reads. On GPU memory bandwidth is the bottleneck — interleaving is free performance.

**Our current pattern:**
- Embedding storage: `embeddings.bin` (separate) + `metadata.jsonl` (separate) = two reads per tile
- CSP solver: constraint data (separate) + variable state (separate) = multiple reads per propagation

**What to apply:**
- Interleave tile metadata with embedding vectors: `[embedding_0, tile_id_0, timestamp_0, embedding_1, tile_id_1, ...]`
- This means one GPU kernel launch loads everything needed for similarity + metadata
- In the CSP solver, interleave constraint definition with its current state for cache-friendly propagation

---

## Lesson 6: The ES Module Pattern — Explicit References

**Grig's pattern:**
```typescript
import { WebGPUContext } from '../core/WebGPUContext.js';  // Explicit .js extension
```

No build-time path aliases. No webpack resolve magic. Every import is a real path that resolves in the runtime. This makes the code greppable, refactorable, and debuggable without understanding the build system.

**Our current pattern:**
```python
from equipment.plato import PlatoClient  # Implicit import path
from plato_provenance import TileSigner  # Third-party crate import
```

**What to apply:**
- Our imports are already explicit (Python doesn't have implicit imports). The lesson is about *readability* — every import tells you exactly where the thing comes from. Keep it that way.
- The deeper lesson: **don't abstract away the dependency graph.** Let developers see what depends on what.

---

## The Meta-Lesson: Constraints Shape Engineering Language

Grig builds WebGPU rendering engines. His constraints:
- GPU memory is finite and must be explicitly managed
- Shader compilation can fail with cryptic GPU driver errors
- `requestAnimationFrame` gives you 16ms to render a frame
- The WebGPU API is low-level and has no standard library

These constraints produce his engineering language:
- Defensive initialization (the GPU might not exist)
- Aggressive caching (compilation is expensive)
- Explicit lifecycle (allocation/deallocation must match)
- Static factories (common patterns shouldn't be rewritten)

Casey builds boats and fishing operations. His constraints:
- Salt water destroys everything
- Seawater electrical ground is not always reliable
- You can't reboot the ocean
- Crew members have different skills and experience levels

These constraints produce a different engineering language:
- Mechanical fail-safes (physical override on every automated system)
- Mixed-criticality systems (the rudder solenoid and the GPS can fail independently)
- Crew training (the dojo model — agents level up through experience)

Our constraint system sits at the intersection of these two languages. The fleet's math (Laman rigidity, H¹ cohomology) is the lingua franca that lets both perspectives contribute without translating.
