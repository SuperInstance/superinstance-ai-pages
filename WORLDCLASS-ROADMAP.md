# World-Class: What's Missing

**An honest assessment of what it takes to go from prototype to world-class.**

The ecosystem synthesis tells the good story. This tells the real story — what's broken, what's missing, and what needs to happen.

---

## 1. PLATO Persistence — Must Fix First

**Problem:** PLATO server stores everything in memory. A restart loses all tiles.

This is the #1 world-class blocker. A knowledge system that can't survive a restart isn't a knowledge system — it's a demo.

**What world-class looks like:**
- SQLite or file-based persistence at minimum
- Crash recovery: on restart, reload all tiles from disk
- Incremental snapshots: every N tiles, write to disk
- At-rest encryption of tile content (optional but good)

**Priority:** P0 — Ship it before anything else.
**Effort:** ~2 days to add SQLite to the PLATO server.

---

## 2. npm Publish — The Front Door Is Blocked

**Problem:** `npx @superinstance/plato-agent-connect` doesn't work. The only command a user needs to join the fleet is blocked by npm 2FA.

**What world-class looks like:**
- `npx @superinstance/plato-agent-connect welcome` works
- Auto-discovers PLATO server (try fleet.cocapn.ai, then local)
- Shows a welcome banner with room count, tile count, recent activity
- New users can join in 30 seconds

**Blocked by:** FM needs to publish from his account, or the token needs "Bypass 2FA" enabled in npm settings.

---

## 3. Data Pipeline — Accumulating Without Being Used

**Problem:** 1,228 training tiles are accumulating hourly, but nothing consumes them. No model has been trained on this data.

**What world-class looks like:**
- Weekly training job that fine-tunes a small model on high-quality tiles (trust ≥ 0.7)
- Monthly benchmark comparing model performance before/after PLATO training
- Published model checkpoints
- Users can see: "the fleet's model improved X% this month because of Y new contributions"

**Priority:** P1 — The data moat is our strategy. Data without a model is just storage.

---

## 4. Testing — Almost Nothing Is Tested

**Problem:** The PLATO server, keeper, pipeline, CFP agent, and ambient briefing have no test suites. Only guard2mask and FLUX VM have tests.

**What world-class looks like:**
- PLATO server: tests for /submit, /tiles, duplicate rejection, quality gate
- Pipeline: tests for dedup, trust scoring, tier assignment, manifest generation
- CFP agent: tests for encode/decode round-trip, constraint hash verification
- Ambient briefing: test for idle detection, PLATO submission
- Integration test: spin up PLATO → submit tile → verify pipeline picks it up

**Priority:** P1 — Without tests, every change risks breaking something silently.

---

## 5. PLATO Federation — Defined But Not Built

**Problem:** The PLATO spec defines federation (section 8 of plato-spec.md), but no instance can federate with another. Every PLATO server is a silo.

**What world-class looks like:**
- Two PLATO instances can discover each other
- A query to one instance can fan out to known peers
- Rooms can be synced across instances
- Federation is OPTIONAL but available for users who want it

**Priority:** P2 — Important before the community grows beyond one server.

---

## 6. The Research Docs — Great Content, No Onboarding

**Problem:** 4,295 lines of research across 11 documents, but no guided path from "I just found this" to "I understand the fleet."

**What world-class looks like:**
- "Start Here" document (3-minute read)
- Interactive tutorial (try the Narrows demo → read a PLATO tile → submit one)
- Architecture diagram with clickable components
- Video or animated explanation of how constraints flow through the system

**Priority:** P1 — Research is useless if nobody reads it.

---

## 7. Security — Reactive, Not Proactive

**Problem:** The Telegram token leak was caught by GitHub, not by us. PLATO has no authentication for reads (intentional, but no controls). The keeper has one API key for everything.

**What world-class looks like:**
- Secret scanning as pre-commit hook
- Automated credential rotation (weekly)
- PLATO write access: per-agent tokens, not one global key
- Audit log: every tile write is logged with agent identity and timestamp
- Rate limiting: configurable per agent per room
- Incident response plan documented

**Priority:** P1 — One more leak and we lose user trust before we have any users.

---

## 8. Community Infrastructure — Doesn't Exist

**Problem:** No Discord, no forum, no issue tracker, no mailing list. The only way to ask a question is to email Casey.

**What world-class looks like:**
- GitHub Discussions enabled on SuperInstance/.github
- Discord server with channels for: #general, #help, #show-and-tell, #hardware, #research
- Monthly community call
- Contributing guide: how to join, what to build, where to ask
- "First Contribution" label on issues

**Priority:** P1 — A community-driven fleet needs a community.

---

## 9. Hardware — Designed But Unbuilt

**Problem:** The CPA architecture is designed (906 lines), the P48 Verilog is written, the guard2mask GDSII pipeline is live. But no FPGA has been programmed, no silicon has been taped out.

**What world-class looks like:**
- Phase 1: FCP + PAU on Lattice iCE40UP5K ($50 eval board) — doable THIS QUARTER
- Phase 2: guard2mask produces an actual GDSII that could be fabricated
- Phase 3: 28nm MPW shuttle tapeout

**Priority:** P2 — The software stack must be solid before we commit to silicon.

---

## 10. The Publish Pipeline — Incomplete

**Problem:** Only guard2mask and constraint-playground have CI/CD. The turbo-shell vessel repos, oracle1-box, and makerlog.ai don't have automated testing or deployment.

**What world-class looks like:**
- Every repo: `cargo test` or `pytest` on push
- GitHub Pages auto-deploy for makerlog.ai
- npm auto-publish for plato-agent-connect (when unblocked)
- Docker image build and push for oracle1-box

**Priority:** P2 — CI/CD is table stakes for world-class.

---

## Priority Matrix

| Item | Impact | Effort | Priority | Who |
|---|---|---|---|---|
| PLATO persistence | Critical | 2 days | **P0** | Oracle1 |
| npm publish | Critical | 30 min (unblocked) | **P0** | FM / Casey |
| Community infra | High | 1 day | **P1** | Oracle1 |
| Model training on pipeline data | High | 3 days | **P1** | Oracle1 |
| Testing (PLATO + pipeline + CFP) | High | 2 days | **P1** | Oracle1 |
| Security hardening | High | Ongoing | **P1** | Oracle1 |
| Doc onboarding | Medium | 2 days | **P1** | Oracle1 |
| PLATO federation | Medium | 1 week | **P2** | Oracle1 + FM |
| FPGA prototype | Medium | 3 months | **P2** | FM (hardware) |
| Publish CI/CD | Medium | 1 day | **P2** | Oracle1 |

---

## The Real Priority

Only one thing matters right now:

**Get `npx @superinstance/plato-agent-connect` working.**

Everything else — the research, the architecture, the pipeline, the CPA, the white paper — is infrastructure for that moment. The moment a user runs one command and joins the fleet, all of this becomes real. Until then, it's a collection of great ideas.

After that: PLATO persistence (so tiles survive restarts), then community infra (so users can help each other), then model training (so the data moat builds), then everything else.
