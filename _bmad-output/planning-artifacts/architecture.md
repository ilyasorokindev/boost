---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
lastStep: 8
status: complete
completedAt: '2026-06-17'
inputDocuments:
  - AGENTS.md
  - _bmad-output/planning-artifacts/prds/prd-Boost-2026-06-17/prd.md
workflowType: architecture
project_name: Boost C++ Superproject
user_name: Ilyasorokin
date: '2026-06-17'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements:**
19 FRs across five groups:
- Core algorithm (FR-1–5): two-overload public API, natural run detection,
  O(N)/O(N log N) complexity, edge-case safety
- Library integration (FR-6–9): header-only subdirectory following spinsort/pdqsort
  layout, cumulative-header inclusion, boost::sort namespace, no new dependencies
- Test suite (FR-10–13): correctness, stability, edge cases, type coverage
  (int, std::string, user-defined struct)
- Benchmarks (FR-14–16): 5 data shapes × 4 algorithms × 2 element types
- Documentation (FR-17–19): README row, header comment, usage example

**Non-Functional Requirements:**
- C++11 minimum (despite repo using C++17 — library targets C++11)
- Header-only; no compiled sources, no external dependencies
- ASAN + UBSAN clean; zero warnings under -Wall -Wextra -Wpedantic
- Performance: ≥2× vs. spinsort on nearly-sorted (N=1M int) — SM-1, primary value prop
- Performance: ≤20% regression vs. spinsort on random data — SM-3, floor not goal

**Scale & Complexity:**
- Primary domain: C++ template algorithm library (header-only)
- Complexity level: low-medium
- Key algorithmic subsystems: 4
  (run detection, binary insertion sort, run stack + invariant enforcement, merge + galloping)

### Technical Constraints & Dependencies

- Random-access iterators only (std::RandomAccessIterator concept)
- Single-threaded only (parallel variant explicitly out of scope)
- Merge buffer: O(N) — single std::vector allocation ≤ N/2, allocated once at
  sort start before any data is touched; never grown dynamically
- Run stack: O(log N) depth — two invariants maintained (à la Python timsort)
- MIN_GALLOP: adaptive, starts at 7, resets per sort call — must live in a
  per-call SortState struct, not a global/static/thread_local
- Minrun: canonical Python-derived formula (6 MSBs of N, round up if lower bits set);
  result is always in [32, 64]
- Build: CMake + Ninja or b2; tested via ctest or b2 test targets
- Code style: Google C++ Style Guide (2-space indent, snake_case/PascalCase,
  80-char lines, #pragma once, no using namespace in headers)
- Include order: own header → Boost headers → stdlib headers (alphabetically within groups)

### Cross-Cutting Concerns Identified

- **C++11 compatibility**: all components (header, tests, benchmarks) must compile
  with -std=c++11; no C++14/17/20 features. Key pitfalls:
  - Use `std::iterator_traits<Iter>::value_type` (not `Iter::value_type`) throughout
  - Use `reserve` + `push_back`/`emplace_back` for merge buffer (never `resize` — breaks
    non-default-constructible types)
  - Check `std::is_nothrow_move_constructible` before moving into merge buffer;
    fall back to copy when false
- **Stability invariant**: must be preserved across all merge operations and any
  short-range fallbacks (e.g. binary insertion sort); the merge tie-breaking rule
  must consistently favor the left (earlier) element on equal comparison
- **Template hygiene**: no using namespace in headers; all internal helpers in
  boost::sort::detail; iterator access exclusively through iterator_traits alias
- **API consistency**: public signatures mirror spinsort/pdqsort exactly;
  RandomAccessIterator constraint enforced via static_assert in the public header
- **Sanitizer cleanliness**: all test runs must pass ASAN + UBSAN; implies no
  out-of-bounds on merge buffer, no signed overflow in index arithmetic,
  no reads from moved-from elements
- **Exception safety precondition**: comparator must not throw (document as
  precondition for v1; do not silently corrupt the range on throw)

## Starter Template Evaluation

### Primary Technology Domain

C++ header-only algorithm library — no external project generator applies.
The "starter" is the established boost::sort per-algorithm layout pattern.

### Pattern Options Considered

**Option A — spinsort/pdqsort pattern (single monolithic header)**
Used by all existing single-thread single-algorithm sorts (spinsort: 570 lines,
pdqsort: 618 lines). Each algorithm lives entirely in one .hpp file.

**Option B — spreadsort pattern (entry point + detail/ subdirectory)**
Used by spreadsort, which has three independently-usable sort variants
(float, integer, string). Subsystems are independently reusable.

### Selected Pattern: Option A (spinsort/pdqsort layout)

**Rationale:**
- Consistent with all existing single-thread single-algorithm precedent
- Timsort's 4 subsystems couple tightly through SortState — not independently
  reusable in the way spreadsort's variants are
- ~500-700 lines is within the readable range for a single file
- Simpler to review, patch, and vendor-copy

**File layout:**

    libs/sort/include/boost/sort/timsort/
      timsort.hpp                ← full implementation, ~500-700 lines
    libs/sort/test/
      test_timsort.cpp           ← correctness/stability/edge-case/type tests
    libs/sort/benchmark/single/
      benchmark_numbers.cpp      ← extend with timsort column (not a new file)
      benchmark_strings.cpp      ← extend with timsort column (not a new file)
    libs/sort/include/boost/sort/
      sort.hpp                   ← add #include <boost/sort/timsort/timsort.hpp>

**Architectural decisions established by this pattern:**
- Language: C++11, header-only, no compiled sources
- No new build targets needed — test_timsort.cpp picked up by existing Jamfile/CMake
- No new benchmark binary — timsort columns added to existing benchmark executables
- Internal structure of timsort.hpp: namespace sections separated by comments
  (run detection → insertion sort → merge stack → merge engine → public API),
  all in boost::sort with helpers in boost::sort::detail (anonymous namespace
  or detail namespace within the single file, not a separate file)

**Note:** "Refactor to detail/ subdirectory" is a deferred option if the
implementation grows past ~900 lines. Document this threshold in a comment
at the top of timsort.hpp.

---

### Open Architectural Questions Resolved in Step 4

Previously open questions, now closed:
1. File structure → single-file (spinsort/pdqsort pattern) — decided in Step 3
2. Merge buffer memory → fixed N/2 at sort start, no cap for v1 — acceptable tradeoff
3. Benchmark scope → 4-way (timsort, spinsort, flat_stable_sort, std::stable_sort) per PRD FR-15
4. "Nearly-sorted" definition → 95% pre-sorted (Option A, see D4.3 below)

---

## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions (block implementation):**
- D1.1: SortState struct design
- D1.2: Merge direction policy
- D1.3: Run stack invariant
- D2.1: Iterator enforcement mechanism
- D3.1: Detail namespace name

**Important Decisions (shape architecture):**
- D2.2: Comparator default type
- D3.2: Dependency policy (stdlib-only includes)
- D4.2: Benchmark scope
- D4.3: "Nearly-sorted" quantitative definition

**Deferred Decisions (post-v1):**
- Refactor to detail/ subdirectory if implementation exceeds ~900 lines
- Buffer cap / merge-in-place fallback for very large N
- Adaptive MIN_GALLOP tuning per data shape (currently: start at 7, adapt per call)

---

### Algorithm Architecture

**D1.1 — SortState struct: template struct (Option B)**

```cpp
template <typename ValueType>
struct SortState {
    std::vector<ValueType> buffer;  // merge buffer, capacity <= N/2
    int min_gallop;                  // adaptive gallop threshold, starts at 7
};
```

Rationale: type-safe, clean ownership of both per-call mutable resources,
matches reference implementation pattern. Instantiated once on the stack
in the top-level timsort() call and passed by reference through all
internal merge functions. Run detection and insertion sort do not receive it.

**D1.2 — Merge direction: bidirectional (canonical)**

When merging adjacent runs A (left, length la) and B (right, length lb):
- If la <= lb: copy A into buffer → merge buffer + B into range left-to-right
- If la > lb:  copy B into buffer → merge A + buffer into range right-to-left

Bounds buffer at N/2. Allocate buffer once at sort start (reserve N/2),
clear() between merge operations, never grow after initial reserve.

**D1.3 — Run stack invariant: canonical two-invariant policy**

Stack entries are (base, length) pairs. Before pushing a new run, enforce:
- Invariant 1: len[n-1] > len[n]         (top two runs)
- Invariant 2: len[n-2] > len[n-1] + len[n]  (top three runs)

Enforcement loop: while either invariant is violated, merge the smaller of
the top two or three runs. This bounds stack depth at O(log N) and
minimises total merge work.

---

### API Contract & Type Safety

**D2.1 — Iterator enforcement: static_assert**

```cpp
template <typename Iter, typename Compare>
void timsort(Iter first, Iter last, Compare comp) {
    static_assert(
        std::is_base_of<
            std::random_access_iterator_tag,
            typename std::iterator_traits<Iter>::iterator_category
        >::value,
        "boost::sort::timsort requires RandomAccessIterator"
    );
    // ...
}
```

Placed in the public overloads in `boost::sort`, not buried in `tim_detail`.
Fires at compile time with a readable message. Follows pdqsort's style.

**D2.2 — Comparator default: std::less<value_type> (C++11)**

```cpp
template <typename Iter>
void timsort(Iter first, Iter last) {
    typedef typename std::iterator_traits<Iter>::value_type value_type;
    timsort(first, last, std::less<value_type>());
}
```

`std::less<>` (transparent comparator) requires C++14 — excluded.

**D2.3 — Exception safety precondition (documented, not enforced)**

Precondition: the comparator `comp` must not throw.
If `comp` throws during a merge, elements are split between the live range
and the merge buffer — the range is left in a valid but unspecified state.
Document this in the header comment. No try/catch wrapper in v1.

---

### Internal Organization

**D3.1 — Detail namespace: `tim_detail`**

All internal implementation functions live in `namespace boost::sort::tim_detail`.
The public API (`boost::sort::timsort`) is defined after the `tim_detail` block.

Follows the `spin_detail` convention in spinsort.hpp. Allows test files to
access internals via `using tim_detail::...` for targeted unit tests of
`compute_minrun`, the merge function, and the run stack invariant checker.

**D3.2 — Dependency policy: stdlib headers only**

`timsort.hpp` includes only:
```cpp
#include <algorithm>    // std::min, std::rotate (insertion sort fallback)
#include <cstddef>      // std::size_t, std::ptrdiff_t
#include <functional>   // std::less
#include <iterator>     // std::iterator_traits, std::distance, std::advance
#include <vector>       // std::vector (merge buffer)
```

No Boost headers. Verified by FR-9 testable consequence.

**D3.3 — iterator_traits alias (enforced throughout)**

```cpp
// At top of tim_detail namespace:
template <typename Iter>
using iter_value_t = typename std::iterator_traits<Iter>::value_type;
template <typename Iter>
using iter_diff_t  = typename std::iterator_traits<Iter>::difference_type;
```

All iterator-derived types accessed exclusively through these aliases.
Never via `Iter::value_type` directly.

**D3.4 — Move-vs-copy policy for merge buffer**

Use `std::move_if_noexcept` when copying elements into the merge buffer:
```cpp
buffer.push_back(std::move_if_noexcept(*src));
```
Falls back to copy when the move constructor may throw, preserving source
integrity. C++11 has `std::move_if_noexcept` in `<utility>` — add to includes.

> Note: add `<utility>` to the include list (D3.2 updated accordingly).

---

### Test & Benchmark Strategy

**D4.1 — Test framework: Boost.Test (test_exec_monitor)**

Matches `test/test_spinsort.cpp` exactly:
```cpp
#include <boost/test/included/test_exec_monitor.hpp>
#include <boost/test/test_tools.hpp>
```

No external test framework beyond what the library already uses (FR-10 confirmed).

**D4.2 — Benchmark scope: 4-way (PRD FR-15)**

Benchmark compares: timsort, spinsort, flat_stable_sort, std::stable_sort.
Extends `benchmark/single/benchmark_numbers.cpp` and `benchmark_strings.cpp`
by adding a `timsort` timing column alongside the existing algorithms.
No new benchmark binary created.

**D4.3 — "Nearly-sorted" definition: 95% pre-sorted**

For SM-1 (≥2× vs. spinsort on nearly-sorted, N=1M int):

Input generation: start with a fully sorted array of N elements, then
randomly swap 5% of randomly selected adjacent pairs (N * 0.05 swaps total).
This produces a predictable, reproducible perturbation that models
log/event-queue data arriving mostly in order with occasional out-of-order
entries (matching UJ-1: Anton's timestamped event log).

Document the seed (mt19937, seed=42) and swap count in the benchmark source.

### Decision Impact Analysis

**Implementation sequence (order matters):**
1. `compute_minrun(n)` — standalone, no dependencies within timsort
2. `tim_detail::SortState<V>` struct — depends on nothing
3. Binary insertion sort helper — standalone, uses iter_value_t alias
4. Run detection (`count_run`) — standalone
5. Run stack invariant checker + `merge_collapse` — depends on SortState
6. Merge engine with galloping (`merge_lo`, `merge_hi`) — depends on SortState
7. Top-level `timsort()` public overloads — assembles all of the above
8. `sort.hpp` update — one line addition
9. `test_timsort.cpp` — against public API + tim_detail internals
10. Benchmark extension — add timsort column to two existing files

**Cross-component dependencies:**
- SortState is the central shared state; merge_lo/merge_hi write min_gallop
  and use buffer — they must not be called without an initialized SortState
- compute_minrun result determines how many insertion-sort extensions happen
  before the first run is pushed — must be correct before the main loop
- Run stack invariant depends on merge_lo/merge_hi being correct — test
  merge functions in isolation before testing stack invariant enforcement

---

## Implementation Patterns & Consistency Rules

### Critical Conflict Points Identified

7 areas where AI agents could independently make incompatible choices.

### P1 — Canonical Function Names

All internal functions must use these exact names — no synonyms, no abbreviations.

| Function | Signature sketch | Purpose |
|---|---|---|
| `compute_minrun` | `(iter_diff_t n) -> iter_diff_t` | Minrun formula |
| `count_run` | `(Iter first, Iter last, Compare comp) -> iter_diff_t` | Detect + reverse descending runs |
| `binary_insertion_sort` | `(Iter first, Iter last, Iter start, Compare comp)` | Extend short runs to minrun |
| `gallop_left` | `(const V& key, Iter base, iter_diff_t len, iter_diff_t hint, Compare comp) -> iter_diff_t` | Insertion point in left run |
| `gallop_right` | `(const V& key, Iter base, iter_diff_t len, iter_diff_t hint, Compare comp) -> iter_diff_t` | Insertion point in right run |
| `merge_lo` | `(Iter base1, iter_diff_t len1, Iter base2, iter_diff_t len2, SortState<V>& state, Compare comp)` | Merge when left run is shorter |
| `merge_hi` | `(Iter base1, iter_diff_t len1, Iter base2, iter_diff_t len2, SortState<V>& state, Compare comp)` | Merge when right run is shorter |
| `merge_collapse` | `(RunStack& stack, Iter data, SortState<V>& state, Compare comp)` | Enforce stack invariants |
| `merge_force_collapse` | `(RunStack& stack, Iter data, SortState<V>& state, Compare comp)` | Drain stack at end of input |

**SortState member names (exact):** `buffer`, `min_gallop`
**RunStack entry names (exact):** `base` (Iter), `len` (iter_diff_t)

### P2 — Namespace & File Layout

```
namespace boost {
namespace sort {
namespace tim_detail {
  // 1. iterator_traits aliases (iter_value_t, iter_diff_t)
  // 2. compute_minrun
  // 3. binary_insertion_sort
  // 4. count_run
  // 5. gallop_left, gallop_right
  // 6. SortState<V> struct, RunStack typedef/struct
  // 7. merge_lo, merge_hi
  // 8. merge_collapse, merge_force_collapse
} // namespace tim_detail

// Public API (outside tim_detail, inside boost::sort):
template <typename Iter, typename Compare> void timsort(Iter, Iter, Compare);
template <typename Iter>                  void timsort(Iter, Iter);
} // namespace sort
} // namespace boost
```

Rules:
- `static_assert` for iterator category is in the public `timsort()` overload only
- No `using namespace` anywhere in the header
- All internal helpers are `static` functions within `tim_detail`
- `#pragma once` guard (not `#ifndef`) — consistent with spinsort.hpp

### P3 — Gallop Tie-Breaking Rule

Left-side element always wins on equality — this is what preserves stability.

- `gallop_left` uses `comp(key, base[mid])` — stops when false (key not less than mid)
- `gallop_right` uses `comp(base[mid], key)` — stops when true

This asymmetry is the mechanism for stability. Reversing either comparison
silently breaks the stability guarantee with no compile error.

### P4 — typedef vs using

```cpp
// Correct at namespace scope:
template <typename Iter>
using iter_value_t = typename std::iterator_traits<Iter>::value_type;

// Correct in function bodies (wider C++11 compat):
typedef typename std::iterator_traits<Iter>::value_type value_type;

// Avoid in function bodies (may warn on some C++11 compilers):
// using value_type = typename std::iterator_traits<Iter>::value_type;
```

### P5 — Merge Direction Condition

Use `<=` (not `<`):

```cpp
if (len1 <= len2)
    tim_detail::merge_lo(base1, len1, base2, len2, state, comp);
else
    tim_detail::merge_hi(base1, len1, base2, len2, state, comp);
```

### P6 — Test Structure

`test/test_timsort.cpp` function organization (exact):

```cpp
void test_correctness();   // FR-10: random, sorted, reverse, duplicates
void test_stability();     // FR-11: index-tagged pairs
void test_edge_cases();    // FR-12: N=0,1,2, all-equal, reversed
void test_type_coverage(); // FR-13: int, std::string, custom struct+comp
void test_adversarial();   // gallop exit, stack boundary, minrun boundary

int test_main(int, char*[]) {
    test_correctness();
    test_stability();
    test_edge_cases();
    test_type_coverage();
    test_adversarial();
    return 0;
}
```

Use `BOOST_REQUIRE` when sort output size is wrong (remaining assertions meaningless).
Use `BOOST_CHECK` for individual element/property assertions after a successful sort.

### P7 — Benchmark Column Format

Add timsort as a new column alongside existing algorithms — do not change
existing algorithm output. Document the nearly-sorted input immediately above
its benchmark case:

```cpp
// Nearly-sorted: sorted array, N*0.05 random adjacent swaps, mt19937(42)
```

### Enforcement Guidelines

**All AI agents MUST:**
- Use exact function names from P1 — no synonyms
- Place `static_assert` only in the public overload
- Apply gallop tie-breaking from P3 exactly
- Use `<=` in the merge direction condition (P5)
- Follow typedef vs using convention from P4
- Never use `buffer.resize(n)` — only `reserve` + `push_back`/`emplace_back`
- Run all tests under ASAN + UBSAN before marking a story done

**Anti-patterns:**
- `buffer.resize(n)` — breaks non-default-constructible value types
- `!comp(b,a) && !comp(a,b)` for equality — just use `!comp(b, a)`
- `using namespace` in the header
- Reading moved-from elements after `std::move_if_noexcept`
- `Iter::value_type` — always use `std::iterator_traits<Iter>::value_type`

---

## Project Structure & Boundaries

### Complete File Tree

All changes are confined to `libs/sort/` (the git submodule). No superproject files are touched.

```
libs/sort/
├── include/boost/sort/
│   ├── sort.hpp                    MODIFY — add one #include line (FR-7)
│   └── timsort/
│       └── timsort.hpp             CREATE — full implementation (FR-1–9, FR-18)
├── test/
│   └── test_timsort.cpp            CREATE — test suite (FR-10–13)
├── example/
│   └── timsort_example.cpp         CREATE — usage example (FR-19)
├── benchmark/single/
│   ├── benchmark_numbers.cpp       MODIFY — add timsort column (FR-14–16)
│   └── benchmark_strings.cpp       MODIFY — add timsort column (FR-14–16)
└── README.md                       MODIFY — add timsort row to table (FR-17)
```

No Jamfile, CMakeLists.txt, or `.gitmodules` changes — `test_timsort.cpp` is
picked up by the existing wildcard build rules.

### Internal Layout of timsort.hpp

```
[Boost license header + file doc comment]
#pragma once
#include <algorithm>
#include <cstddef>
#include <functional>
#include <iterator>
#include <utility>
#include <vector>

namespace boost { namespace sort {
namespace tim_detail {
  // 1. iter_value_t / iter_diff_t aliases
  // 2. compute_minrun
  // 3. binary_insertion_sort
  // 4. count_run
  // 5. gallop_left / gallop_right
  // 6. SortState<V> struct + RunEntry + RunStack typedef
  // 7. merge_lo / merge_hi
  // 8. merge_collapse / merge_force_collapse
} // namespace tim_detail

// Public API
template <typename Iter, typename Compare> void timsort(Iter, Iter, Compare);
template <typename Iter>                  void timsort(Iter, Iter);
} } // namespace boost::sort
```

### Requirements-to-File Mapping

| FR | File | Content |
|---|---|---|
| FR-1,2 | timsort.hpp | Two public overloads + static_assert |
| FR-3 | timsort.hpp | `tim_detail::count_run()` |
| FR-4 | timsort.hpp | Algorithm correctness — verified by tests |
| FR-5 | timsort.hpp | Early-exit guard: `if (n < 2) return;` |
| FR-6 | timsort/timsort.hpp | The file at this exact path |
| FR-7 | sort.hpp | One `#include` line added |
| FR-8 | timsort.hpp | `namespace boost { namespace sort { ... } }` |
| FR-9 | timsort.hpp | Stdlib-only includes (6 headers) |
| FR-10 | test_timsort.cpp | `test_correctness()` |
| FR-11 | test_timsort.cpp | `test_stability()` |
| FR-12 | test_timsort.cpp | `test_edge_cases()` |
| FR-13 | test_timsort.cpp | `test_type_coverage()` |
| FR-14–16 | benchmark_numbers.cpp, benchmark_strings.cpp | Timsort column, 5 shapes, 4 algos |
| FR-17 | README.md | New algorithm table row |
| FR-18 | timsort.hpp | Top-of-file comment block |
| FR-19 | example/timsort_example.cpp | 5-10 line working example |

### Architectural Boundaries

**Namespace boundary:** `tim_detail` (private) ↔ `boost::sort` (public API — two overloads only)

**Dependency boundary:** timsort.hpp → stdlib only; no Boost headers

**Data flow:**
```
timsort(first, last, comp)
  → SortState<value_type> on stack; buffer.reserve(n/2)
  → compute_minrun(n)
  → scan loop: count_run → binary_insertion_sort → push → merge_collapse
  → merge_force_collapse
  → SortState destroyed; range sorted in-place
```

---

## Architecture Validation Results

### Coherence Validation ✅

All decisions are mutually consistent: C++11 throughout, header-only enforced by
stdlib-only includes, Google C++ style applied uniformly, single-file layout
matches the complexity level, SortState ties the four subsystems together cleanly.

### Requirements Coverage Validation ✅

All 19 FRs are architecturally supported (see Requirements-to-File Mapping above).

NFR coverage:
- C++11 min: enforced by include policy and typedef/using rules
- ASAN+UBSAN: mandatory in enforcement guidelines (P-rules)
- Zero warnings: enforced via static_assert and type-alias discipline
- SM-1 (≥2× on nearly-sorted): enabled by canonical galloping + minrun + 95%-sorted definition
- SM-3 (≤20% on random): guaranteed by canonical merge policy matching reference implementations

### Implementation Readiness Validation ✅

Every story can be implemented independently following this document:
- Function signatures are specified exactly (P1)
- File layout is unambiguous (P2, project structure section)
- Tie-breaking rule is explicit (P3)
- Test structure is templated (P6)
- Benchmark extension format is specified (P7)

### Gap Analysis Results

**Critical gaps:** None.

**Minor gaps (non-blocking):**
- Jamfile wildcard behavior not verified by reading the file — assumed from convention;
  implementer should confirm `test/test_timsort.cpp` is picked up before marking done
- `test_adversarial()` is not a named FR but is required to catch gallop/stack bugs;
  included in test structure (P6) and implementation sequence (D4 impact analysis)

### Architecture Completeness Checklist

**Requirements Analysis**
- [x] Project context thoroughly analyzed
- [x] Scale and complexity assessed
- [x] Technical constraints identified
- [x] Cross-cutting concerns mapped

**Architectural Decisions**
- [x] Critical decisions documented
- [x] Technology stack fully specified (C++11, clang, CMake/b2, stdlib-only)
- [x] Integration patterns defined (sort.hpp inclusion, Jamfile wildcard)
- [x] Performance considerations addressed (SM-1/SM-3, galloping, minrun formula, nearly-sorted definition)

**Implementation Patterns**
- [x] Naming conventions established (P1 canonical function names)
- [x] Structure patterns defined (P2 namespace + file layout)
- [x] Communication patterns specified (SortState data flow)
- [x] Process patterns documented (P6 test structure, P7 benchmark format)

**Project Structure**
- [x] Complete directory structure defined
- [x] Component boundaries established (tim_detail vs boost::sort)
- [x] Integration points mapped (FR-to-file table)
- [x] Requirements to structure mapping complete

### Architecture Readiness Assessment

**Overall Status:** READY FOR IMPLEMENTATION

**Confidence Level:** High

**Key Strengths:**
- Exact function names, signatures, and internal ordering specified — no ambiguity for agents
- All 19 FRs mapped to specific files
- Stability invariant mechanically specified (P3 gallop tie-breaking) — the hardest correctness property to get right
- Performance success criteria are quantitative and reproducible (95%-sorted definition, mt19937 seed 42)

**Areas for Future Enhancement (post-v1):**
- detail/ subdirectory refactor if implementation exceeds ~900 lines
- Merge buffer cap + in-place fallback for very large N
- C++20 ranges/concepts interface
- Parallel timsort variant

### Implementation Handoff

**First implementation priority:** Create `libs/sort/include/boost/sort/timsort/timsort.hpp`
starting with `compute_minrun` and `count_run` — these are standalone, testable in isolation,
and unblock all subsequent work.

**AI Agent Guidelines:**
- Follow canonical function names from P1 exactly
- Implement and test `compute_minrun` assertions before writing the main sort loop
- Verify stability invariant with targeted merge-level tests (not just end-to-end)
- Run ASAN + UBSAN before marking any story complete
