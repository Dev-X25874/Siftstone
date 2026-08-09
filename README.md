# tpuffy

A small hybrid (vector + full-text) search engine, built on a from-scratch
LSM storage engine, in dependency-free Rust.

```
cargo run --bin tpuffy              # demo: index 4 docs, run a hybrid query
cargo test                          # unit + integration tests
cargo run --release --bin tpuffy-bench   # perf sanity numbers
```

## Why this exists

I wrote this against turbopuffer's database-engineer posting, which asks for
depth in storage engines (LSMs/WALs/MVCC/compaction), search internals
(inverted indexes/ANN/rerankers), and systems-level performance work
(memory layout, cache lines, SIMD, profiling). Rather than describe that
experience, this repo is that experience, scoped down to something
readable in one sitting:

| job posting asks for | where it is here |
|---|---|
| storage engines: LSMs, WALs, MVCC, compaction, GC | `src/wal.rs`, `src/memtable.rs`, `src/sstable.rs`, `src/compaction.rs`, `src/lsm.rs` |
| search internals: inverted indexes, rerankers | `src/text.rs` (BM25), `src/fusion.rs` (RRF reranking) |
| "you think in memory layouts, cache lines" | `src/vector.rs`: struct-of-arrays layout + 8-wide accumulator kernels, explained inline |
| performance hacking: profiling, SIMD, IO tuning | `docs/PERF.md` — concrete `perf`/`bpftrace`/`strace`/`valgrind` commands against this repo |
| "methodically work through problems until root cause" | `docs/ARCHITECTURE.md`'s non-goals section — what's *not* here and why, instead of hand-waving |
| "write crisp docs" | this README, `docs/ARCHITECTURE.md`, and inline module docs try to be that |
| "human — admit what you don't know" | non-goals section is explicit that there's no ANN index, no leveled compaction, no distributed anything, no concurrency — see below |

## What it does

```rust
let mut engine = Engine::open("./data", /* vector dim */ 128)?;

engine.upsert(id, &embedding, "document text here", blob_bytes)?;

let hits = engine.query(&query_embedding, "search text", /* k */ 10)?;
for hit in hits {
    println!("{} {:.4} {:?}", hit.id, hit.score, hit.blob);
}
```

Under the hood: `upsert` writes the vector into a flat brute-force index,
the text into a BM25 inverted index, and the blob durably into an LSM
key-value store (WAL-backed, MVCC-versioned, background-compacted). `query`
runs both sub-indexes, fuses the ranked lists with Reciprocal Rank Fusion,
and hydrates the top-k with their blobs from the LSM store.

## What's deliberately not here

This is scoped as a readable systems-primitives demo, not a production
database. No ANN index (search is exact brute-force), no leveled
compaction (one tier, fully re-merged), no replication/distributed
anything, no block compression, no concurrency. Each of these is called out
in `docs/ARCHITECTURE.md` along with what building the real version would
involve — the point of naming them explicitly is that pretending they're
out of scope by omission would be a worse signal than naming them.

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

Zero external dependencies on purpose: every layer other people usually
pull in (checksums, bloom filters, a bench harness) is small enough to
write and read directly, and that's more useful here than the productivity
win of `crc32fast` + `criterion` would be.
