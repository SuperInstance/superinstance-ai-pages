# FPGA Prototype Plan: FCP + PAU on Lattice iCE40

**Strategy:** Prove the Constraint Processor Array on reprogrammable hardware before committing to silicon. FPGA is the proof point that de-risks the ASIC.

**Cost:** $50 (Lattice iCE40UP5K eval board) — no single-point spending.

---

## Phase 1: FCP on iCE40UP5K (Week 1-2)

### What
Synthesize the FLUX Constraint Processor (22K gates, 4-stage pipeline, 30-opcode subset) onto a $50 Lattice iCE40UP5K.

### Status
- FCP microarchitecture designed (906-line CPA document)
- FLUX bytecode format defined (30 opcodes in 7 categories)
- P48 Verilog module written (176 lines, 2-stage pipeline, 2 BRAMs)
- guard2mask produces GDSII from constraints

### What's needed
1. Convert the FCP Verilog from the CPA architecture doc to synthesizable form
2. Map to Lattice iCE40 primitives (SB_IO, SB_GB, BRAM)
3. Synthesize with yosys + nextpnr
4. Program the iCE40UP5K

### Expected results
- FCP runs at ~16 MHz on iCE40 (limited by BRAM timing)
- 16 MHz × 30-opcode pipeline = ~480M constraint ops/sec per chip
- P48 lookup at 1 cycle = 16M trust compositions/sec
- Power: ~20mW (iCE40 typical)

### Build instructions
```bash
# Install open-source FPGA tools
sudo apt install yosys nextpnr-ice40 fpga-icestorm

# Synthesize the FCP
cd flux-hardware/fpga/fcp
yosys -p "synth_ice40 -blif fcp.blif" fcp.v
nextpnr-ice40 --hx8k --json fcp.json --pcf fcp.pcf --asc fcp.asc
icepack fcp.asc fcp.bin

# Program the iCE40
iceprog fcp.bin
```

### Artifacts
- fcp.bin — FPGA bitstream for iCE40UP5K
- fcp.json — Utilization report (LUTs, FFs, BRAMs, DSPs)
- fcp_timing.rpt — Timing report (Fmax, setup/hold slack)

---

## Phase 2: Multi-FPGA Distributed CPA (Week 3-6)

### What
Network 4 iCE40UP5K boards to simulate a 128-core Constraint Processor Array.

### Architecture
```
┌─────────────┐     ┌─────────────┐
│  FPGA 0     │     │  FPGA 1     │
│  Cores 0-31  │◄───►│  Cores 32-63│
│  FCP + PAU  │     │  FCP + PAU │
└──────┬──────┘     └──────┬──────┘
       │                    │
       │    SPI / UART      │
       │    crossbar        │
       │                    │
┌──────┴──────┐     ┌──────┴──────┐
│  FPGA 2     │     │  FPGA 3     │
│  Cores 64-95 │◄───►│  Cores 96-127│
│  FCP + PAU  │     │  FCP + PAU │
└─────────────┘     └─────────────┘
```

### Inter-FPGA protocol
- 8-bit parallel bus at 8 MHz (64 Mbps per link)
- Frame: [4-bit target_core][4-bit opcode][16-bit operand]
- Scoreboard: each FPGA publishes its busy/free status on a shared line
- Arbitration: round-robin token passing (CDC 6600 style)

### Expected results
- 128 cores at 16 MHz = ~2B constraint ops/sec
- Inter-FPGA latency: ~1μs per hop (vs 10μs for network-bound distributed CPA)
- Power: ~80mW (4 × 20mW)

---

## Phase 3: Simulation Farm (Week 7-12)

### What
Scale the CPA simulation in software to 1,000+ cores, running on a workstation or cloud instance.

### How
The Verilog simulator (iverilog or verilator) can simulate multiple CPA instances:
```bash
# Simulate 1,000 CPA cores on a 32-thread workstation
verilator --cc --Mdir obj_fcp --top-module fcp_core --threads 32 \
  fcp_core.v p48_unit.v h1_coprocessor.v --exe sim_driver.cpp
make -C obj_fcp -f Vfcp_core.mk
./obj_fcp/Vfcp_core
```

### Simulation metrics
- 1,000 cores at simulated 16 MHz = ~16B constraint ops/sim-second
- 1 hour of simulation = 57 trillion constraint checks
- Enough to validate: scaling laws, contention patterns, worst-case latency
- Compare: a real 32-core CPA on silicon would take 1 hour to verify in hardware

### What the simulation validates
1. **Scaling law verification** — throughput is O(E/cores) as predicted
2. **Worst-case latency** — what happens when all 1,000 cores contend for the same variable
3. **Fault injection** — what happens when a core produces wrong results (silent data corruption)
4. **Network partition** — what happens when the inter-FPGA links drop
5. **Thermal simulation** — what happens when all 1,000 cores run at max utilization

---

## Phase 4: ASIC Only When Proven (Month 6+)

### Go/no-go criteria
The ASIC tapeout ($500K-$2M NRE) happens ONLY when:

1. **FPGA prototype runs a real application** — not a synthetic benchmark. A real constraint verification workload from the fleet (e.g., boat autopilot or temperature sensor embodiment).

2. **Multiple FPGAs coordinate correctly** — the distributed CPA protocol works across 4+ boards, demonstrating that the architecture scales.

3. **Simulation predicts ASIC performance within 20%** — the Verilog simulation matches the FPGA measurement matches the ASIC prediction.

4. **A customer or grant funds it** — no speculative silicon. The tapeout is paid for by: a defense grant, an automotive safety contract, or a hardware accelerator program.

5. **The $50 FPGA can't do the job** — there's a real application that needs more than 2B ops/sec. If the FPGA is fast enough, stay on FPGA.

---

## Tools Required

| Tool | Cost | Source |
|---|---|---|
| Lattice iCE40UP5K eval board | $50 | DigiKey/Mouser |
| USB Blaster (or built-in programmer) | $0 (included) | — |
| yosys (synthesis) | Free | Open source |
| nextpnr (place-and-route) | Free | Open source |
| icepack/iceprog (bitstream) | Free | Project IceStorm |
| iverilog/verilator (simulation) | Free | Open source |
| 4 × iCE40 for multi-FPGA | $200 | DigiKey/Mouser |
| Breadboard + wires | $15 | Any electronics store |
| **Total** | **$265** | — |

All tools are open source. No vendor lock-in. No license costs.

---

## Success Criteria

1. ✅ FCP on iCE40UP5K: <200mW, verified bytecode execution, matches software VM
2. ✅ PAU on iCE40UP5K: 16M compositions/sec, matches Verilog testbench
3. ✅ 4-board CPA: correct scoreboarding, no protocol errors
4. ✅ 1,000-core simulation: scaling law matches O(E/cores) prediction
5. ✅ ASIC decision: go/no-go based on data, not hope
