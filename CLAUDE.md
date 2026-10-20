# VectorForge

A decoupled storage engine for vectors and payloads, with 100% pluggable search — a Rust learning side project motivated by a real gap identified during a university parallelism assignment.

## Context

The author has strong C experience, no prior Rust experience, and works full-time at Qdrant. This project is developed pair-programming with Claude Code, at side-project pace.

**Current priority: learning Rust through this specific use case.** A working engine is a bonus outcome, not the goal. See "Collaboration mode" below — it governs how Claude should behave in this repo more than anything else in this file.

The original planning document (in Portuguese) lives at `docs/vectorforge_projeto.pdf` and is the source of truth for motivation, stack evaluation, and architecture. This file and `docs/phase-1/` translate and operationalize it in English, since the repo is being standardized to English going forward.

## Collaboration mode

- Claude's role is to **teach and guide**, not to write the implementation. The user types and runs every command and every line of Rust themselves.
- Do not use Bash/Write/Edit to scaffold source files, write Rust code, or run cargo/rustup commands on the user's behalf unless explicitly asked to do so for a specific step (e.g. documentation files like this one are fine — Rust source is not).
- Explain concepts by tying them back to VectorForge's actual architecture (mmap safety, ownership, concurrency) rather than abstract tutorials.
- Prefer the user writing a first (possibly wrong) attempt before Claude explains or corrects — that's what fixes the learning.
- Explain `unsafe` blocks line by line before the user accepts them, especially in Phase 1 and Phase 2 work.

## Architecture

Two decoupled data segments, mirroring the storage/index split from the author's university parallelism work — now persisted to disk via mmap instead of living only in RAM.

**Vector Matrix Segment**
Flat binary file with dense vectors, fixed layout (offset of vector `i` = `header_size + i * D * sizeof(f32)`). Mapped via `mmap`, exposed to user code as a memory slice (`&[f32]`) — not as a query API.

**Payload Store Segment**
Separate key-value store (`fjall` or `redb`), vector ID as key. Only accessed after the user's search algorithm has already decided which IDs are "winners" — keeps the search hot path free of metadata I/O.

**Memory Sandbox (the product's central idea)**
The engine exposes no search function. It exposes: (1) a memory-mapped slice of all vectors, (2) a payload-read-by-ID function, (3) memory hint primitives (`madvise`/`mlock`, later phase). The user writes their own brute-force, LSH, HNSW, or IVF on top of this, parallelizing with `rayon` however they want — the engine never decides how to search, only how to access the data.

## Stack

| Crate | Role | Notes |
|---|---|---|
| `memmap2` | mmap | de facto standard, low risk |
| `rayon` | parallelism | good entry point coming from OpenMP/C `#pragma omp parallel for` |
| `bincode` | payload serialization | start here; simple and mature |
| `fjall` or `redb` | payload key-value store | chosen over `sled` (stalled in beta for years) |
| `nix` (madvise/mlock) | memory hints | deferred to a phase after the MVP |
| `rkyv` (zero-copy) | payload serialization | only swap in for `bincode` if a benchmark proves deserialization is a real bottleneck |

## MVP scope decisions

- **Append-only, read-only after initial load.** Update/delete are explicitly out of scope for the MVP, not just unmentioned.
- **Crash safety is minimal for the MVP**: a "write complete" flag (e.g. a valid-vector counter updated only after the whole mmap is `msync`'d), not a real WAL. A real WAL is a later-phase concern.
- **Validation**: benchmark against Faiss (`IndexFlatL2`) with the same dataset and thread count before claiming the engine is competitive.

## Roadmap

Realistic pace for learning Rust from zero as a side project alongside full-time work: ~5-6 months total.

- **Phase 0 — Rust fundamentals** (2-3 weeks): ownership, borrow checker, lifetimes, via small throwaway exercises, no `unsafe`.
- **Phase 1 — Vector Matrix Segment basic CRUD** (3-4 weeks): see `docs/phase-1/`.
- **Phase 2 — Memory Sandbox + parallelism** (3-4 weeks): expose the mapped slice, brute-force search, then `rayon`. Heaviest `unsafe` work of the project.
- **Phase 3 — Payload store** (2-3 weeks): integrate `fjall`/`redb` + `bincode`.
- **Phase 4 — Validation and benchmark** (2 weeks): compare against Faiss.
- **Phase 5 (optional)** — a second search algorithm (LSH or simplified HNSW) to validate that the storage/search decoupling promise holds.

Validation milestone: at the end of Phase 2, it's already possible to tell whether the memory sandbox architecture works in practice.

## Risks

- **Time**: side project pace alongside full-time work moves slower than planned — expected, revisit pace each phase.
- **Scope**: temptation to generalize too early (every index type, every payload type) before validating the simplest case. Resist until Phase 4.
- **Market**: LanceDB and Zvec (Alibaba, 2026) occupy part of the same space — doesn't invalidate the project as learning/portfolio, but reduces novelty if the goal evolves into a real product.
