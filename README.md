# tpuffy

A small hybrid (vector + full-text) search engine, built on a from-scratch
LSM storage engine, in dependency-free Rust.

```
cargo run --bin tpuffy                    # demo: index 4 docs, run a hybrid query
cargo test                                # unit + integration tests
cargo run --release --bin tpuffy-bench    # perf sanity numbers
```

## What it does

```rust
let mut engine = Engine::open("./data", /* vector dim */ 128)?;

engine.upsert(id, &embedding, "document text here", blob_bytes)?;

let hits = engine.query(&query_embedding, "search text", /* k */ 10)?;
for hit in hits {
    println!("{} {:.4} {:?}", hit.id, hit.score, hit.blob);
}
```

`upsert` writes the vector into a flat brute-force index, the text into a
BM25 inverted index, and the vector/text/blob together into an LSM
key-value store (WAL-backed, MVCC-versioned, background-compacted).
`query` runs both sub-indexes, fuses the ranked lists with Reciprocal Rank
Fusion, and hydrates the top-k with their blobs from the LSM store. Both
indexes are rebuilt from the LSM store on open, so a restart doesn't lose
anything that was durably written.

## Architecture

```
                    ┌─────────────────────┐
                    │       Engine         │  upsert / delete / query
                    └──────────┬───────────┘
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
      ┌───────────────┐ ┌──────────────┐ ┌──────────────┐
      │ VectorIndex    │ │ InvertedIndex│ │  LsmEngine    │
      │ (flat, SoA)    │ │ (BM25)       │ │  (documents)  │
      └───────────────┘ └──────────────┘ └──────┬────────┘
                                                  │
                          ┌───────────────────────┼───────────────────────┐
                          ▼                        ▼                       ▼
                    ┌──────────┐            ┌──────────────┐       ┌─────────────┐
                    │   WAL     │            │  MemTable     │       │  SSTable(s)  │
                    │ (durability)│          │ (sorted MVCC) │       │ (immutable)  │
                    └──────────┘            └──────────────┘       └──────┬───────┘
                                                                            │
                                                                     ┌──────▼──────┐
                                                                     │ Compaction   │
                                                                     │ (k-way merge)│
                                                                     └─────────────┘
```

* **WAL** (`src/wal.rs`) — length-prefixed records with an FNV-1a checksum.
  Replay stops at the first bad checksum or short read instead of erroring,
  since a torn tail record is the expected shape of a crash mid-write.
* **MemTable** (`src/memtable.rs`) — `BTreeMap<InternalKey, Value>`.
  `InternalKey` sorts `(user_key ASC, seq DESC)`, which makes MVCC point
  lookups a single forward range-scan.
* **SSTable** (`src/sstable.rs`) — data block + sparse index + bloom filter
  + footer, read back-to-front from EOF. A lookup is: bloom filter
  (probably-absent → skip the file) → binary search the in-memory sparse
  index → one seek + sequential scan of one block.
* **Compaction** (`src/compaction.rs`) — k-way merge over sstable iterators
  using a min-heap keyed by `InternalKey`'s own `Ord`, so old-version and
  tombstone GC falls out as a single `if` rather than a second pass.
* **LsmEngine** (`src/lsm.rs`) — wires the above into `put` / `delete` /
  `get_at(key, snapshot_seq)`.
* **VectorIndex** (`src/vector.rs`) — flat brute-force search, struct-of-
  arrays layout, 8-wide manual accumulator in the distance kernels so LLVM
  can autovectorize on stable Rust, bounded min-heap for O(n log k) top-k.
* **InvertedIndex** (`src/text.rs`) — postings lists + Okapi BM25.
* **fusion** (`src/fusion.rs`) — Reciprocal Rank Fusion across the two
  ranked lists, since cosine similarity and BM25 scores aren't on
  comparable scales.

See `docs/ARCHITECTURE.md` for the full design writeup, including what's
deliberately out of scope (no ANN index, no leveled compaction, no
replication, no concurrency) and why. See `docs/PERF.md` for profiling
this repo on Linux with `perf`, `bpftrace`, `strace`, and `valgrind`.

## Layout

```
src/
  wal.rs         crash-durable append log, checksummed records
  memtable.rs    in-memory MVCC-sorted write buffer
  sstable.rs     immutable on-disk sorted run: data + sparse index + bloom filter
  compaction.rs  k-way merge, old-version/tombstone GC
  lsm.rs         ties the above into a KV engine
  vector.rs      flat SoA vector index, cache/SIMD-conscious distance kernels
  text.rs        inverted index + BM25
  fusion.rs      reciprocal rank fusion
  engine.rs      public API
  main.rs        CLI demo
benches/bench.rs manual timing benchmarks (no criterion — zero-dep by design)
tests/           end-to-end integration tests
docs/            architecture + profiling notes
```

## Non-goals

This is scoped as a readable systems-primitives project, not a production
database:

* **No ANN index** — vector search is exact brute-force, O(n·dim) per query.
* **No leveled compaction** — one tier, fully re-merged on every compaction.
* **No replication or distributed anything** — single-node only.
* **No block compression**, no two-level sstable index.
* **No real tokenizer** — lowercase + split-on-non-alphanumeric, no
  stemming, no stopwords, no phrase queries.
* **No concurrency** — every structure is `&mut self`, single-threaded by
  design.

Each of these is expanded on in `docs/ARCHITECTURE.md` along with what
building the real version would involve.

## Dependencies

Zero. Checksums, bloom filters, and the bench harness are all small enough
to write and read directly rather than pull in `crc32fast` + `criterion`.

## License

MIT — see `LICENSE`.
