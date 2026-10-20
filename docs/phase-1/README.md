# Phase 1 — Vector Matrix Segment: basic CRUD

Estimated: 3-4 weeks at side-project pace. Goal: a flat binary file storing dense vectors with a fixed layout, first written and read with plain `std::fs` I/O, then re-read through `mmap`. No `unsafe` beyond what `memmap2`'s safe API requires yet — the heavy `unsafe` work is Phase 2.

## Status

Not started. Phase 0 (ownership/borrowing fundamentals) is in progress in `src/main.rs` via throwaway exercises (not meant to be kept).

## Tasks

- [ ] **Define the binary file layout**
  - Header: `num_vectors` (u32), `dimension` (u32), `version` (u16) — or similar; finalize exact types and byte order while implementing.
  - Body: `num_vectors * dimension` `f32` values, contiguous, no padding between vectors.
  - Offset of vector `i` = `header_size + i * dimension * size_of::<f32>()`.
- [ ] **Write a batch write function** (no `mmap` yet — direct I/O via `std::fs::File` + `std::io::Write` is enough here)
  - Takes a slice/collection of vectors and writes header + body to a file.
- [ ] **Switch to `memmap2` for reads**
  - Map the file, read a vector by ID by computing its offset into the mapped region.
- [ ] **Test with a small randomly generated dataset** before anything larger — validate the round trip (write with plain I/O, read back via mmap) matches.

## Explicitly out of scope for this phase

- Any search/index logic (that's Phase 2, on top of the Memory Sandbox).
- `mmap`-based writes (writing still goes through plain I/O in Phase 1; only reads use `mmap`).
- Update/delete (out of scope for the whole MVP — see `CLAUDE.md`).
- `madvise`/`mlock` memory hints (deferred phase).

## Concepts this phase should build, in rough order

1. `struct` + `impl` — modeling the header and the segment as types.
2. Basic file I/O (`std::fs`, `std::io`) — writing bytes to disk.
3. Byte-level layout control — turning a `struct` into a fixed sequence of bytes and back (this is where `bincode` or manual byte packing enters for the header; vectors themselves are raw `f32` bytes, not `bincode`-serialized, per the flat-array design).
4. `memmap2` basics — mapping a file read-only, indexing into the resulting slice.
5. `Result`/error handling for file I/O — Rust doesn't use exceptions; this phase is a natural place to learn `Result<T, E>` and `?`.

## Reference

Full architecture and rationale: `../vectorforge_projeto.pdf` (original planning doc, Portuguese) and `../../CLAUDE.md` (English summary, project conventions, collaboration mode).
