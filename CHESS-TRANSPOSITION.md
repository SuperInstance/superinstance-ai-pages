# Transposition Table Pattern (from Grig's chess engine)

## Original
Grig's Rust chess engine (github.com/vdmo/chess) uses transposition tables to cache position evaluations. A chess position is hashed (Zobrist hashing) and the evaluation is cached. When the same position is reached via a different move sequence, the cached evaluation is returned instead of recomputing.

## Application to guard2mask CSP Solver

Our CSP solver (`guard2mask/src/solver.rs`) uses AC-3 + backtracking + MRV. When backtracking, the same constraint state can be reached via multiple assignment paths. Currently we recompute constraint propagation from scratch.

### The Pattern

```rust
// Chess engine pattern (Grig):
struct TranspositionTable<K, V> {
    entries: Vec<Option<(u64, V)>>,  // Zobrist hash → evaluation
}

fn lookup(&self, hash: u64) -> Option<&V> {
    let idx = hash as usize % self.entries.len();
    self.entries[idx].as_ref()
        .filter(|(h, _)| *h == hash)
        .map(|(_, v)| v)
}

// CSP application:
struct CspTranspositionTable {
    entries: Vec<Option<(u64, ConstraintState)>>,  // hash → pruned domains
}

fn compute_hash(vars: &[Variable], domains: &[Domain]) -> u64 {
    // XOR of (variable_id, domain_bitmask) pairs
    vars.iter().zip(domains.iter())
        .fold(0u64, |h, (v, d)| h ^ hash_pair(v.id, d.bitmask()))
}
```

### Expected Improvement
- 2-5x speedup on complex CSPs (conflict-directed backjumping + cached propagation)
- Zero correctness change — transposition table is a cache, not a logic change

## Usage in guard2mask
1. Compute hash of (variable_ids, current_domains) before propagation
2. Look up in transposition table
3. If found: return cached result
4. If not found: run AC-3, cache result, return

## Credit
Pattern extracted from Grig (vdmo)'s chess engine. Transposition tables are a well-known chess optimization; applying them to CSP constraint propagation is the novel connection.
