# GPU-Centric PLATO Vector Persistence

> Turning PLATO from a tile store into a semantic knowledge engine

---

## 1. Vision

PLATO currently persists tiles as rows in SQLite — a reliable, transactional store that treats every tile as a dumb key-value pair. The next evolution makes PLATO **semantic**: every tile carries a learned embedding vector that captures its meaning. Search becomes similarity in vector space, not string matching. Clusters emerge naturally from the data. Agents discover related knowledge they never explicitly linked.

The GPU-centric vector persistence layer turns PLATO into a **real-time semantic knowledge graph** where:
- Tiles embed once at creation time and are queryable by meaning
- Similarity search runs at GPU speed (<10μs on Jetson Orin for 10K tiles)
- Knowledge emergence, decay, and cross-pollination are detected automatically
- Browser clients perform privacy-preserving vector search via WebGPU
- The entire knowledge base fits in GPU VRAM for embedded deployments

This is PLATO evolving from a journal to a brain.

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                     PLATO Vector Store                         │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Submission Flow:                                             │
│                                                               │
│  Agent submits tile ──► Embedding Pipeline ──► Storage Layer  │
│                                │                               │
│                         ┌──────┴──────┐                       │
│                         │ GPU (Jetson) │                       │
│                         │ CPU (ARM64)  │  (model inference)    │
│                         │ API (cloud)  │                       │
│                         └──────────────┘                       │
│                                │                               │
│                                ▼                               │
│                     ┌──────────────────┐                       │
│                     │  Storage Layer    │                       │
│                     │                   │                       │
│                     │ embeddings.bin   │  (MMAP'd, columnar)   │
│                     │ metadata.jsonl   │  (JSON lines)         │
│                     │ index.faiss      │  (optional, for CPU)  │
│                     └──────────────────┘                       │
│                                │                               │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Query Flow:                                                  │
│                                                               │
│  Query text ──► Embed (same model) ──► Similarity Search ──►  │
│                                              │                 │
│                                    ┌─────────┴─────────┐      │
│                                    │ GPU    │ CPU       │      │
│                                    │ CUDA   │ FAISS     │      │
│                                    │ WGSL   │ HNSW      │      │
│                                    └─────────┴─────────┘      │
│                                              │                 │
│                                              ▼                 │
│                                    Top-K tile IDs              │
│                                              │                 │
│                                              ▼                 │
│                                    Metadata lookup             │
│                                    (CPU, by tile_id)           │
│                                              │                 │
│                                              ▼                 │
│                                    Return results              │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Component Interactions

| Component | Responsibility | Target Hardware | Language |
|-----------|---------------|-----------------|----------|
| Embedding Pipeline | Convert tile text → vector | Jetson (GPU), ARM64 (CPU), Cloud API | Python/CUDA |
| Similarity Engine | Nearest-neighbor search | GPU (CUDA/WGSL) or FAISS CPU | CUDA/WGSL/C++ |
| Storage Layer | Persist + load embeddings | Any (mmap-friendly) | C/Python |
| Metadata Store | Question/answer/origin | Any (JSONL is portable) | JSON |
| Index Manager | Periodically rebuild FAISS | CPU (offline) | Python FAISS |

---

## 3. The Embedding Pipeline

### Model Selection

| Model | Dims | Memory (10K tiles) | Quality | Inference Target |
|-------|------|-------------------|---------|-----------------|
| all-MiniLM-L6-v2 | 384 | 15.36 MB | Good | Jetson GPU, ARM64 CPU |
| BGE-small | 384 | 15.36 MB | Good | Jetson GPU, ARM64 CPU |
| BGE-base | 768 | 30.72 MB | Better | Jetson GPU, Cloud API |
| E5-base | 768 | 30.72 MB | Better | Cloud API |
| OpenAI ada-002 | 1536 | 61.44 MB | Best | Cloud API |
| Cohere embed | 1024 | 40.96 MB | Best | Cloud API |

**For Jetson Orin:** all-MiniLM-L6-v2 running via TensorRT (FP16). The model is ~90MB on disk, ~180MB in FP32 VRAM. With TensorRT FP16 optimization, this drops to ~45MB VRAM leaving ~2GB for embeddings and other workloads. Batch inference at ~500 tiles/second with batch size 32.

**For Oracle Cloud ARM64:** all-MiniLM-L6-v2 via ONNX Runtime (CPU optimized). ~200 tiles/second single-threaded, ~600/sec with 4 threads.

**For ESP32/MCU:** Embedding is too heavy on-device. Two options:
- **Gateway model:** Tile text is sent to a lightweight REST endpoint on the gateway (Jetson or cloud) for embedding
- **LSH fallback:** Use a locality-sensitive hash of the tile text as a fast approximate fingerprint. Not semantic, but sub-10μs and zero dependencies

### Batching Strategy

```python
class EmbeddingBatcher:
    """Batch tile embeddings to maximize GPU utilization."""

    def __init__(self, model, max_batch_size=32, device="cuda"):
        self.model = model
        self.max_batch_size = max_batch_size
        self.device = device
        self.buffer = []

    def add(self, tile_id: str, text: str):
        self.buffer.append((tile_id, text))
        if len(self.buffer) >= self.max_batch_size:
            self.flush()

    def flush(self):
        if not self.buffer:
            return
        ids, texts = zip(*self.buffer)
        embeddings = self.model.encode(list(texts), batch_size=len(texts))
        for tid, emb in zip(ids, embeddings):
            self._append_to_store(tid, emb)
        self.buffer = []

    def _append_to_store(self, tile_id, embedding):
        # Append to embeddings.bin (binary, columnar)
        with open("embeddings.bin", "ab") as f:
            f.write(embedding.astype(np.float32).tobytes())
        # Append to metadata.jsonl
        # (metadata is written separately by the tile submission pipeline)
```

### TensorRT Optimization (Jetson)

```python
# Convert ONNX model to TensorRT FP16 engine
import tensorrt as trt

def build_embedding_engine(onnx_path: str, engine_path: str):
    logger = trt.Logger(trt.Logger.WARNING)
    builder = trt.Builder(logger)
    network = builder.create_network(1 << int(trt.NetworkDefinitionCreationFlag.EXPLICIT_BATCH))
    parser = trt.OnnxParser(network, logger)
    with open(onnx_path, "rb") as f:
        parser.parse(f.read())

    config = builder.create_builder_config()
    config.set_flag(trt.BuilderFlag.FP16)
    config.set_memory_pool_limit(trt.MemoryPoolType.WORKSPACE, 1 << 30)  # 1GB

    engine = builder.build_serialized_network(network, config)
    with open(engine_path, "wb") as f:
        f.write(engine)

# Inference with the engine
import pycuda.driver as cuda
import pycuda.autoinit

def embed_tiles_trt(engine, tiles: list[str], max_batch=32) -> np.ndarray:
    """Batch embed tiles through TensorRT engine."""
    context = engine.create_execution_context()
    # Allocate device memory for input/output
    # Bindings: input tokens (int32[N, 128]), output embeddings (float32[N, 384])
    ...
    return embeddings
```

---

## 4. GPU Similarity Kernel

### CUDA Kernel (Jetson Orin)

```cuda
// plato_similarity.cu
// Computes cosine similarity between a query and all tile embeddings
// Launch configuration: grid(N/256 + 1, 1, 1), block(256, 1, 1)
// Thread count = N (one thread per tile)

__global__ void plato_cosine_similarity(
    const float* __restrict__ query,      // [D] query embedding
    const float* __restrict__ embeddings,  // [N x D] all tile embeddings (row-major)
    float* __restrict__ similarities,      // [N] output: cosine similarities
    int* __restrict__ indices,             // [N] output: tile indices (for sort)
    const int N,
    const int D
) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx >= N) return;

    // Load query into registers (D <= 768, fits in registers)
    // Use vectorized float4 loads for memory coalescing
    float dot = 0.0f;
    float norm_e = 0.0f;

    // Pre-computed: query_norm = sqrtf(dot(query, query)) computed on CPU
    // Pass as kernel argument to avoid redundant computation
    #pragma unroll
    for (int d = 0; d < D; d += 4) {
        float4 q = reinterpret_cast<const float4*>(query)[d / 4];
        float4 e = reinterpret_cast<const float4*>(embeddings + idx * D)[d / 4];
        dot   += q.x * e.x + q.y * e.y + q.z * e.z + q.w * e.w;
        norm_e += e.x * e.x + e.y * e.y + e.z * e.z + e.w * e.w;
    }

    similarities[idx] = dot * rsqrtf(norm_e * query_norm + 1e-8f);
    indices[idx] = idx;
}
```

**Launch configuration for 10K tiles @ 384D:**
- Threads: 10,000 (40 blocks × 256 threads)
- Memory reads: Each thread reads 96 float4s (384/4) = 384 floats = 1536 bytes
- Total DRAM read: 10,000 × 1536 B = 15.36 MB (fits in L2 cache on Orin)
- DRAM bandwidth at 204 GB/s (Orin): ~75μs for the read
- Computation: 10,000 × 384 × 2 FMA = 7.68M FLOPs
- At 1 TFLOPS FP16: ~7.7μs compute
- **Total estimated kernel time: ~85μs** (memory-bound on Orin)

### Top-K Selection via Parallel Reduction

```cuda
// After similarity computation, find top-K indices
// Uses a warp-level bitonic sort for each set of 32 tiles

__global__ void plato_topk(
    const float* __restrict__ similarities,
    const int* __restrict__ indices,
    int* __restrict__ topk_indices,   // [K] output
    const int N,
    const int K
) {
    // Shared memory for block-level top-K
    extern __shared__ float s_scores[];
    int* s_indices = reinterpret_cast<int*>(s_scores + blockDim.x);

    int tid = threadIdx.x;
    int gid = blockIdx.x * blockDim.x + tid;
    if (gid < N) {
        s_scores[tid] = similarities[gid];
        s_indices[tid] = indices[gid];
    } else {
        s_scores[tid] = -FLT_MAX;
        s_indices[tid] = -1;
    }
    __syncthreads();

    // Bitonic sort in shared memory
    for (int size = 1; size < blockDim.x; size *= 2) {
        for (int stride = size; stride > 0; stride /= 2) {
            int partner = tid ^ stride;
            if (partner > tid) {
                bool ascending = (tid & size) == 0;
                if ((ascending && s_scores[tid] < s_scores[partner]) ||
                    (!ascending && s_scores[tid] > s_scores[partner])) {
                    // swap
                    float tmp_s = s_scores[tid];
                    int tmp_i = s_indices[tid];
                    s_scores[tid] = s_scores[partner];
                    s_indices[tid] = s_indices[partner];
                    s_scores[partner] = tmp_s;
                    s_indices[partner] = tmp_i;
                }
            }
            __syncthreads();
        }
    }

    // First K threads write results
    if (tid < K && gid < N) {
        topk_indices[blockIdx.x * K + tid] = s_indices[tid];
    }
}
```

### WGSL Kernel (WebGPU — Browser Clients)

```wgsl
// plato_similarity.wgsl
// Same algorithm, browser-native

@group(0) @binding(0) var<storage, read> query     : array<f32>;
@group(0) @binding(1) var<storage, read> embeddings : array<f32>;
@group(0) @binding(2) var<storage, read_write> similarities : array<f32>;
@group(0) @binding(3) var<uniform> params : Params;

struct Params {
    N : u32,        // number of tiles
    D : u32,        // embedding dimension
    query_norm : f32, // pre-computed norm of query
};

const DIM : u32 = 384u;  // compile-time constant, change per model

@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) id : vec3<u32>) {
    let idx = id.x;
    if (idx >= params.N) {
        return;
    }

    var dot = 0.0f;
    var norm_e = 0.0f;
    for (var d = 0u; d < DIM; d++) {
        let q_val = query[d];
        let e_val = embeddings[idx * DIM + d];
        dot += q_val * e_val;
        norm_e += e_val * e_val;
    }

    let cosine = dot / (sqrt(norm_e) * params.query_norm + 1e-8f);
    similarities[idx] = cosine;
}
```

### Performance Estimates

| Configuration | Hardware | Tile Count | Dims | Query Time |
|--------------|----------|------------|------|------------|
| CUDA kernel | Jetson Orin (2048 cores) | 10K | 384 | ~85μs |
| CUDA kernel | Jetson Orin (2048 cores) | 100K | 384 | ~850μs |
| CUDA kernel | Jetson Orin (2048 cores) | 10K | 768 | ~170μs |
| FAISS IVF (CPU) | Oracle ARM64 (4 cores) | 10K | 384 | ~50μs |
| FAISS HNSW (CPU) | Oracle ARM64 (4 cores) | 100K | 384 | ~100μs |
| FAISS flat (CPU) | Oracle ARM64 (4 cores) | 10K | 384 | ~5ms |
| WGSL (WebGPU) | Browser (RTX 3060) | 10K | 384 | ~200μs |
| WGSL (WebGPU) | Browser (integrated) | 10K | 384 | ~1ms |
| LSH (ESP32) | ESP32-S3 | Any | 32-bit hash | <1μs |

### Memory Footprint (Embeddings Only)

| Tile Count | 384d F32 | 768d F32 | 1536d F32 |
|-----------|----------|----------|-----------|
| 1,000 | 1.5 MB | 3.0 MB | 6.0 MB |
| 10,000 | 15.4 MB | 30.7 MB | 61.4 MB |
| 100,000 | 153.6 MB | 307.2 MB | 614.4 MB |
| 1,000,000 | 1.5 GB | 3.0 GB | 6.0 GB |

Jetson Orin has 8-12GB unified memory. 10K tiles at 384d = 15.4MB. Even 100K tiles at 768d = 300MB remains comfortable.

---

## 5. Storage Format

### Why Not SQLite for Vectors

SQLite stores rows in B-tree pages. Loading all embeddings means traversing the B-tree for every row — O(N log N) I/O for N tiles. Each row read also pulls in metadata fields (question, answer, tile_id strings) that are irrelevant for similarity computation. The result: 10K SQLite row reads for a single vector search.

**A columnar binary file is a single read() call.**

```bash
# Loading 10K embeddings from SQLite (simulated)
$ time sqlite3 plato.db "SELECT embedding FROM tiles" > /dev/null
# ~15-30ms (10K rows, B-tree traversal + text parsing)

# Loading 10K embeddings from binary
$ time dd if=embeddings.bin of=/dev/null bs=1M count=1
# ~0.5ms (single contiguous read)
```

The binary format is also trivially MMAP-friendly. The operating system pages in the embeddings file lazily. On a Jetson with unified memory, the CUDA kernel can access the MMAP'd file directly without a copy.

### Binary Format: embeddings.bin

```
embeddings.bin
┌──────────────────────────┐
│ Header (64 bytes):       │
│  - magic: "PLATEMBED\0"  │  (10 bytes + padding)
│  - version: uint32       │  (4 bytes)
│  - num_tiles: uint32     │  (4 bytes)
│  - dim: uint32           │  (4 bytes)
│  - element_type: uint8   │  (1 byte: 0=float32, 1=float16)
│  - stride: uint32        │  (4 bytes, dim * sizeof(element))
│  - reserved: 35 bytes    │  (padding to 64 bytes)
├──────────────────────────┤
│ Embedding data:          │
│  [float32 × D for tile_0] │
│  [float32 × D for tile_1] │
│  ...                      │
│  [float32 × D for tile_N-1] │
├──────────────────────────┤
│ (Optional) Index region: │
│  FAISS IVF centroids     │
│  or HNSW graph           │
└──────────────────────────┘
```

### Metadata Format: metadata.jsonl

```jsonl
{"tile_id":"abc123","room":"oracle1","question":"What is PLATO persistence?","answer":"A tile store...","created_at":1746662400,"agent_id":"jetsonclaw1","embedding_index":0}
{"tile_id":"def456","room":"oracle1","question":"Define constraint satisfaction","answer":"CSP is...","created_at":1746662460,"agent_id":"jetsonclaw1","embedding_index":1}
```

The `embedding_index` field maps metadata row to the row in embeddings.bin. This allows O(1) lookup: gather top-K indices from similarity search, then do K random-access reads from metadata.jsonl (by line number ≈ embedding_index).

### MMAP Strategy

```python
import numpy as np
import mmap

class VectorStore:
    """GPU-friendly vector store with MMAP'd binary format."""

    def __init__(self, path: str):
        self.path = path
        self._load_header()
        # MMAP the embedding file — kernel can DMA from this
        with open(path, "rb") as f:
            self.mmap = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)
        # Numpy view over the embedding data (no copy)
        data_offset = 64  # skip header
        self.embeddings_np = np.frombuffer(
            self.mmap,
            dtype=np.float32,
            offset=data_offset,
            count=self.num_tiles * self.dim
        ).reshape(self.num_tiles, self.dim)

    def _load_header(self):
        with open(self.path, "rb") as f:
            magic = f.read(10)
            assert magic == b"PLATEMBED\0", "Bad magic"
            self.version = int.from_bytes(f.read(4), "little")
            self.num_tiles = int.from_bytes(f.read(4), "little")
            self.dim = int.from_bytes(f.read(4), "little")

    def get_embeddings_gpu(self):
        """Return pointer to tensor on GPU (Jetson unified memory)."""
        # For Jetson with CUDA unified memory:
        # The MMAP'd region is accessible from GPU via cudaHostRegister
        import pycuda.driver as cuda
        import pycuda.autoinit
        # Register MMAP memory for GPU access (zero-copy on unified memory)
        cuda.host_register(self.embeddings_np, cuda.host_register_flags.DEFAULT)
        return cuda.host_get_pointer(self.embeddings_np)  # device pointer
```

### Why This Works for GPU

On Jetson Orin with unified memory:
1. `embeddings.bin` is MMAP'd and `cudaHostRegister`'d
2. The CUDA kernel reads embeddings directly from the MMAP'd region
3. No `cudaMemcpy` — zero-copy access
4. The kernel launch is ~85μs, total query latency ~150μs including scheduler overhead
5. Results are K tile IDs ∈ O(1) to return to application

---

## 6. Incremental Updates

### Tile Submission (No Index Rebuild)

When a new tile is created:

```python
class VectorStoreWriter:
    """Append-only writer for embeddings.bin and metadata.jsonl."""

    def __init__(self, emb_path: str, meta_path: str):
        self.emb_path = emb_path
        self.meta_path = meta_path
        self.emb_file = open(emb_path, "ab")   # append mode
        self.meta_file = open(meta_path, "a")   # append mode
        self.count = self._count_existing()

    def append(self, tile_id: str, embedding: np.ndarray, metadata: dict):
        """Append a single tile. O(1) write."""
        # Update header count in embeddings.bin
        # (handle carefully — see atomic update below)

        # Write embedding — single fwrite, contiguous
        self.emb_file.write(embedding.astype(np.float32).tobytes())

        # Write metadata — single fwrite
        metadata["tile_id"] = tile_id
        metadata["embedding_index"] = self.count
        self.meta_file.write(json.dumps(metadata) + "\n")
        self.meta_file.flush()

        self.count += 1

    def atomic_flush(self):
        """Ensure both files are flushed atomically."""
        self.emb_file.flush()
        os.fsync(self.emb_file.fileno())
        self.meta_file.flush()
        os.fsync(self.meta_file.fileno())

    def close(self):
        self.atomic_flush()
        self._update_header_count()
        self.emb_file.close()
        self.meta_file.close()
```

**Key insight:** Because the similarity search is a brute-force O(N) scan, there is NO index rebuild needed after each insertion. Every query just scans all N embeddings. New tiles are immediately visible to the next query.

This is the single biggest advantage over indexed approaches (HNSW, IVF) for a write-heavy workload:

| Approach | Insert Time | Query Time | Rebuild Needed |
|----------|------------|------------|----------------|
| Brute-force scan | O(1) append | O(N) | Never |
| FAISS IVF | O(1) append | O(log N) | When |centroids| + 10% |
| HNSW | O(log N) insertion | O(log N) | When graph degrades |
| SQLite + FTS5 | O(1) insert | O(log N) | Never (but no semantics) |

### Header Update (Atomic)

The header of `embeddings.bin` stores `num_tiles`. Appending a new tile needs to update this count. To avoid corruption:

```python
def _update_header_count(self):
    """Atomically update the tile count in the header."""
    with open(self.emb_path, "r+b") as f:
        f.seek(14)  # offset of num_tiles field
        f.write(struct.pack("<I", self.count))
        f.flush()
        os.fsync(f.fileno())
```

For crash safety: maintain a small WAL file `embeddings.wal` that tracks pending counts. On recovery, replay the WAL.

### Periodic FAISS Rebuild (Optional, CPU)

For larger deployments (100K+ tiles) where brute-force becomes slow on CPU:

```python
import faiss

def rebuild_faiss_index(emb_path: str, index_path: str):
    """Rebuild FAISS IVF index from embeddings.bin."""
    store = VectorStore(emb_path)
    dim = store.dim
    nlist = int(np.sqrt(store.num_tiles))  # IVF centroids
    
    quantizer = faiss.IndexFlatIP(dim)  # inner product ≈ cosine if normalized
    index = faiss.IndexIVFFlat(quantizer, dim, nlist, faiss.METRIC_INNER_PRODUCT)
    
    # Normalize embeddings for cosine similarity (L2 normalize)
    faiss.normalize_L2(store.embeddings_np)
    
    index.train(store.embeddings_np)
    index.add(store.embeddings_np)
    index.nprobe = min(nlist // 10, 64)  # search probes
    
    faiss.write_index(index, index_path)
```

Schedule: rebuild when `num_tiles` has grown by 10% since last rebuild. Run as background process on CPU while GPU continues serving queries.

---

## 7. Training Data Generation

With vector embeddings, PLATO becomes a data engine for itself. Every pair of tiles is either a positive or negative example for semantic similarity.

### Automated Pair Mining

```python
class TrainingDataMiner:
    """Mine positive and negative pairs from the embedding space."""

    def __init__(self, store: VectorStore, metadata: list[dict]):
        self.store = store
        self.metadata = metadata
        self.embeddings = store.embeddings_np  # [N, D]

    def mine_pairs(
        self,
        pos_threshold: float = 0.8,
        neg_threshold: float = 0.2,
        max_pairs: int = 10000
    ) -> tuple[list, list]:
        """Mine positive and negative pairs for contrastive learning."""
        # Compute pairwise similarity — O(N²) but GPU-accelerated
        # On GPU: compute dot product matrix in one kernel launch
        N = self.embeddings.shape[0]
        
        # Normalize embeddings
        norms = np.linalg.norm(self.embeddings, axis=1, keepdims=True)
        normalized = self.embeddings / norms
        
        # Similarity matrix (N × N) — use batched matmul
        sim_matrix = normalized @ normalized.T  # [N, N]
        
        positives = []
        negatives = []
        
        for i in range(N):
            for j in range(i + 1, N):
                sim = sim_matrix[i, j]
                if sim > pos_threshold:
                    positives.append({
                        "anchor": self.metadata[i]["tile_id"],
                        "positive": self.metadata[j]["tile_id"],
                        "similarity": float(sim),
                        "anchor_text": self.metadata[i]["question"],
                        "positive_text": self.metadata[j]["question"],
                    })
                elif sim < neg_threshold:
                    negatives.append({
                        "anchor": self.metadata[i]["tile_id"],
                        "negative": self.metadata[j]["tile_id"],
                        "similarity": float(sim),
                    })
        
        # Sample if too many pairs
        if len(positives) > max_pairs:
            positives = random.sample(positives, max_pairs)
        if len(negatives) > max_pairs:
            negatives = random.sample(negatives, max_pairs)
        
        return positives, negatives
```

### Contrastive Learning Loop

The mined pairs train an improved embedding model for PLATO:

```python
import torch
import torch.nn.functional as F

class ContrastivePLATO(torch.nn.Module):
    """Fine-tune embedding model on PLATO tile pairs."""
    
    def __init__(self, base_model_name="sentence-transformers/all-MiniLM-L6-v2"):
        super().__init__()
        from sentence_transformers import SentenceTransformer
        self.encoder = SentenceTransformer(base_model_name)
        self.projection = torch.nn.Linear(384, 128)  # project to contrastive space
    
    def forward(self, anchor, positive, negative):
        a_emb = self.encoder.encode(anchor, convert_to_tensor=True)
        p_emb = self.encoder.encode(positive, convert_to_tensor=True)
        n_emb = self.encoder.encode(negative, convert_to_tensor=True)
        
        a_proj = F.normalize(self.projection(a_emb), dim=-1)
        p_proj = F.normalize(self.projection(p_emb), dim=-1)
        n_proj = F.normalize(self.projection(n_emb), dim=-1)
        
        # Triplet loss: anchor closer to positive than negative
        pos_dist = (a_proj - p_proj).pow(2).sum(-1)
        neg_dist = (a_proj - n_proj).pow(2).sum(-1)
        loss = F.relu(pos_dist - neg_dist + 0.5).mean()
        
        return loss
```

**Why this matters:** PLATO's knowledge base becomes domain-specific training data. A fine-tuned model produces embeddings that are more accurate for PLATO-specific concepts (constraint theory, fleet comms, agent architecture) than any general-purpose embedding model. The system bootstraps itself.

---

## 8. Novel Applications

### A. Real-Time Emergence Detection

Combining the H¹ cohomology emergence detector with vector embeddings:

```python
class EmergenceDetector:
    """Detect new knowledge domains emerging in PLATO."""
    
    def __init__(self, store: VectorStore, h1_detector):
        self.store = store
        self.h1 = h1_detector  # geometric emergence detector
        self.clusters = {}      # {cluster_id: list[tile_ids]}
    
    def detect_emergence(self):
        """Two independent emergence signals."""
        embeddings = self.store.embeddings_np
        
        # 1. H¹ cohomology signal (geometric)
        h1_signal = self.h1.compute()
        
        # 2. Semantic cluster signal (vector space)
        from sklearn.cluster import DBSCAN
        clustering = DBSCAN(eps=0.3, min_samples=3, metric="cosine")
        labels = clustering.fit_predict(embeddings)
        
        new_clusters = set(labels) - set(self.clusters.keys())
        
        signals = {
            "h1_cohomology": h1_signal,
            "new_clusters": len(new_clusters),
            "unclustered_tiles": sum(1 for l in labels if l == -1),
            "total_clusters": len(set(labels) - {-1}),
        }
        
        if h1_signal > 0.7 or new_clusters:
            return {"emergence_detected": True, **signals}
        return {"emergence_detected": False, **signals}
```

**Two signals, one conclusion:**
- H¹ cohomology detects geometric voids → "knowledge is forming a hole"
- Vector clustering detects new semantic clusters → "new domain emerged"
- When both fire simultaneously → "a new knowledge domain formed in the gap"

### B. Agent Persona Matching

```python
class AgentPersona:
    """Represent an agent's knowledge interests as a vector centroid."""

    def __init__(self, store: VectorStore, agent_id: str):
        self.agent_id = agent_id
        self.tile_indices = self._get_agent_tiles()
        self.centroid = self._compute_centroid()

    def _get_agent_tiles(self):
        # Get all tile indices for this agent from metadata
        return [m["embedding_index"] for m in metadata if m["agent_id"] == self.agent_id]

    def _compute_centroid(self):
        if not self.tile_indices:
            return None
        tiles = self.store.embeddings_np[self.tile_indices]
        return tiles.mean(axis=0)

    def similarity_to(self, other_persona: "AgentPersona") -> float:
        if self.centroid is None or other_persona.centroid is None:
            return 0.0
        cos_sim = np.dot(self.centroid, other_persona.centroid)
        cos_sim /= (np.linalg.norm(self.centroid) * np.linalg.norm(other_persona.centroid))
        return float(cos_sim)

    def nearest_tiles(self, query_embedding: np.ndarray, k: int = 5):
        """Find tiles this agent would find most interesting."""
        # Weight similarity by closeness to agent's known interests
        agent_tiles = self.store.embeddings_np[self.tile_indices]
        agent_preference = agent_tiles.mean(axis=0)  # centroid
        
        # Re-rank using personalized query
        personalized = 0.7 * query_embedding + 0.3 * agent_preference
        return top_k_search(personalized, k)
```

**Application:** Route new tiles to the agent whose persona centroid is nearest. Automated knowledge routing without explicit routing rules.

### C. Knowledge Decay Detection

```python
class KnowledgeDecayDetector:
    """Find tiles whose semantic context has shifted."""

    def check_decay(self, store: VectorStore, metadata: list, window: int = 100):
        """Check each tile for semantic drift from its local neighborhood."""
        embeddings = store.embeddings_np
        N = len(embeddings)
        decayed = []

        for i in range(N):
            # Get current nearest neighbors
            query = embeddings[i]
            sims = embeddings @ query  # dot product (normalized)
            neighbors = np.argsort(sims)[-11:-1]  # top 10, exclude self
            
            # Compare with historical nearest neighbors
            # (stored in metadata under "historical_nn" field)
            historical_nn = metadata[i].get("historical_nn", [])
            
            if historical_nn:
                current_nn = set(neighbors.tolist())
                historical_nn_set = set(historical_nn)
                
                # Jaccard distance between current and historical neighbors
                overlap = len(current_nn & historical_nn_set)
                total = len(current_nn | historical_nn_set)
                jaccard = overlap / total if total > 0 else 1.0
                
                if jaccard < 0.5:  # less than 50% overlap
                    decayed.append({
                        "tile_id": metadata[i]["tile_id"],
                        "jaccard": jaccard,
                        "num_neighbors_changed": len(current_nn - historical_nn_set),
                        "neighbors_added": [
                            metadata[j]["tile_id"] for j in (current_nn - historical_nn_set)
                        ],
                    })
            
            # Update historical neighbors
            metadata[i]["historical_nn"] = neighbors.tolist()
        
        return decayed
```

**Tiles flagged as decayed** suggest:
- The knowledge is obsolete (better answers exist now)
- The tile belongs to a different knowledge domain than when created
- New tiles have covered this ground more thoroughly

### D. Cross-Room Knowledge Flow

```python
class CrossRoomRouter:
    """Detect when a tile semantically belongs in a different room."""

    def suggest_room(self, tile_embedding: np.ndarray, metadata: list, rooms: dict):
        """Given a new tile, suggest which room it should go to."""
        # For each room, compute the centroid of its tiles
        room_centroids = {}
        for room_name, room_tile_ids in rooms.items():
            indices = [m["embedding_index"] for m in metadata if m["tile_id"] in room_tile_ids]
            if indices:
                room_embeds = self.store.embeddings_np[indices]
                room_centroids[room_name] = room_embeds.mean(axis=0)
        
        if not room_centroids:
            return None
        
        # Find nearest room centroid
        best_room = None
        best_sim = -1.0
        for room, centroid in room_centroids.items():
            sim = np.dot(tile_embedding, centroid)
            sim /= (np.linalg.norm(tile_embedding) * np.linalg.norm(centroid))
            if sim > best_sim:
                best_sim = sim
                best_room = room
        
        return {"suggested_room": best_room, "confidence": best_sim}
```

This enables **automatic knowledge organization**. An agent working in room A who creates a tile about constraint math will have it suggested for placement in a constraint-theory room. No manual tagging, no category hierarchy to maintain.

---

## 9. Implementation Path

### Phase 1: CPU Vector Search with FAISS (This Month)

**Goal:** Get vector search running on Oracle Cloud ARM64 with existing SQLite tile store.

```python
# Phase 1: Simple FAISS wrapper over existing PLATO SQLite store

class PLATOPhase1:
    def __init__(self, db_path: str, model_name="all-MiniLM-L6-v2"):
        from sentence_transformers import SentenceTransformer
        self.model = SentenceTransformer(model_name)
        self.db_path = db_path
        self._load_tiles()
        self._build_index()

    def _load_tiles(self):
        import sqlite3
        conn = sqlite3.connect(self.db_path)
        self.tiles = conn.execute(
            "SELECT tile_id, room, question, answer FROM tiles"
        ).fetchall()
        conn.close()

    def _build_index(self):
        texts = [t[2] for t in self.tiles]  # questions
        if not texts:
            self.index = None
            return
        self.embeddings = self.model.encode(texts, show_progress_bar=True)
        
        import faiss
        self.index = faiss.IndexFlatIP(384)  # inner product for cosine
        faiss.normalize_L2(self.embeddings)
        self.index.add(self.embeddings)

    def search(self, query: str, k: int = 10):
        q_emb = self.model.encode([query])
        faiss.normalize_L2(q_emb)
        scores, indices = self.index.search(q_emb, k)
        return [
            {
                "tile_id": self.tiles[i][0],
                "room": self.tiles[i][1],
                "question": self.tiles[i][2],
                "answer": self.tiles[i][3],
                "score": float(scores[0][j]),
            }
            for j, i in enumerate(indices[0])
        ]
```

**Deliverables:**
- ✅ FAISS index built from existing SQLite tiles on server startup
- ✅ Semantic search API (`/api/plato/search?q=...`)
- ✅ Bulk re-index on `SIGUSR1` or periodic cron
- ⬜ Columnar binary format (Phase 2)
- ⬜ GPU acceleration (Phase 3)

### Phase 2: GPU Similarity on Jetson (Next Month)

**Goal:** Deploy CUDA similarity kernel on Deckboss (Jetson Orin).

```
┌──────────────────────┐     ┌──────────────────────┐
│  Oracle Cloud ARM64   │     │  Jetson Orin          │
│  (primary server)     │     │  (GPU accelerator)    │
│                       │     │                       │
│  SQLite + embeddings  │────►│  embeddings.bin       │
│  → writes             │     │  (MMAP'd + registered)│
│  → sync to Jetson     │     │  → CUDA kernel        │
│                       │     │  → return K indices   │
└──────────────────────┘     └──────────────────────┘
```

**Changes from Phase 1:**
1. `embeddings.bin` replaces FAISS as the primary storage format
2. Synced from Oracle ARM64 to Jetson via rsync (or NFS)
3. Jetson runs `plato_similarity.cu` via PyCUDA
4. Results returned via ZeroMQ or shared memory

**CUDA server (Jetson side):**

```python
class PlatoGPUServer:
    """GPU-accelerated similarity server on Jetson."""
    
    def __init__(self, emb_path: str):
        self.store = VectorStore(emb_path)
        self.gpu_embeddings = self.store.get_embeddings_gpu()
        # Pre-load CUDA kernel
        with open("plato_similarity.ptx", "r") as f:
            self.module = cuda.module_from_file(f.read())
        self.kernel = self.module.get_function("plato_cosine_similarity")
    
    def query(self, query_embedding: np.ndarray, k: int = 10) -> list[int]:
        N, D = self.store.num_tiles, self.store.dim
        query_gpu = cuda.mem_alloc(D * 4)
        cuda.memcpy_htod(query_gpu, query_embedding.astype(np.float32))
        
        sim_gpu = cuda.mem_alloc(N * 4)
        idx_gpu = cuda.mem_alloc(N * 4)
        
        block = 256
        grid = (N + block - 1) // block
        
        query_norm = np.linalg.norm(query_embedding)
        
        self.kernel(
            query_gpu, self.gpu_embeddings, sim_gpu, idx_gpu,
            np.int32(N), np.int32(D), np.float32(query_norm),
            block=(block, 1, 1), grid=(grid, 1, 1)
        )
        
        # Top-K on GPU
        sim = np.empty(N, dtype=np.float32)
        idx = np.empty(N, dtype=np.int32)
        cuda.memcpy_dtoh(sim, sim_gpu)
        cuda.memcpy_dtoh(idx, idx_gpu)
        
        top_k = np.argsort(sim)[-k:][::-1]
        return idx[top_k].tolist()
```

### Phase 3: WebGPU for Browser Clients (Next Quarter)

**Goal:** End-to-end browser-based similarity search.

```javascript
// plato-webgpu.js — Browser-side PLATO similarity

class PlatoWebGPU {
    constructor(device) {
        this.device = device;
        this.module = null;
        this.embeddings = null;
    }

    async initialize(embeddingsUrl) {
        // Fetch and parse embeddings.bin header
        const response = await fetch(embeddingsUrl);
        const buffer = await response.arrayBuffer();
        const header = new Uint32Array(buffer, 10, 3);
        this.numTiles = header[1]; // skip magic
        this.dim = header[2];

        // Copy embedding data to GPU buffer
        const embSize = this.numTiles * this.dim * 4;
        this.embeddings = this.device.createBuffer({
            size: embSize,
            usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
        });
        this.device.queue.writeBuffer(
            this.embeddings, 0,
            new Float32Array(buffer, 64, this.numTiles * this.dim)
        );

        // Create compute pipeline
        const shader = this.device.createShaderModule({
            code: WGSL_SHADER_SOURCE
        });
        this.pipeline = this.device.createComputePipeline({
            layout: 'auto',
            compute: { module: shader, entryPoint: 'main' }
        });
    }

    async search(queryEmbedding, k = 10) {
        // Upload query
        const queryBuffer = this.device.createBuffer({
            size: this.dim * 4,
            usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
        });
        this.device.queue.writeBuffer(queryBuffer, 0, queryEmbedding);

        // Output buffer
        const simBuffer = this.device.createBuffer({
            size: this.numTiles * 4,
            usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_SRC,
        });

        // Dispatch compute
        const bindGroup = this.device.createBindGroup({
            layout: this.pipeline.getBindGroupLayout(0),
            entries: [
                { binding: 0, resource: { buffer: queryBuffer } },
                { binding: 1, resource: { buffer: this.embeddings } },
                { binding: 2, resource: { buffer: simBuffer } },
            ],
        });

        const encoder = this.device.createCommandEncoder();
        const pass = encoder.beginComputePass();
        pass.setPipeline(this.pipeline);
        pass.setBindGroup(0, bindGroup);
        pass.dispatchWorkgroups(Math.ceil(this.numTiles / 64));
        pass.end();
        this.device.queue.submit([encoder.finish()]);

        // Read results
        const result = await this.device.queue.readBuffer(simBuffer, 0, this.numTiles * 4);
        const sims = new Float32Array(result);
        
        // Top-K (CPU-side since k is small)
        const indices = Array.from({length: this.numTiles}, (_, i) => i);
        indices.sort((a, b) => sims[b] - sims[a]);
        return indices.slice(0, k);
    }
}
```

**Privacy:** The browser never sends embeddings to the server. The entire similarity search runs client-side. The server only serves the static embeddings.bin (which is shareable across all clients anyway — it's compressed knowledge, not user data).

---

## 10. Comparison: SQLite vs Vector DB vs GPU-Accelerated

| Dimension | SQLite (current) | Vector DB (Milvus/Pinecone) | GPU-Accelerated (PLATO) |
|-----------|-----------------|----------------------------|------------------------|
| **Query semantics** | String match (WHERE question LIKE '%X%') | Semantic (cosine similarity) | Semantic (cosine similarity) |
| **Query latency (10K tiles)** | ~1-5ms | ~3-10ms (network) | ~85μs (GPU) / ~50μs (FAISS CPU) |
| **Query latency (100K tiles)** | ~10-50ms | ~5-15ms | ~850μs (GPU) / ~100μs (FAISS HNSW) |
| **Insert latency** | ~1ms (INSERT) | ~5-20ms (network + indexing) | ~0.1ms (append to binary) |
| **Index rebuild** | Never (B-tree) | Yes (IVF/HNSW build) | Never (brute-force) or ~1s/10K (FAISS) |
| **GPU utilization** | None | Varies by provider | Full (custom CUDA kernel) |
| **Embedding at query time** | Not applicable | Provided by client | Same model, all in pipeline |
| **Offline capability** | Yes | No (needs network) | Yes (full local stack) |
| **Privacy (browser)** | Server query | Server query | Fully local (WebGPU) |
| **Storage size (10K @ 384d)** | ~5MB (SQLite) | ~15-30MB (with metadata) | ~15.4MB (binary) |
| **Metadata lookup** | Row by ID | By vector ID | By embedding_index → JSONL |
| **Training data generation** | Manual queries | Via API | Built-in (auto pair mining) |
| **Emergence detection** | Manual (new tags) | Via clustering API | Built-in (H¹ + cluster) |
| **Complexity** | Low (SQL) | Medium (client SDK) | Medium (CUDA/WGSL) |
| **Dependencies** | sqlite3 | milvus/pinecone SDK | numpy, (py)cuda, FAISS, torch |
| **Cost** | Free | $0.10-1.00/1M vectors/month | Free (self-hosted on Jetson) |

### Why Not a Vector DB?

Vector databases (Pinecone, Milvus, Weaviate, Qdrant) are excellent for large-scale production search (millions of vectors, high availability, managed infrastructure). They are **overkill** for PLATO:

1. **PLATO's scale is small:** Even 100K tiles at 384d = 153MB. Fits in any laptop RAM.
2. **PLATO needs tight GPU integration:** Our CUDA kernel is <100 lines. A vector DB abstracts this away.
3. **PLATO needs incremental append:** Vector DBs require index rebuilds or streaming index maintenance. Our brute-force approach never needs a rebuild.
4. **PLATO runs offline:** On a fishing boat. No internet for Pinecone.
5. **PLATO runs on Jetson:** Custom CUDA kernels for embedded GPU. Vector DBs don't target Jetson.
6. **Cost:** Zero marginal cost vs. managed vector DB pricing.

### When to Use SQLite

SQLite remains useful for:
- Transactional metadata (tile creation, edits, deletions)
- Agent state, session management
- The **reference** storage for tile data (source of truth)
- Reporting and analytics (COUNT, GROUP BY, etc.)

**The hybrid approach:**
- SQLite for ACID transactions + metadata queries
- `embeddings.bin` + GPU for vector search
- JSONL for fast metadata lookup by index
- FAISS index (optional) for faster CPU queries on larger datasets

---

## Appendix: Data Flow Sequence Diagram

```
Timeline   Agent         Embedding         Vector Store       Query
          ─────         ─────────         ─────────────       ─────
          │             │                 │                   │
  t=0     │ submit tile │                 │                   │
          │────────────►│                 │                   │
          │             │ embed(text)     │                   │
          │             │───┬───          │                   │
          │             │ GPU│CPU         │                   │
          │             │◄──┴───          │                   │
          │             │ append(bin,json)│                   │
          │             │────────────────►│                   │
          │             │                 │ update MMAP       │
          │             │                 │───┬───            │
          │◄─tile_id────│                 │   │ done          │
          │             │                 │◄──┴───            │
          │             │                 │                   │
  ...later...           │                 │                   │
          │             │                 │                   │ search("vector persistence")
          │             │                 │◄──────────────────│
          │             │                 │ embed(query)      │
          │             │                 │───┬───            │
          │             │                 │ GPU│CPU           │
          │             │                 │◄──┴───            │
          │             │                 │ launch kernel     │
          │             │                 │───┬───            │
          │             │                 │   │ 85μs          │
          │             │                 │◄──┴───            │
          │             │                 │ top-K + metadata  │
          │             │                 │───┬───            │
          │             │                 │   │ lookups        │
          │             │                 │◄──┴───            │
          │             │                 │                   │ top-K results
          │             │                 │───────────────────►│
          │             │                 │                   │
```

---

## Summary

GPU-centric vector persistence transforms PLATO from a tile journal into a semantic knowledge engine. The key innovations:

1. **Columnar binary storage** (not SQLite rows) for GPU-fast loading
2. **Custom CUDA/WGSL kernels** for <100μs similarity search on 10K+ tiles
3. **Zero-copy MMAP** on Jetson unified memory — embeddings are GPU-accessible at file-read speed
4. **Append-only writes** — no index rebuilds, O(1) tile insertion
5. **Built-in training data mining** — PLATO generates its own contrastive learning pairs
6. **Agent personas, knowledge decay, emergence detection** — semantic capabilities impossible with string matching
7. **WebGPU browser clients** — privacy-preserving client-side search

The implementation path is phased: start with FAISS CPU this month, add CUDA kernel next month for Jetson, then WebGPU for browser clients. Each phase is independently useful and backward-compatible.

**PLATO evolves from storing what agents said to understanding what agents meant.**

---

*"Knowledge without meaning is just noise. The vector store gives PLATO a semantic skeleton."*


---

# Spline Anchoring + Vector PLATO Persistence: Synergy

## The Core Insight

The spline-physics repo demonstrates that **multi-agent debate converges to a unique equilibrium** because the physics (beam bending energy) is unique. Five agents with different computational methods all find the same beam shape.

The same principle applies to knowledge: **knowledge tiles in a PLATO room form a smooth semantic manifold, and spline anchoring is the equilibrium search.**

## Spline-Embedding Analogy

| Beam Physics | Knowledge Manifold |
|---|---|
| Beam equilibrium minimizes bending energy U | Knowledge equilibrium minimizes semantic curvature |
| Control points define the spline | Tile embeddings anchor the knowledge surface |
| Quadratic Bezier: B(t) = (1-t)²P0 + 2(1-t)tP1 + t²P2 | Knowledge state at time t is a weighted blend of nearby tiles |
| Consensus IS the physics | Consensus IS the knowledge |
| 5 agents with different priors agree | N tiles should agree on semantic neighborhood |

## Novel Contribution: Spline-Accelerated Vector Search

Instead of storing 10,000 individual tile embeddings for similarity search, store ~100 spline control points that define the knowledge manifold surface:

1. Embed each tile (D=384 for lightweight model)
2. Fit a spline surface through the embedding cloud (via PCA first, then Bezier segments along the principal components)
3. Store only the spline control points (compression: 100:1)
4. At query time: evaluate the spline at the query point to get approximate similarity
5. Only compute exact cosine similarity for the top-K candidates identified by the spline

This is the hardware-native approach: spline evaluation is cheap (a few FMAs per query), while full cosine similarity is O(N*D). On GPU, spline evaluation can be done in shared memory without loading all N embeddings from VRAM.

## Hardware Mapping

| Component | What It Computes |
|---|---|
| Jetson GPU | Spline evaluation + cosine similarity for top-K |
| CPA core | Constraint satisfaction for the knowledge manifold topology |
| P48 unit | Trust routing between knowledge clusters |
| PLATO-in-Silicon | SHA-256 anchoring of spline control points |

## The Killer App

**Real-time knowledge manifold visualization on Deckboss.** A boat's Helm UI shows the knowledge landscape as a 3D spline surface. Tiles are points on the surface. Clusters are knowledge domains. New tiles anchor to the surface and the surface deforms to accommodate them. The captain sees the fleet's knowledge grow in real time as a smooth, navigable terrain.

---
