# search-engine-from-scratch

A full-text and vector search engine written from scratch in TypeScript on Node.js, with no search libraries. It has an inverted index stored as immutable segments, compressed postings, BM25 ranking with Block-Max WAND top-k retrieval, HNSW vector search and hybrid ranking. It will be benchmarked against Postgres full-text search (GIN), pgvector and Lucene on public datasets.

> **Status:** work in progress. The design and roadmap are below. Results will be added here as each part lands.

## Why

`SELECT ... WHERE tsv @@ query ORDER BY ts_rank(...) LIMIT 10` and `ORDER BY embedding <=> $1 LIMIT 10` look simple. Underneath, a search engine makes a series of decisions that set its speed and the quality of its results:

- **Why the index is a set of immutable segments.** Rewriting one big index on every insert is too slow. New documents are buffered in memory and written out as small read-only segments, which are merged in the background. Deletes become tombstones until a merge drops them, and readers search a fixed snapshot of segments without taking locks.
- **Why postings are compressed.** A posting list is a sorted list of document IDs. Storing the gaps between IDs, bit-packed in blocks of 128, makes the index several times smaller and faster to scan, because memory bandwidth, not arithmetic, is the bottleneck.
- **Why top-k does not have to score every match.** A query for "postgres replication slot" matches millions of documents that contain "postgres". WAND keeps an upper bound on each term's best possible score and skips documents that cannot reach the current top 10. Block-Max WAND stores that bound per block of 128 postings, so it skips far more.
- **Why Postgres ranking gets slow.** A GIN index finds every matching row, but it cannot return them in score order. `ts_rank` must then be computed for every match before the sort, so a common term means scoring the whole match set.
- **Why vector search is approximate.** An exact nearest-neighbour search compares the query with every vector. HNSW walks a layered proximity graph instead, and its `efSearch` parameter trades recall for speed. The only honest way to report it is as a recall-versus-throughput curve.
- **Why hybrid search.** Keyword search finds exact terms such as error codes, names and IDs. Vector search finds paraphrases. Reciprocal Rank Fusion combines the two ranked lists using only ranks, so the two scores never need calibrating against each other.

This project builds each of those pieces by hand and measures them against the systems people actually use.

## Design

| Component | What it does |
|---|---|
| **Analyzer** | Unicode NFKC normalisation, lowercasing, tokenisation, stop words and a Porter stemmer written by hand |
| **Indexing buffer** | New documents go into an in-memory buffer, which is flushed to a new segment when it reaches a size limit |
| **Segment format** | Per segment: a term dictionary (sorted, front-coded blocks with a sparse in-memory index), postings (doc-ID gaps bit-packed in blocks of 128, term frequencies and positions), document lengths, stored fields, and a CRC32 checksum per file |
| **Skip data** | For each postings block: the last doc ID, its file offset and the block's maximum BM25 score |
| **Commits** | A commit file lists the live segments. It is written to a temp file, fsynced and renamed, so a crash always leaves the last complete commit readable |
| **Deletes and merges** | Deletes are per-segment tombstone bitsets. A tiered merge policy merges segments of similar size and drops deleted documents |
| **Query parser** | Terms, AND / OR / NOT, quoted phrases and field filters |
| **Query execution** | Document-at-a-time iterators with `next` and `advance`; exhaustive OR, WAND and Block-Max WAND for top-k; phrase matching on positions |
| **Scoring** | BM25 with configurable `k1` and `b`, using collection statistics summed across segments |
| **Vectors** | One HNSW graph per segment over `Float32Array`s (parameters `M`, `efConstruction`, `efSearch`), cosine and L2 distance, optional int8 scalar quantisation |
| **Hybrid** | Reciprocal Rank Fusion of the BM25 and vector result lists, with the same filters applied to both |
| **Concurrency** | Segments are searched in parallel on `worker_threads`; each search sees a point-in-time snapshot while indexing and merges continue |
| **Server** | A JSON HTTP API on `node:http`: index, delete, search and stats |

## Benchmarks (planned)

Every number will come with the script that produced it, the hardware and the exact settings.

- **Ranking quality:** nDCG@10 on BEIR datasets (SciFact, NFCorpus, FiQA-2018, TREC-COVID), compared with Lucene's BM25 run through Pyserini. The goal is to match Lucene, which shows the analyzer and BM25 are right.
- **Query speed:** MS MARCO passage ranking (8,841,823 passages, 6,980 dev queries). It reports p50 and p99 top-10 latency for exhaustive scoring, WAND and Block-Max WAND, and compares them with Postgres GIN + `ts_rank_cd` on the same passages.
- **Index build:** documents per second, index size in bytes per posting, and merge cost.
- **Vector search:** `sift-128-euclidean` (1M vectors) and `glove-100-angular` (1.18M vectors) from ann-benchmarks. It reports recall@10 against QPS across a sweep of `efSearch`, build time and memory, compared with pgvector's HNSW (`m = 16`, `ef_construction = 64`, its defaults) at matched recall.
- **Hybrid:** nDCG@10 for BM25 alone, vectors alone (embeddings from an open sentence-embedding model such as `BAAI/bge-small-en-v1.5`) and RRF fusion, on SciFact and FiQA-2018.
- **Crash safety:** `kill -9` during indexing and merges, repeated many times, then a check that the index opens at its last commit with every checksum valid.

## Testing

- Every top-k strategy (exhaustive, WAND, Block-Max WAND) must return the same top-k documents and scores as a brute-force scorer, on random documents and random queries.
- Codecs round-trip: bit-packing, front coding and the term dictionary are checked on random input, including edge cases such as empty blocks, single postings and the maximum doc ID.
- HNSW recall@10 is checked against an exact search at fixed parameters.
- Merges keep search results unchanged: the same queries return the same results before and after a forced merge.

## Roadmap

- [x] Scope and design
- [ ] Analyzer: tokeniser, stop words, Porter stemmer
- [ ] In-memory index and BM25 with exhaustive scoring
- [ ] Segment format: term dictionary, bit-packed postings, positions, checksums
- [ ] Commits, deletes and tiered merges
- [ ] WAND and Block-Max WAND
- [ ] Phrase queries and the query parser
- [ ] HNSW, then int8 scalar quantisation
- [ ] Hybrid search with RRF
- [ ] Parallel segment search on worker threads, and the HTTP API
- [ ] Benchmarks against Postgres GIN, pgvector and Lucene
- [ ] Write-up

## Planned structure

```
src/
  analysis/   # tokeniser, stop words, Porter stemmer
  codec/      # bit-packing, varints, front-coded term dictionary, CRC32
  index/      # indexing buffer, segment writer and reader, commits, merge policy
  search/     # postings iterators, BM25, WAND, Block-Max WAND, phrases, query parser
  vector/     # HNSW, scalar quantisation
  hybrid/     # Reciprocal Rank Fusion
  server/     # HTTP API
bench/        # MS MARCO, BEIR and ann-benchmarks runners, plus Postgres/pgvector and Pyserini baselines
test/
```

## Requirements

Node.js 24 or newer and TypeScript. The engine uses only Node.js built-ins at runtime. The benchmarks also need Docker (for Postgres with pgvector) and Python (for Pyserini and for computing embeddings).
