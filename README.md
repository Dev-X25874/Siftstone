# Siftstone

> Dependency-free hybrid search in Rust — LSM storage, BM25 full-text, flat vector index, and RRF fusion, all from scratch.

---

## What it is

A hybrid (vector + full-text) search engine written in Rust with zero external dependencies. Every layer — the WAL, memtable, SSTable format, bloom filter, BM25 inverted index, vector distance kernels, and reciprocal rank fusion — is written from scratch and fits in one file each. Built to be read end-to-end, not run at scale.

---

## What it does

```rust
let mut engine = Engine::open("./data", /* vector dim */ 128)?;

// Store a vector, text, and a blob together under one id
engine.upsert(id, &embedding, "document text here", blob_bytes)?;

// Hybrid query: vector similarity + BM25, fused by reciprocal rank
let hits = engine.query(&query_embedding, "search text", /* top k */ 10)?;
for hit in hits {
    println!("{} {:.4} {:?}", hit.id, hit.score, hit.blob);
}
```

On restart, both the vector index and the inverted index are rebuilt from the LSM store — anything durably written (WAL-synced) survives a crash.

---

## What's inside

| File | What it does |
|------|-------------|
| `src/wal.rs` | Append-only write-ahead log. Length-prefixed records checksummed with FNV-1a. Replay stops at the first bad checksum — a torn tail record is the expected shape of a crash, not something to panic over. |
| `src/memtable.rs` | In-memory sorted write buffer backed by `BTreeMap<InternalKey, Value>`. `InternalKey` sorts `(user_key ASC, seq DESC)` so MVCC snapshot reads are a single forward range scan, not a version index lookup. |
| `src/sstable.rs` | Immutable on-disk sorted run: data block + sparse index + bloom filter + fixed footer, read back-to-front from EOF. Point lookup is: bloom filter (probably absent → skip file) → binary search sparse index → one seek + sequential scan of one block. Same shape as LevelDB sstables, minus block compression and two-level indexing. |
| `src/compaction.rs` | k-way merge over sstable iterators using a min-heap keyed by `InternalKey`'s own `Ord`. Because that ordering already puts the newest version of a key first, GC of shadowed versions and tombstones is a single `if` in the merge loop, not a second pass. |
| `src/lsm.rs` | Ties WAL + memtable + sstables into a KV engine with `put` / `delete` / `get_at(key, snapshot_seq)`. Single-tier compaction — not LevelDB-style leveling. Compaction triggers when the tier count hits 4 files. |
| `src/vector.rs` | Flat brute-force vector search. Storage is struct-of-arrays: one contiguous `Vec<f32>` of length `n * dim`, not `Vec<Vec<f32>>` — sequential prefetch instead of a cache miss per row. Distance kernels use an 8-wide manual accumulator loop so LLVM autovectorizes to AVX2 on `--release -C target-cpu=native`. Top-k selection is a bounded min-heap, O(n log k). Supports dot product, L2, and cosine. |
| `src/text.rs` | Inverted index with Okapi BM25 ranking. Tokenizer is lowercase + split-on-non-alphanumeric — no stemming, no stopwords, no phrase queries. |
| `src/fusion.rs` | Reciprocal Rank Fusion across the two ranked lists. Cosine similarity and BM25 scores aren't on comparable scales, so fusion happens over rank (`1 / (k + rank)`), not over raw score. `k=60` from the original RRF paper. |
| `src/engine.rs` | Public API. Each document's vector, text, and blob are packed into one LSM value so a single WAL-durable write covers the whole upsert. Both indexes are rebuilt from `iter_all()` on open. |

---

## What it does NOT do

- **No ANN index.** Vector search is exact, O(n·dim) per query. HNSW or IVF would be the natural next layer on top of the same SoA storage.
- **No leveled compaction.** One tier, fully re-merged on every compaction. Fine at this scale; wrong past it, since compaction cost becomes O(total data) instead of O(size of the level).
- **No concurrency.** Every structure is `&mut self`, single-threaded by design.
- **No block compression.** Not in the sstable format.
- **No real tokenizer.** Lowercase + split-on-non-alphanumeric. No stemming, no stopwords, no positional/phrase queries.
- **No replication or distributed anything.** Single-node only.

Each of these is documented in `docs/ARCHITECTURE.md` alongside what building the real version would actually involve.

---

## Build & run

Zero dependencies — just `cargo`.

```bash
cargo run --bin tpuffy                    # demo: indexes 4 docs, runs a hybrid query
cargo test                                # unit + integration tests
cargo run --release --bin tpuffy-bench    # throughput/latency numbers

# AVX2 for the vector distance kernels:
RUSTFLAGS="-C target-cpu=native" cargo build --release
```

See `docs/PERF.md` for profiling with `perf`, `bpftrace`, `strace`, and `valgrind`.

---

## Layout

```
src/
  wal.rs         crash-durable append log, FNV-checksummed records
  memtable.rs    in-memory MVCC write buffer
  sstable.rs     immutable on-disk sorted run: data + sparse index + bloom filter
  compaction.rs  k-way merge, version/tombstone GC
  lsm.rs         KV engine tying the above together
  vector.rs      flat SoA vector index, AVX2-friendly distance kernels
  text.rs        inverted index + BM25
  fusion.rs      reciprocal rank fusion
  engine.rs      public hybrid-search API
  main.rs        CLI demo
benches/bench.rs timing benchmarks (hand-rolled, no criterion — zero deps)
tests/           end-to-end integration tests
docs/            architecture writeup + Linux profiling guide
```
