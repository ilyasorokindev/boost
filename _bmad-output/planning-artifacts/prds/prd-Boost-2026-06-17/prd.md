---
title: "boost::sort::timsort — Adaptive Stable Sort"
status: final
created: 2026-06-17
updated: 2026-06-17
---

# PRD: boost::sort::timsort — Adaptive Stable Sort

## 0. Document Purpose

This PRD is written for the author and any future contributors to the `boost::sort` library. It defines requirements for adding `timsort` as a new single-thread stable sorting algorithm. The document follows the established `boost::sort` feature pattern: one algorithm per subdirectory, integrated into the cumulative header, with tests and benchmarks. Downstream artifacts include implementation stories and benchmark targets.

---

## 1. Vision

`boost::sort` provides the fastest sorting algorithms available for different data shapes, but it currently lacks an algorithm that exploits **natural order already present in the data**. On real-world inputs — log streams, database result sets, event queues, incrementally updated containers — a significant portion of the data arrives pre-sorted or in partially-sorted segments. No existing algorithm in the library is specifically designed to take advantage of this structure.

**Timsort** is a well-proven adaptive stable sort, used as the default sort in Python, Java (Arrays.sort for objects), Android, and V8. Its core insight is to detect and merge existing sorted subsequences ("natural runs") rather than treating the input as random. On already-sorted data it runs in O(N); on random data it degrades gracefully to O(N log N) — matching `spinsort` without sacrificing generality.

Adding `timsort` to `boost::sort` gives C++ developers a first-class stable sort that is uniquely suited to real-world, non-random inputs, following the same ergonomic API they already know.

---

## 2. Target User

### 2.1 Jobs To Be Done

- **Sort nearly-sorted data efficiently** — logs, event streams, time-series, incremental updates; the developer knows data has structure but doesn't want to write a custom sort.
- **Guarantee stable sort semantics** — preserve relative order of equal elements (critical for multi-key sort, UI rendering, record processing).
- **Drop in a known algorithm by name** — use `timsort` by name because it is the industry-standard adaptive stable sort, predictable in behavior and well-documented outside Boost.
- **Stay within the existing `boost::sort` API convention** — no new learning curve; `timsort(first, last)` works just like `spinsort(first, last)`.

### 2.2 Non-Users (v1)

- Users needing a **parallel** sort — not in scope.
- Users needing an **in-place** stable sort (O(1) extra memory) — WikiSort/GrailSort addresses that; timsort uses O(N) auxiliary memory.
- Users targeting **non-random-access iterators** — timsort requires random-access iterators, same constraint as existing algorithms.

### 2.3 Key User Journeys

- **UJ-1. Anton sorts a timestamped event log.**
  Anton's app accumulates events in a vector; each batch arrives mostly in order with occasional out-of-order entries. He calls `boost::sort::timsort(events.begin(), events.end(), by_timestamp)`. The sort completes faster than `spinsort` because it detects the existing runs; relative order of events with identical timestamps is preserved. He does not need to benchmark alternatives — the algorithm name tells him it was designed exactly for this pattern.

- **UJ-2. Developer integrates timsort via the cumulative header.**
  Developer adds `#include <boost/sort/sort.hpp>` (as they already do) and calls `boost::sort::timsort(...)`. No additional include or linker step needed. Works with C++11 compiler, same as the rest of the library.

---

## 3. Glossary

- **Natural run** — a maximal contiguous subsequence that is already sorted in ascending or descending order. Descending runs are reversed in-place to become ascending runs.
- **Minrun** — minimum run length accepted without merging. Computed from input size N as a value in [32, 64] such that ⌈N/minrun⌉ is a power of two or just below, minimising merge passes.
- **Galloping mode** — optimised merge strategy that switches from element-by-element comparison to exponential search when one side of a merge dominates consecutively. Exits galloping when the advantage drops below `MIN_GALLOP` (default 7).
- **Run stack** — a stack of pending (base, length) pairs representing detected runs awaiting merge. Timsort maintains two invariants on the stack to bound its depth at O(log N).
- **Stable sort** — a sort that preserves the relative order of elements that compare equal.
- **Merge buffer** — temporary auxiliary storage, up to N/2 elements, used during the merge phase.
- **Random-access iterator** — an iterator satisfying the C++ `RandomAccessIterator` concept (supports `+`, `-`, `[]` in O(1)).

---

## 4. Features

### 4.1 Core Algorithm — `timsort`

**Description:** Header-only implementation of timsort for random-access iterator ranges. The algorithm scans the input for Natural Runs (reversing descending runs), extends short runs to Minrun length using binary insertion sort, then merges runs from the Run Stack using a Merge Buffer, applying Galloping Mode when beneficial. The implementation is Stable. Follows the canonical timsort specification (Tim Peters, 2002) with the standard stack-invariant merge policy.

The algorithm uses `[ASSUMPTION: N/2 merge buffer]` as the upper bound for the Merge Buffer — consistent with `spinsort`. On systems where heap allocation is constrained this is the dominant cost.

**Functional Requirements:**

#### FR-1: Basic sort interface
The developer can call `boost::sort::timsort(first, last)` on any range defined by two random-access iterators, using `operator<` as the comparator.

**Consequences (testable):**
- Result is sorted in ascending order by `operator<`.
- Relative order of elements comparing equal is preserved (stability).
- Compiles with C++11 (`-std=c++11`).

#### FR-2: Custom comparator interface
The developer can call `boost::sort::timsort(first, last, comp)` with any strict-weak-ordering comparator.

**Consequences (testable):**
- Result is sorted according to `comp`.
- Stability holds: elements for which `!comp(a,b) && !comp(b,a)` retain their original relative order.

#### FR-3: Natural run detection
The algorithm detects and exploits Natural Runs present in the input.

**Consequences (testable):**
- On an already-sorted range of N elements, the number of comparisons is O(N) (verified by comparison counter in tests).
- On a range consisting of K equal-length sorted segments, performance scales with K, not N.

#### FR-4: Complexity guarantees

**Consequences (testable):**
- Best case (already sorted): O(N) comparisons.
- Average and worst case: O(N log N) comparisons.
- Auxiliary memory: O(N) — no more than N/2 elements in the Merge Buffer at any point.
- Stack depth: O(log N) — Run Stack invariants enforced.

#### FR-5: Edge cases handled without error

**Consequences (testable):**
- Empty range `[first, first)` — no-op, no crash.
- Single element — no-op.
- Two elements — single comparison, correct result.
- All elements equal — stable, O(N) comparisons.
- All elements in reverse order — correctly sorted, O(N log N).

---

### 4.2 Library Integration

**Description:** `timsort` is delivered as a self-contained subdirectory under `boost/sort/`, following the exact layout of `spinsort` and `pdqsort`. The cumulative header `sort.hpp` is updated to include it so existing users get timsort automatically on next recompile after updating their include path.

**Functional Requirements:**

#### FR-6: File location
The implementation lives at `include/boost/sort/timsort/timsort.hpp`.

**Consequences (testable):**
- `#include <boost/sort/timsort/timsort.hpp>` compiles standalone.
- No other file in the library is modified except `sort.hpp` and the test/benchmark registries.

#### FR-7: Cumulative header inclusion
`include/boost/sort/sort.hpp` includes `<boost/sort/timsort/timsort.hpp>`.

**Consequences (testable):**
- `#include <boost/sort/sort.hpp>` exposes `boost::sort::timsort`.
- No breaking change to any existing symbol.

#### FR-8: Namespace
All public symbols are in namespace `boost::sort`.

**Consequences (testable):**
- `boost::sort::timsort(...)` resolves correctly.
- No pollution of the global namespace.

#### FR-9: No new dependencies
The implementation introduces no external dependencies beyond the C++11 standard library.

**Consequences (testable):**
- `grep` for `#include` in `timsort.hpp` shows only `<algorithm>`, `<iterator>`, `<functional>`, `<vector>` (or subset thereof) — all standard headers present in C++11.

---

### 4.3 Test Suite

**Description:** A dedicated test file verifying correctness, stability, and edge-case behaviour. Follows the pattern of existing `boost::sort` tests (see `test/` directory). Tests use plain arrays and `std::vector`; no external test framework required beyond what the library already uses.

**Functional Requirements:**

#### FR-10: Correctness tests
Tests cover: random data, already-sorted data, reverse-sorted data, data with many duplicates, single-run data.

**Consequences (testable):**
- All tests pass with `std::is_sorted` (or equivalent) assertion after sort.
- Tests exercise both `timsort(first, last)` and `timsort(first, last, comp)` overloads.

#### FR-11: Stability test
A test verifies that equal elements retain their original relative order.

**Consequences (testable):**
- Construct a range of pairs `{key, original_index}`; sort by key; verify `original_index` is non-decreasing within equal-key groups.

#### FR-12: Edge-case tests
Tests cover: empty range, one element, two elements, all-equal elements, reversed range.

**Consequences (testable):**
- No crash, no undefined behaviour (run under sanitizers: ASAN, UBSAN).
- Correct output for all cases.

#### FR-13: Type coverage
Tests include at least: `int`, `std::string`, a user-defined struct with custom comparator.

**Consequences (testable):**
- All three types compile and pass.

---

### 4.4 Benchmarks

**Description:** A benchmark comparing `timsort` against `spinsort`, `flat_stable_sort`, and `std::stable_sort` across data shapes that expose timsort's adaptive advantage. Follows the structure of existing benchmarks in `benchmark/`. Results are written to stdout in tabular form.

**Functional Requirements:**

#### FR-14: Benchmark data shapes
Benchmarks run on: random, already-sorted, reverse-sorted, nearly-sorted (1% perturbation), and pipe-organ (ascending then descending halves).

**Consequences (testable):**
- All five shapes × all four algorithms produce timing output without crash.

#### FR-15: Benchmark comparison algorithms
Benchmark includes: `boost::sort::timsort`, `boost::sort::spinsort`, `boost::sort::flat_stable_sort`, `std::stable_sort`.

**Consequences (testable):**
- Results table has 4 columns (one per algorithm) × 5 rows (one per data shape).

#### FR-16: Benchmark element types
Benchmarks run on `int` and `std::string` to capture both comparison-cheap and comparison-expensive types.

**Consequences (testable):**
- Output includes separate tables for `int` and `std::string`.

---

### 4.5 Documentation

**Description:** In-source documentation and an entry in the library's README describing the algorithm, its complexity, memory usage, and the target use case. Follows the style of the existing README table.

**Functional Requirements:**

#### FR-17: README entry
`libs/sort/README.md` single-thread algorithm table gains a `timsort` row.

**Consequences (testable):**
- Row includes: algorithm name, Stable=yes, Additional memory=N/2, Best/avg/worst=N / N LogN / N LogN, Comparison method=Comparison operator.

#### FR-18: Header documentation comment
`timsort.hpp` contains a top-of-file comment describing: algorithm name, stability, complexity, memory, author/origin reference (Tim Peters 2002), Boost licence header.

**Consequences (testable):**
- Comment present; Boost Software License header present.

#### FR-19: Usage example
A short usage example (5-10 lines) is included either inline in the header comment or in a separate `example/timsort_example.cpp`.

**Consequences (testable):**
- Example compiles and runs correctly.

---

## 5. Non-Goals (Explicit)

- **No parallel timsort** — single-thread only in v1; parallel variants belong to the parallel algorithm group and require a separate design.
- **No C++17 ranges / execution policies** — API is C++11 iterator-based only.
- **No in-place (O(1) memory) variant** — out of scope; WikiSort covers that niche if needed later.
- **No formal Boost library review** — this is personal/team use; Boost.Review process not followed.
- **No Boostbook/Quickbook documentation** — plain Markdown/comments only.
- **No support for non-random-access iterators** — consistent with existing library algorithms.
- **No string specialisation** — `spreadsort` already handles string radix sorting; timsort is generic comparison-based.

---

## 6. MVP Scope

### 6.1 In Scope

- `include/boost/sort/timsort/timsort.hpp` — full timsort implementation, C++11, header-only.
- Integration in `include/boost/sort/sort.hpp`.
- `test/timsort_test.cpp` — correctness, stability, edge cases, type coverage.
- `benchmark/` entry comparing timsort vs. spinsort, flat_stable_sort, std::stable_sort across 5 data shapes.
- README row + header comment + one usage example.

### 6.2 Out of Scope for MVP

- Parallel timsort — deferred to v2 if needed.
- Ranges / C++20 concepts interface — deferred; low priority for personal use.
- Performance tuning beyond standard timsort parameters (minrun, MIN_GALLOP) — deferred; baseline first.
- CI/CD integration — `[NOTE FOR PM]` if this repo gains CI, add timsort to the test matrix.

---

## 7. Success Metrics

**Primary**

- **SM-1:** On nearly-sorted data (95%+ pre-sorted, N = 1 000 000 `int`), `timsort` is at least **2× faster** than `spinsort`. Validates FR-3, FR-4, FR-14.
- **SM-2:** All correctness, stability, and edge-case tests pass with ASAN + UBSAN enabled. Validates FR-10, FR-11, FR-12.

**Secondary**

- **SM-3:** On random `int` data (N = 1 000 000), `timsort` is within **20% of `spinsort`** wall-clock time (no catastrophic regression on the worst case for timsort). Validates FR-4, FR-15.
- **SM-4:** `timsort` compiles cleanly with `-std=c++11 -Wall -Wextra -Wpedantic` — zero warnings. Validates FR-6, FR-9.

**Counter-metrics (do not optimise)**

- **SM-C1:** Do not optimise timsort specifically for random data at the expense of nearly-sorted performance — SM-3 is a floor, not a goal. The adaptive advantage (SM-1) is the primary value proposition.

---

## 8. Open Questions

1. **Minrun computation** — use the canonical Python-derived formula (find the 6 most-significant bits of N, round up if any lower bits are set)? Or a fixed value of 32/64? `[ASSUMPTION: canonical formula used]`
2. **Merge buffer allocation** — allocate once at sort start (up to N/2), or grow dynamically? Dynamic is safer for very large N but adds allocation overhead. `[ASSUMPTION: allocate once, up to N/2]`
3. **MIN_GALLOP tuning** — should MIN_GALLOP be adaptive per sort call (as in Python's original) or fixed at 7? Adaptive is more complex but better for pathological inputs. `[ASSUMPTION: adaptive, starting at 7, matching Python reference implementation]`
4. ~~**Test framework**~~ — **Resolved:** No external framework. Tests use plain C++ with manual assertions (see `test/test_spinsort.cpp`). New test file: `test/test_timsort.cpp`, same pattern.
5. ~~**Benchmark harness**~~ — **Resolved:** Add `using bsort::timsort;` + timing block to existing `benchmark/single/benchmark_numbers.cpp` (and `benchmark_strings.cpp`). No new binary needed.

---

## 9. Assumptions Index

- **§4.1** — Merge Buffer is at most N/2 elements (matches `spinsort` convention).
- **§8 / Q1** — Minrun uses the canonical Python-derived formula.
- **§8 / Q2** — Merge buffer allocated once at sort start.
- **§8 / Q3** — MIN_GALLOP is adaptive, starting at 7.
