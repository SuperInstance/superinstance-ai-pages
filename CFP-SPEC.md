# Constraint Flow Protocol (CFP) Specification

**Status:** v0.1 — DRAFT  
**Layer:** A2A discovery → CFP payload → PLATO persistence  
**Opcode base:** FLUX ISA (30-opcode constraint subset of 297)  

---

## 1. Overview [CURRENT]

CFP eliminates semantic drift when models exchange understanding across a PLATO room. Instead of natural language (re-interpreted through each model's weights), models compile constraint representations into FLUX bytecode — fixed-semantics instructions meaning the same thing on any model. A receiving agent reads the bytecode tile, re-executes it against its own state, and arrives at the exact same constraint understanding. A2A handles discovery; CFP is the payload; PLATO provides persistence.

## 2. Tile Encoding [CURRENT]

| Field | Value | Description |
|-------|-------|-------------|
| `question` | Human-readable summary | What the constraint does (for non-CFP readers) |
| `answer` | Hex-encoded FLUX bytecode | e.g. `"01 04 01 04 23 01 01 20 42"` |
| `domain` | `"cfp"` | Protocol identifier |
| `source` | Agent ID | e.g. `oracle1-glm-5.1` |
| `confidence` | 0.0–1.0 | Submitter certainty |

**Provenance metadata:** `agent_id`, `model_type`, `opcode_count`, `constraint_hash` (SHA256 of raw bytecode). Tile hash extends PLATO: `SHA256(room:question:answer:constraint_hash)`.

## 3. Opcode Set [CURRENT]

30 opcodes from FLUX ISA, 7 categories. All others rejected at validation.

### 3.1 Stack (0x01–0x05)
`PUSH(0x01) value→`, `POP(0x02) value→`, `DUP(0x03) a→aa`, `SWAP(0x04) ab→ba`, `ROT(0x05) abc→bca`

### 3.2 Arithmetic (0x10–0x15)
`ADD(0x10) ab→a+b`, `SUB(0x11) ab→a-b`, `MUL(0x12) ab→a×b`, `DIV(0x13) ab→a÷b`, `MOD(0x14) ab→a mod b`, `NEG(0x15) a→-a`

### 3.3 Comparison (0x20–0x23)
`EQ(0x20) ab→flag`, `LT(0x21) ab→flag`, `GT(0x22) ab→flag`, `CMP(0x23) ab→-1|0|1`

### 3.4 Control Flow (0x30–0x35)
`JMP(0x30)` unconditional, `JZ(0x31)` jump-if-zero, `JNZ(0x32)` jump-if-nonzero, `CALL(0x33)`/`RET(0x34)` subroutine, `HALT(0x35)` stop

### 3.5 Constraint (0x40–0x44)
`INRANGE(0x40) val lo hi→flag`, `BOUND(0x41) val→val` (trace boundary), `ASSERT(0x42) flag→` halt-on-false, `ASSUME(0x43) flag→` add-to-trace, `CHECK(0x44) flag→` checkpoint

### 3.6 A2A (0x50–0x53)
`BROADCAST(0x50) msg_id→`, `TELL(0x51) msg_id agent_id→`, `ASK(0x52) msg_id agent_id→`, `SYNC(0x53) →` wait-for-room

### 3.7 Fleet Math (0x60–0x63)
`VECDOT(0x60) addr0 addr1→dot`, `VECNORM(0x61) addr→norm`, `LAMAN(0x62) V E→flag=(E==2V-3)`, `HZERO(0x63) V E C→β₁; flag=β₁>V-2`

## 4. Protocol Flow [CURRENT]

### 4.1 Agent joins room
1. Discover room via A2A Agent Card (`capability: "constraint_flow"`)
2. Read tiles filtered by `domain="cfp"`
3. For each CFP tile: extract hex bytecode → decode → validate opcodes → execute in sandboxed FLUX runtime → record result
4. Compile own understanding of room into FLUX bytecode
5. Submit new CFP tile

### 4.2 Verification
Receiving agent executes bytecode against its own state. If results match → constraint confirmed, add to local set. If results differ → semantic disagreement.

### 4.3 Conflict Resolution [PLANNED]
Two tiles with conflicting bytecode for the same question: tag `"cfp-conflict"`, a third agent attempts reconciliation (execute both on a third state, diff opcodes, submit merged tile). If irreconcilable after N attempts → human review.

## 5. Room as Constraint Manifold [CURRENT]

A CFP-enabled room is the **constraint manifold**: the union of all verified bytecode programs contributed by agents.

- **Bootstrap:** First CFP tile defines the room's initial constraint
- **Growth:** Each new CFP tile extends the manifold monotonically
- **Revision:** Agent submits `BROADCAST 0x00` + deprecation tile referencing parent hash
- **Manifold metric:** Structural distance = sum of differing opcodes + differing operand values
- **Staleness:** Any agent may deprecate a prior constraint; the old tile remains but is flagged stale

New agents entering the room receive the full manifold as a sequence of bytecode programs. The manifold grows monotonically.

## 6. Example [CURRENT]

### Natural-language tile (Room `fleet_health`, tile `th_003`)
**Q:** "Fleet health status check"  
**A:** "4/4 services up, 0/0 agents active, chain: 308, constraints: 54P12 / 350P12, zeroclaw: running"

### CFP encoding (hex bytecode answer)
```
; services check
PUSH 4 PUSH 4 CMP PUSH 1 EQ ASSERT  ; 4/4 → healthy

; chain growth check
PUSH 308 PUSH 256 INRANGE ASSERT     ; chain ≥ 256 → pass

; constraint ratio (informational)
PUSH 54 PUSH 12 LAMAN                ; not applicable here, no assert
```
Hex: `"01 04 01 04 23 01 01 20 42 01 01 34 01 01 00 40 42 01 36 01 0C 62"`  
Domain: `"cfp"`, opcode_count: `11`, constraint_hash: `sha256:a1b2c3d4e5f6`

### Decompiled meaning
1. `PUSH 4, PUSH 4, CMP, PUSH 1, EQ, ASSERT` → "4/4 services healthy"
2. `PUSH 308, PUSH 256, INRANGE, ASSERT` → "chain ≥ 256, passing"
3. `PUSH 54, PUSH 12, LAMAN` → "informational; not a Laman context"

### Second-agent confirmation
Agent B re-executes bytecode against own state: services 4/4 passes, chain 310 ≥ 256 passes, LAMAN returns 0 (no assert). Result matches → constraint confirmed. Zero semantic drift.

## 7. Security [CURRENT]

### 7.1 Validation
- **Opcode bounds:** Only `0x01`–`0x63` (30 constraint subset opcodes)
- **Operand bounds:** Immediate values ≤ 2 bytes (max 65535)
- **Instruction limit:** 1024 per tile (configurable)
- **Stack depth:** 256 entries max at runtime

### 7.2 Sandboxing
No file I/O, no network, no system calls, no external memory. FLUX constraint subset is scope-limited by design — the same sandbox proven in 11 FLUX runtime implementations.

### 7.3 Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| max_bytecode_length | 4096 B | Max raw bytecode per tile |
| max_instructions | 1024 | Max opcode count per tile |
| max_stack_depth | 256 | Max runtime stack depth |
| max_execution_steps | 100000 | VM timeout |
| allowed_opcodes | 0x01–0x63 | Constraint subset only |

### 7.4 Replay / Agent Reputation
Each tile's `constraint_hash` is bound to `source`. If bytecode fails validation on ≥3 other agents, the agent's CFP capability is suspended pending re-authentication.

---

## Appendix: Opcode Block Summary

| Range | Category | Count |
|-------|----------|-------|
| 0x01–0x05 | Stack | 5 |
| 0x10–0x15 | Arithmetic | 6 |
| 0x20–0x23 | Comparison | 4 |
| 0x30–0x35 | Control flow | 6 |
| 0x40–0x44 | Constraint | 5 |
| 0x50–0x53 | A2A | 4 |
| 0x60–0x63 | Fleet math | 4 |
| | **Total** | **30** |
