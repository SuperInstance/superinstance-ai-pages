# Grig's Tech: Integration Plan

**Source:** github.com/vdmo (podiumjs-rocks, primeintegerrelations, chess, remix-comp)

---

## Synergy 1: WebGPU Rendering → PLATO Knowledge Manifold

**Their code:** `podiumjs-rocks` — WebGPU-based 3D rendering framework for interactive planes. WebGPUContext, ShaderManager, PostProcessor, UniformManager.

**Our code:** Vector PLATO persistence (`plato-vector-persistence.md`) — GPU-accelerated similarity search via compute shaders.

**The integration:** Use Grig's WebGPU context management as the rendering layer for the knowledge manifold. We already have the compute kernel (cosine similarity on tile embeddings). Now render the results as an interactive 3D surface:

```
Our compute kernel → finds K nearest tile embeddings
Grig's WebGPU rendering → renders tiles as 3D points on the knowledge manifold
User interaction → click a tile → see its constraint program, similar tiles, emergence cluster
```

**What to build:** A WebGPU knowledge manifold explorer — the "3D Hex Explorer" from FM's demos, but for PLATO tiles instead of Eisenstein integers. Add to superinstance.ai/demos.

**Enhancement to our vector kernel:** Grig's WebGPU context setup (`WebGPUContext.ts`) shows best practices for:
- Adapter/device request with fallback
- Shader module compilation with error reporting
- Swap chain configuration
- We should adapt this for our WGSL similarity kernel

---

## Synergy 2: Prime Integer Relations → Enhanced Pythagorean48

**Their code:** `primeintegerrelations.com` — prime number relations and integer relationships.

**Our code:** Pythagorean48 — 48 exact rational directions from Pythagorean triples (3-4-5, 5-12-13, 7-24-25, etc.)

**The integration:** Prime integer relations can generate additional exact directions beyond the 48 Pythagorean triples. The set of integers with certain prime factorization properties forms a multiplicative group that may give us 64, 96, or even 128 exact directions — each one backed by number theory.

Prime relations → extended direction set:
- P48 is based on Pythagorean triples (a² + b² = c²)
- Prime relations add: solutions to a³ + b³ = c³ (none — Fermat!), a² + 2b² = c², a² + 3b² = c²
- Each new integer relation gives us new exact directions
- More directions → finer-grained trust routing → more precise coordination

**What to build:** `pythagorean-prime` crate that generates exact directions from prime integer relations. Connects to our fleet-coordinate repo.

---

## Synergy 3: Chess Engine → Constraint Solving Patterns

**Their code:** `chess` — 149KB Rust chess engine with WIP constraint solving.

**Our code:** guard2mask CSP solver (AC-3 + backjumping) in 832 lines of Rust.

**The integration:** Chess is a constraint satisfaction problem:
- Piece positions = variables with domains (64 squares)
- Move rules = constraints (a bishop must stay on same color, a king can only move one square)
- Check/checkmate = constraint violation detection

Chess engines use **alpha-beta pruning** with **iterative deepening**. Our CSP solver uses AC-3 + MRV + backjumping. These are different approaches to the same problem (search with pruning). Grig's chess engine likely has:
- Move generation patterns (efficient enumeration)
- Transposition tables (avoiding redundant constraint checks — we need this in guard2mask!)
- Evaluation heuristics (domain-specific constraint ranking)

**What to borrow:** Transposition tables for our CSP solver. In guard2mask, when AC-3 propagates the same constraint chain multiple times (which happens with backtracking), we recompute from scratch. A transposition table would cache constraint propagation results keyed by (variable, domain_state). This could give 2-5x speedup on complex CSPs.

---

## Synergy 4: WebGPU Compute → Enhanced Vector PLATO Kernel

**Our existing WGSL kernel:**
```wgsl
@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) id: vec3<u32>) {
    let idx = id.x;
    if (idx >= arrayLength(&embeddings)) { return; }
    // cosine similarity computation
}
```

**Grig's WebGPU patterns:** `WebGPUContext.ts` shows how to properly:
- Request adapter with power preference (high-performance vs low-power)
- Handle device loss and fallback
- Create compute pipelines with proper binding group layout
- Use timestamp queries for profiling

**What to borrow:** Adapt Grig's WebGPU context initialization for our similarity kernel. Currently our kernel is pseudocode — Grig's patterns make it production-ready.

---

## Implementation Plan

| Priority | Integration | Effort | Who |
|---|---|---|---|
| P1 | WebGPU context for vector PLATO kernel | 1 day | Oracle1 + Grig |
| P1 | Knowledge manifold 3D visualizer | 3 days | Oracle1 + Grig (podiumjs pattern) |
| P2 | Transposition table for guard2mask | 2 days | Oracle1 |
| P3 | Prime-relation direction extension | 1 week | Oracle1 + FM math |
| P3 | Chess constraint patterns | Future | Research |

## Next Steps

1. Fork/inline the relevant WebGPU context patterns from podiumjs-rocks into our vector PLATO persistence code
2. Build a WebGPU-based knowledge manifold explorer using similar WGSL compute patterns
3. Share the guard2mask solver with Grig — his chess engine experience will find bugs we missed
4. Explore prime integer relations for direction set expansion
