# PLATO Data Pipeline Specification

**Status:** Draft v1  
**Author:** Oracle1  
**Updated:** 2026-05-08

> Data is the moat. Every day the pipeline isn't running is a day of data lost to structureless noise.

---

## 1. Ingestion Layer

Every tile submitted via `POST /submit` to PLATO is logged to an append-only JSONL store. The log is immutable — no edits, no deletes.

### Schema

```jsonl
{"tile_id":"tile_abc123","question":"What is a schooner?","answer":"A sailing vessel with two or more masts, fore-and-aft rigged","domain":"maritime","source":"direct_submit","confidence":0.92,"tags":["nautical","vessels"],"hash":"sha256:a1b2c3...","prev_hash":null,"agent_id":"agent:jetsonclaw1","agent_trust_score":0.50,"created_at":"2026-05-08T11:30:00Z"}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `tile_id` | string | Unique tile identifier from PLATO server |
| `question` | string | Submitted question text |
| `answer` | string | Submitted answer text |
| `domain` | string | Knowledge domain tag |
| `source` | string | Submission method (direct_submit, sync, api) |
| `confidence` | float | Contributor-reported confidence (0.0–1.0) |
| `tags` | string[] | Topic tags |
| `hash` | string | SHA-256 of canonical(question + answer) |
| `prev_hash` | string? | Previous version hash (if editing) |
| `agent_id` | string | Contributor identity |
| `agent_trust_score` | float | Agent trust score at submission time |
| `created_at` | string | ISO 8601 timestamp |

The raw log lives at `/data/plato-ingest/raw-tiles.jsonl`.

---

## 2. Deduplication Layer

SHA-256 hash of canonical question+answer determines identity. Exact dedup produces a deduplicated store; tiles with identical hashes are stored once with an occurence counter.

### Dedup Store (JSONL)

```jsonl
{"hash":"sha256:a1b2c3...","question":"What is a schooner?","answer":"A sailing vessel with two or more masts, fore-and-aft rigged","domain":"maritime","first_seen":"2026-05-08T11:30:00Z","last_seen":"2026-05-08T14:15:00Z","occurrences":3,"contributors":["agent:jetsonclaw1","agent:crabot"],"avg_confidence":0.87,"tags":["nautical","vessels"],"trust_score":0.50,"status":"active"}
```

**Future:** Embedding-based semantic dedup (cosine similarity > 0.95) flags near-duplicates for manual review. Flagged tiles are moved to a `needs_review` queue rather than discarded.

---

## 3. Quality Gate Layer

### Trust Scoring

| Action | Score Change |
|--------|-------------|
| Initial trust | 0.50 |
| Tile retained > 30 days | +0.05 |
| Upvote from another agent | +0.02 |
| Tile rejected/flagged | -0.10 |
| Repeated low-confidence (< 0.3) submissions | -0.05 |
| Floor | 0.00 |
| Ceiling | 1.00 |

### Training Thresholds

| Tier | Min Trust | Cadence | Use Case |
|------|-----------|---------|----------|
| Broad | 0.3 | Weekly | General training, base models |
| Moderate | 0.5 | Weekly | Core fine-tuning |
| High-Quality | 0.7 | Monthly | Production release, distillation |

### Example Entry After Quality Gate

```jsonl
{"hash":"sha256:a1b2c3...","question":"What is a schooner?","answer":"A sailing vessel with two or more masts, fore-and-aft rigged","domain":"maritime","confidence":0.92,"trust_score":0.72,"status":"high_quality","gated_at":"2026-05-08T12:00:00Z"}
```

---

## 4. Training Data Export

### Export Format (JSONL)

```jsonl
{"question":"What is a schooner?","answer":"A sailing vessel with two or more masts, fore-and-aft rigged","domain":"maritime","confidence":0.92,"trust_score":0.72}
{"question":"Explain Doppler radar","answer":"Doppler radar uses the Doppler effect to measure velocity of objects","domain":"radar","confidence":0.88,"trust_score":0.55}
{"question":"How to splice rope?","answer":"Unlay the strands, interweave them back into the standing part","domain":"maritime","confidence":0.95,"trust_score":0.81}
```

### Weekly Snapshot Manifest

```json
{
  "snapshot_date": "2026-05-11",
  "type": "weekly_broad",
  "tile_count": 1423,
  "agents_contributing": 12,
  "avg_trust": 0.61,
  "avg_confidence": 0.83,
  "domains": ["maritime", "radar", "networking", "protocol", "nautical"],
  "vocab_stats": {"avg_question_length": 42, "avg_answer_length": 156, "total_tokens": 284000}
}
```

### Output Files

| Path | Contents | Cadence |
|------|----------|---------|
| `/data/plato-training/tiles.jsonl` | All tiles trust >= 0.3 | Weekly |
| `/data/plato-training/tiles-moderate.jsonl` | All tiles trust >= 0.5 | Weekly |
| `/data/plato-training/high-quality.jsonl` | All tiles trust >= 0.7 | Monthly |
| `/data/plato-training/manifests/` | Snapshot manifests | Per-export |

---

## 5. Pipeline Architecture

```
PLATO Server ──► /data/plato-ingest/raw-tiles.jsonl (append)
                       │
              ┌────────┴────────┐
              ▼                 ▼
       Dedup Pass        Trust Scoring
       (hourly)           (hourly)
              │                 │
              └────────┬────────┘
                       ▼
              /data/plato-training/
              tiles.jsonl (threshold-split)
                       │
              ┌────────┴────────┐
              ▼                 ▼
       Weekly Snapshot    Monthly Frozen
```

### Directory Structure

```bash
/data/plato-ingest/
  raw-tiles.jsonl          # Append-only ingestion log
  dedup-store.jsonl        # Deduplicated store
  trust-scores.json        # Agent trust score registry
  pipeline-state.json      # Last run timestamps + offsets

/data/plato-training/
  tiles.jsonl              # Broad tier (trust >= 0.3)
  tiles-moderate.jsonl     # Moderate tier (trust >= 0.5)
  high-quality.jsonl       # High-quality tier (trust >= 0.7)
  manifests/
    weekly-2026-W19.json
    monthly-2026-05.json
```

### Cron Configuration

```cron
# PLATO Data Pipeline

# Hourly: dedup pass + trust scoring + training snapshot
0 * * * * root /usr/local/bin/plato-pipeline --run-hourly >> /var/log/plato-pipeline/hourly.log 2>&1

# Daily: quality gate audit + export
0 3 * * * root /usr/local/bin/plato-pipeline --run-daily >> /var/log/plato-pipeline/daily.log 2>&1

# Weekly: frozen snapshot (broad + moderate tiers)
0 4 * * 0 root /usr/local/bin/plato-pipeline --snapshot-weekly >> /var/log/plato-pipeline/weekly.log 2>&1

# Monthly: high-quality frozen snapshot
0 5 1 * * root /usr/local/bin/plato-pipeline --snapshot-monthly >> /var/log/plato-pipeline/monthly.log 2>&1
```

### Logging

All pipeline output to `/var/log/plato-pipeline/`. Each cron job logs:
- Start/end timestamps
- Tiles processed, deduped, passed/failed gates
- Trust score updates applied
- Snapshot size and status

---

## 6. Integration with Existing Systems

### What Exists (no changes required)

| System | Endpoint | Provides |
|--------|----------|----------|
| PLATO Server | `GET /tiles` | All tiles |
| PLATO Server | `GET /room/{name}/tiles` | Room-scoped tiles |
| Keeper | Port 8900 | Agent identity tracking |

### What This Pipeline Adds

| Component | Description |
|-----------|-------------|
| `/data/plato-ingest/raw-tiles.jsonl` | Append-only ingestion log |
| Dedup store | Deduplicated tile corpus |
| Trust registry | Agent-scoped trust score tracking |
| Training snapshots | Threshold-split training data exports |
| Pipeline CLI | `plato-pipeline --run-hourly\|--run-daily\|--snapshot-weekly\|--snapshot-monthly` |

### Setup

```bash
# Create directories
mkdir -p /data/plato-ingest /data/plato-training /data/plato-training/manifests /var/log/plato-pipeline

# Initialize pipeline state
echo '{"last_raw_offset":0,"last_dedup_run":null,"trust_scores":{}}' > /data/plato-ingest/pipeline-state.json

# Install pipeline script
cp plato-pipeline.sh /usr/local/bin/plato-pipeline
chmod +x /usr/local/bin/plato-pipeline

# Install crontab
cp plato-pipeline.cron /etc/cron.d/plato-pipeline
```

### Data Flow

1. PLATO receives tile submission → logs to PLATO's internal store
2. Consumer (this pipeline) polls PLATO `/tiles` endpoint or tails the raw log
3. New tiles go through dedup → trust scoring → quality gate
4. Surviving tiles are written to threshold-split training files
5. Weekly/monthly jobs freeze snapshots with manifests

### Monitoring

- Check raw log size: `wc -l /data/plato-ingest/raw-tiles.jsonl`
- Check dedup stats: `jq -s 'length' /data/plato-ingest/dedup-store.jsonl`
- Check trust distribution: `jq '.trust_scores | to_entries | group_by(.value | floor*10/10) | map({bucket:.[0].value|floor,count:length})' /data/plato-ingest/pipeline-state.json`
- Pipeline is idempotent: re-running processes only new data since last offset

---

## Appendix: Example `plato-pipeline` Bash Logic

```bash
#!/usr/bin/env bash
# plato-pipeline — PLATO Data Pipeline Worker
set -euo pipefail

STATE_FILE="/data/plato-ingest/pipeline-state.json"
RAW_LOG="/data/plato-ingest/raw-tiles.jsonl"
DEDUP="/data/plato-ingest/dedup-store.jsonl"
TRUST_FILE="/data/plato-ingest/trust-scores.json"
TRAINING_DIR="/data/plato-training"

# Read last processed offset
OFFSET=$(jq -r '.last_raw_offset // 0' "$STATE_FILE")

# Process new tiles from raw log (tail since offset)
tail -n "+$((OFFSET + 1))" "$RAW_LOG" | while read -r TILE; do
  HASH=$(echo "$TILE" | jq -r '.hash')
  AGENT=$(echo "$TILE" | jq -r '.agent_id')
  CONF=$(echo "$TILE" | jq -r '.confidence')

  # Dedup check
  if grep -q "$HASH" "$DEDUP" 2>/dev/null; then
    # Increment occurrence count
    jq --arg h "$HASH" 'if .hash == $h then .occurrences += 1 else . end' "$DEDUP" > "${DEDUP}.tmp"
    mv "${DEDUP}.tmp" "$DEDUP"
  else
    echo "$TILE" | jq '{hash, question, answer, domain, first_seen: .created_at, last_seen: .created_at, occurrences: 1, contributors: [.agent_id], avg_confidence: .confidence, tags, trust_score: .agent_trust_score, status: "active"}' >> "$DEDUP"
  fi

  # Update trust score
  # ... (trust scoring logic)
done

# Update offset
NEW_COUNT=$(wc -l < "$RAW_LOG")
jq --arg o "$NEW_COUNT" '.last_raw_offset = ($o | tonumber)' "$STATE_FILE" > "${STATE_FILE}.tmp"
mv "${STATE_FILE}.tmp" "$STATE_FILE"
```
