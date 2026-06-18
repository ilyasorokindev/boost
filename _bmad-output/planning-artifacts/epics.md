---
stepsCompleted: [1, 2, 3]
inputDocuments:
  - _bmad-output/planning-artifacts/prds/prd-Boost-2026-06-17/prd.md
  - _bmad-output/planning-artifacts/architecture.md
---

# boost::sort::timsort - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for boost::sort::timsort, decomposing the requirements from the PRD and Architecture into implementable stories.

## Requirements Inventory

### Functional Requirements

FR-1: The developer can call `boost::sort::timsort(first, last)` on any range defined by two random-access iterators, using `operator<` as the comparator. Result is sorted ascending; equal elements retain relative order; compiles C++11.

FR-2: The developer can call `boost::sort::timsort(first, last, comp)` with any strict-weak-ordering comparator. Result sorted per `comp`; stability holds for elements where `!comp(a,b) && !comp(b,a)`.

FR-3: The algorithm detects and exploits Natural Runs. On an already-sorted range of N elements, comparisons are O(N). On K equal-length sorted segments, performance scales with K.

FR-4: Complexity guarantees — best case O(N) comparisons (already sorted), average/worst O(N log N), auxiliary memory O(N) (≤N/2 merge buffer), run stack depth O(log N).

FR-5: Edge cases handled without error: empty range, single element, two elements, all elements equal, all elements in reverse order.

FR-6: Implementation lives at `include/boost/sort/timsort/timsort.hpp`; standalone `#include <boost/sort/timsort/timsort.hpp>` compiles with no other changes to the library.

FR-7: `include/boost/sort/sort.hpp` is updated to `#include <boost/sort/timsort/timsort.hpp>`; `boost::sort::timsort` is exposed via the cumulative header; no existing symbol is broken.

FR-8: All public symbols are in namespace `boost::sort`; no global namespace pollution.

FR-9: `timsort.hpp` includes only stdlib headers: `<algorithm>`, `<cstddef>`, `<functional>`, `<iterator>`, `<utility>`, `<vector>` — no Boost headers, no external dependencies.

FR-10: Correctness tests covering: random data, already-sorted, reverse-sorted, many duplicates, single-run. Both `timsort(first,last)` and `timsort(first,last,comp)` exercised; `std::is_sorted` assertion after each sort.

FR-11: Stability test using index-tagged pairs `{key, original_index}`; after sort by key, `original_index` is non-decreasing within equal-key groups.

FR-12: Edge-case tests for N=0, N=1, N=2, all-equal elements, reversed range; no crash, no UB (run under ASAN + UBSAN).

FR-13: Type coverage tests for `int`, `std::string`, and a user-defined struct with custom comparator; all compile and pass.

FR-14: Benchmarks run on 5 data shapes: random, already-sorted, reverse-sorted, nearly-sorted (1% perturbation, mt19937 seed=42), pipe-organ. All 5 shapes × 4 algorithms produce timing output without crash.

FR-15: Benchmark compares: `boost::sort::timsort`, `boost::sort::spinsort`, `boost::sort::flat_stable_sort`, `std::stable_sort`. Results table has 4 columns × 5 rows.

FR-16: Benchmarks run on `int` and `std::string`; output includes separate tables for each type.

FR-17: `libs/sort/README.md` single-thread algorithm table gains a `timsort` row with: Stable=yes, Additional memory=N/2, Best/avg/worst=N / N log N / N log N, Comparison method=Comparison operator.

FR-18: `timsort.hpp` has a top-of-file comment with: algorithm name, stability, complexity, memory, author/origin reference (Tim Peters 2002), Boost Software License header.

FR-19: A usage example (5-10 lines) in `example/timsort_example.cpp` that compiles and runs correctly.

### NonFunctional Requirements

NFR-1: C++11 minimum compatibility — all code (header, tests, benchmarks) compiles with `-std=c++11`; no C++14/17/20 features used.

NFR-2: Header-only delivery — no compiled sources, no new build targets, no new binaries.

NFR-3: Zero warnings under `-std=c++11 -Wall -Wextra -Wpedantic` (SM-4).

NFR-4: ASAN + UBSAN clean — all test runs must pass with both sanitizers enabled (SM-2).

NFR-5: Performance — on nearly-sorted data (95%+ pre-sorted, N=1M `int`), timsort is ≥2× faster than `spinsort` (SM-1).

NFR-6: Performance floor — on random `int` data (N=1M), timsort is within 20% of `spinsort` wall-clock time (SM-3).

NFR-7: Single-threaded only; no parallel variant.

NFR-8: Random-access iterator requirement — non-random-access iterators are explicitly unsupported.

NFR-9: No external Boost library review process required — personal/team use scope.

### Additional Requirements

- **File layout pattern**: Follow spinsort/pdqsort single-file monolithic header pattern. All implementation in one file `timsort.hpp` (~500–700 lines). Internal structure: run detection → insertion sort → merge stack → merge engine → public API, separated by section comments.

- **Namespace structure**: All internal helpers in `namespace boost::sort::tim_detail`. Public API (two overloads) in `namespace boost::sort` outside `tim_detail`. No `using namespace` anywhere in the header. `#pragma once` guard (not `#ifndef`).

- **Canonical function names (P1 — mandatory, no synonyms)**:
  - `compute_minrun(n) → iter_diff_t`
  - `count_run(first, last, comp) → iter_diff_t`
  - `binary_insertion_sort(first, last, start, comp)`
  - `gallop_left(key, base, len, hint, comp) → iter_diff_t`
  - `gallop_right(key, base, len, hint, comp) → iter_diff_t`
  - `merge_lo(base1, len1, base2, len2, state, comp)`
  - `merge_hi(base1, len1, base2, len2, state, comp)`
  - `merge_collapse(stack, data, state, comp)`
  - `merge_force_collapse(stack, data, state, comp)`
  - SortState members: `buffer`, `min_gallop`
  - RunStack entry fields: `base` (Iter), `len` (iter_diff_t)

- **SortState struct**: Template struct `SortState<ValueType>` with `std::vector<ValueType> buffer` and `int min_gallop`. Instantiated once on the stack in the public `timsort()` call; passed by reference to all merge functions. `buffer.reserve(n/2)` at sort start; `buffer.clear()` between merges; never grown after initial reserve. Use `push_back`/`emplace_back` only — never `buffer.resize(n)`.

- **Merge direction**: Bidirectional — `if (len1 <= len2)` call `merge_lo` (copy left into buffer), else call `merge_hi` (copy right into buffer). Use `<=` strictly (P5).

- **Run stack invariants**: Canonical two-invariant policy — enforce `len[n-1] > len[n]` and `len[n-2] > len[n-1] + len[n]` before each push; merge loop until both hold.

- **Gallop tie-breaking (P3 — stability mechanism)**: `gallop_left` uses `comp(key, base[mid])` (stops when false); `gallop_right` uses `comp(base[mid], key)` (stops when true). Left-side element always wins on equality. Reversing either comparison silently breaks stability.

- **Minrun formula**: Canonical Python-derived formula — 6 MSBs of N, round up if any lower bits are set; result always in [32, 64].

- **MIN_GALLOP**: Adaptive, starts at 7, resets per sort call. Must live in SortState — not global/static/thread_local.

- **iterator_traits aliases (D3.3 — mandatory throughout)**:
  ```cpp
  template <typename Iter>
  using iter_value_t = typename std::iterator_traits<Iter>::value_type;
  template <typename Iter>
  using iter_diff_t  = typename std::iterator_traits<Iter>::difference_type;
  ```
  Never use `Iter::value_type` directly.

- **Move-vs-copy for merge buffer**: `std::move_if_noexcept(*src)` when copying elements into buffer.

- **static_assert for iterator enforcement**: In the public overloads only (not in `tim_detail`), using `std::is_base_of<std::random_access_iterator_tag, iterator_category>`.

- **typedef vs using (P4)**: At namespace scope: `using` aliases. In function bodies: `typedef`. Avoid `using value_type = ...` in function bodies.

- **Test framework**: Boost.Test with `#include <boost/test/included/test_exec_monitor.hpp>` and `#include <boost/test/test_tools.hpp>`. Matches `test/test_spinsort.cpp` pattern exactly.

- **Test structure (P6 — exact function names)**:
  `test_correctness()`, `test_stability()`, `test_edge_cases()`, `test_type_coverage()`, `test_adversarial()` called from `test_main(int, char*[])`. Use `BOOST_REQUIRE` when output size is wrong; `BOOST_CHECK` for individual assertions.

- **Benchmark extension**: Add timsort as a new column to `benchmark/single/benchmark_numbers.cpp` and `benchmark/single/benchmark_strings.cpp`; no new binary. Nearly-sorted input: sorted array, N×0.05 random adjacent swaps, mt19937(42). Document the seed and swap count immediately above the nearly-sorted benchmark case.

- **ASAN + UBSAN mandatory**: Run all tests with both sanitizers before marking any story done.

- **Implementation sequence (order matters)**:
  1. `compute_minrun` (standalone)
  2. `SortState<V>` struct
  3. `binary_insertion_sort` helper
  4. `count_run`
  5. `merge_collapse` + run stack invariant checker
  6. `merge_lo` / `merge_hi` with galloping
  7. Public `timsort()` overloads
  8. `sort.hpp` update
  9. `test_timsort.cpp`
  10. Benchmark extension

- **Files to create**: `timsort.hpp` (CREATE), `test_timsort.cpp` (CREATE), `example/timsort_example.cpp` (CREATE)
- **Files to modify**: `sort.hpp` (one `#include` line), `benchmark_numbers.cpp`, `benchmark_strings.cpp`, `README.md`
- **No Jamfile/CMakeLists.txt changes** — `test_timsort.cpp` picked up by existing wildcard build rules (implementer to verify before marking done).

### UX Design Requirements

N/A — no UX Design document exists for this project (library algorithm, no user interface).

### FR Coverage Map

FR-1:  Epic 1 — timsort(first, last) public overload
FR-2:  Epic 1 — timsort(first, last, comp) public overload
FR-3:  Epic 1 — count_run() natural run detection
FR-4:  Epic 1 — algorithm correctness (complexity by construction)
FR-5:  Epic 1 — early-exit guard: if (n < 2) return
FR-6:  Epic 1 — file at include/boost/sort/timsort/timsort.hpp
FR-7:  Epic 1 — one-line #include added to sort.hpp
FR-8:  Epic 1 — namespace boost::sort, no global pollution
FR-9:  Epic 1 — stdlib-only includes (6 headers)
FR-10: Epic 2 — test_correctness() in test_timsort.cpp
FR-11: Epic 2 — test_stability() in test_timsort.cpp
FR-12: Epic 2 — test_edge_cases() + ASAN+UBSAN in test_timsort.cpp
FR-13: Epic 2 — test_type_coverage() in test_timsort.cpp
FR-14: Epic 3 — 5 data shapes in benchmark_numbers.cpp + benchmark_strings.cpp
FR-15: Epic 3 — 4-way comparison column in benchmark files
FR-16: Epic 3 — int and std::string tables in benchmark output
FR-17: Epic 4 — timsort row added to libs/sort/README.md
FR-18: Epic 1 — top-of-file comment block in timsort.hpp
FR-19: Epic 4 — example/timsort_example.cpp

## Epic List

### Epic 1: Algorithm Core — Usable `timsort` in the Library
Developers can `#include <boost/sort/sort.hpp>` (or the direct header) and call `boost::sort::timsort(first, last)` and `boost::sort::timsort(first, last, comp)` on any random-access iterator range. The algorithm correctly exploits natural runs (O(N) on already-sorted data), handles all edge cases, uses O(N) memory, and is stable.
**FRs covered:** FR-1, FR-2, FR-3, FR-4, FR-5, FR-6, FR-7, FR-8, FR-9, FR-18

### Epic 2: Test Suite — Verified Correctness and Stability
Developers and contributors can run a dedicated test suite that proves timsort is correct on all data shapes and types, preserves stability, handles edge cases without undefined behaviour, and is clean under ASAN + UBSAN.
**FRs covered:** FR-10, FR-11, FR-12, FR-13

### Epic 3: Benchmarks — Demonstrated Adaptive Performance
Developers can run benchmarks comparing timsort against spinsort, flat_stable_sort, and std::stable_sort across 5 data shapes and 2 element types. The primary value proposition (SM-1: ≥2× on nearly-sorted) is empirically verifiable.
**FRs covered:** FR-14, FR-15, FR-16

### Epic 4: Documentation — Discoverable and Usable Algorithm
Developers can find timsort in the README algorithm table, understand its properties and API from the header comment, and run a working usage example.
**FRs covered:** FR-17, FR-19

---

## Epic 1: Algorithm Core — Usable `timsort` in the Library

Developers can `#include <boost/sort/sort.hpp>` (or the direct header) and call `boost::sort::timsort(first, last)` and `boost::sort::timsort(first, last, comp)` on any random-access iterator range. The algorithm correctly exploits natural runs (O(N) on already-sorted data), handles all edge cases, uses O(N) memory, and is stable.

### Story 1.1: Header File Skeleton, Foundational Types, and `compute_minrun`

As a C++ developer contributing to boost::sort,
I want the `timsort.hpp` file created with its full structural skeleton, foundational data structures (`SortState`, `RunEntry`, `RunStack`, iterator aliases), and the `compute_minrun` function,
So that all subsequent implementation stories have a compilable, correctly-structured base to build upon.

**Acceptance Criteria:**

**Given** the file `libs/sort/include/boost/sort/timsort/timsort.hpp` is created with `#pragma once`, the six stdlib includes (`<algorithm>`, `<cstddef>`, `<functional>`, `<iterator>`, `<utility>`, `<vector>`), and namespace structure `boost::sort::tim_detail`
**When** compiled standalone with `-std=c++11 -Wall -Wextra -Wpedantic`
**Then** it compiles with zero warnings and only those six `#include` lines are present (no Boost headers)

**Given** `iter_value_t<Iter>` and `iter_diff_t<Iter>` alias templates defined inside `tim_detail` using `std::iterator_traits`
**When** code inside `tim_detail` uses `iter_value_t<Iter>` to name a type
**Then** it resolves correctly; `Iter::value_type` is never used directly anywhere in the file

**Given** `template <typename V> struct SortState` with members `std::vector<V> buffer` and `int min_gallop`
**When** a `SortState<int>` is default-constructed
**Then** `min_gallop` is initialized to 7; `buffer` is empty

**Given** a `RunEntry` struct with fields `Iter base` and `iter_diff_t<Iter> len`, and `RunStack` defined as a `std::vector<RunEntry>`
**When** accessed
**Then** field names are exactly `base` and `len` (no synonyms)

**Given** `tim_detail::compute_minrun(n)` implemented using the canonical Python-derived formula (shift right until `n < 64`, OR-ing in any bits shifted off)
**When** called with `n` in [1, 63]
**Then** returns `n` unchanged

**When** called with `n = 64`
**Then** returns 32

**When** called with `n = 65`
**Then** returns 33

**When** called with any `n >= 64`
**Then** result is always in the range [32, 64] inclusive

---

### Story 1.2: Run Detection — `count_run` and `binary_insertion_sort`

As a C++ developer contributing to boost::sort,
I want `tim_detail::count_run` and `tim_detail::binary_insertion_sort` implemented,
So that the algorithm can detect natural sorted order in the input and extend short runs to minrun length before merging.

**Acceptance Criteria:**

**Given** `count_run(first, last, comp)` called on a strictly ascending range `[1, 2, 3, 4, 5]`
**When** it returns
**Then** returns 5; range is unmodified

**Given** `count_run` called on a strictly descending range `[5, 4, 3, 2, 1]`
**When** it returns
**Then** returns 5 and the range is reversed in-place to `[1, 2, 3, 4, 5]`

**Given** `count_run` called on `[3, 1, 4, 1, 5]`
**When** it returns
**Then** returns 1 (first element only — second is smaller, so the ascending run is length 1)

**Given** `count_run` called on `[3, 3, 3, 2, 1]` (equal prefix, then descending)
**When** it returns
**Then** treats equal-element prefix as ascending (not reversed), returning at least 3; stability is not broken by treating equal elements as a valid ascending run

**Given** `binary_insertion_sort(first, last, start, comp)` where `[first, start)` is already sorted and `start < last`
**When** called
**Then** all elements in `[first, last)` are sorted correctly per `comp`

**Given** `binary_insertion_sort` called on a range of `std::pair<int,int>` sorted by first element, with duplicate keys
**When** extending the sorted portion
**Then** pairs with equal first elements retain their original relative order (stability preserved by the left-biased binary search)

**Given** both functions compiled with `-std=c++11 -Wall -Wextra -Wpedantic`
**When** compiled
**Then** zero warnings

---

### Story 1.3: Merge Engine — `gallop_left`, `gallop_right`, `merge_lo`, `merge_hi`, `merge_collapse`, `merge_force_collapse`

As a C++ developer contributing to boost::sort,
I want the complete merge subsystem implemented with galloping mode and run stack invariant enforcement,
So that runs are merged efficiently while maintaining stability and O(log N) stack depth.

**Acceptance Criteria:**

**Given** `gallop_left(key, base, len, hint, comp)` where `key` is greater than all elements in `[base, base+len)`
**When** called
**Then** returns `len`

**Given** `gallop_right(key, base, len, hint, comp)` where `key` equals element at position `p`
**When** called
**Then** returns `p` (not `p+1`) — left-run element wins on equality, preserving stability (P3 tie-breaking rule)

**Given** `merge_lo(base1, len1, base2, len2, state, comp)` called with `len1 <= len2` and two adjacent sorted runs
**When** the merge completes
**Then** `[base1, base1+len1+len2)` is sorted; equal elements from the left run precede equal elements from the right run; `state.buffer` was used but never grown past its initial `reserve(n/2)` capacity

**Given** `merge_hi(base1, len1, base2, len2, state, comp)` called with `len1 > len2`
**When** the merge completes
**Then** merged range is sorted with stability preserved; right run was copied into buffer; `state.buffer.resize` was never called

**Given** `state.min_gallop` starts at 7 and one run wins 7 or more consecutive element comparisons
**When** galloping mode activates
**Then** `state.min_gallop` is decremented; when neither side dominates, it is incremented back toward 7 (adaptive per-call behaviour, not global/static)

**Given** a run stack with three entries where `len[n-2] <= len[n-1] + len[n]`
**When** `merge_collapse` is called
**Then** merges occur in the correct order (smaller pair merged first) until both `len[n-1] > len[n]` and `len[n-2] > len[n-1] + len[n]` hold

**Given** `merge_force_collapse` called after the full input has been scanned
**When** called with N runs remaining on the stack
**Then** performs N-1 merges, always merging the top two runs until exactly one remains; the final range is fully sorted

**Given** `buffer.push_back(std::move_if_noexcept(*src))` is used throughout
**When** the value type has a throwing move constructor
**Then** elements are copied (not moved) into the buffer, preserving source range integrity

---

### Story 1.4: Public API Overloads and Library Integration

As a C++ developer using boost::sort,
I want `boost::sort::timsort(first, last)` and `boost::sort::timsort(first, last, comp)` available via both the direct header and `sort.hpp`, with a proper file comment and Boost license header,
So that I can call timsort with the same ergonomics as every other boost::sort algorithm.

**Acceptance Criteria:**

**Given** a C++11 program with `#include <boost/sort/timsort/timsort.hpp>` calling `boost::sort::timsort(v.begin(), v.end())` on a `std::vector<int>`
**When** compiled with `-std=c++11 -Wall -Wextra -Wpedantic`
**Then** compiles with zero warnings and sorts correctly (FR-6)

**Given** a C++11 program with `#include <boost/sort/sort.hpp>` calling `boost::sort::timsort(...)`
**When** compiled and run
**Then** the symbol resolves and the sort is correct — timsort is exposed via the cumulative header (FR-7)

**Given** `boost::sort::timsort(list.begin(), list.end())` where `list` is a `std::list<int>`
**When** compiled
**Then** static_assert fires with the message `"boost::sort::timsort requires RandomAccessIterator"` (D2.1 — assert in public overload only)

**Given** the zero-comparator overload `timsort(Iter first, Iter last)`
**When** inspecting the implementation
**Then** it calls the two-argument overload with `std::less<value_type>()` (not `std::less<>()`) — C++11 compliant (D2.2)

**Given** `timsort.hpp` top-of-file comment block
**When** inspected
**Then** contains: Boost Software License 1.0 text, algorithm name ("timsort"), stability ("stable"), complexity ("O(N) best / O(N log N) average and worst"), memory ("O(N), merge buffer ≤ N/2 elements"), origin ("Tim Peters, 2002"), and the refactor threshold note (`// Refactor to detail/ subdirectory if this file exceeds ~900 lines`) (FR-18)

**Given** `sort.hpp` after modification
**When** `grep "#include.*timsort" sort.hpp` is run
**Then** exactly one matching line exists; no existing `#include` lines are removed or reordered (FR-7)

**Given** `grep -rE "using namespace" libs/sort/include/boost/sort/timsort/` is run
**When** executed
**Then** zero matches (no `using namespace` in any header) (P2 rule)

---

## Epic 2: Test Suite — Verified Correctness and Stability

Developers and contributors can run a dedicated test suite that proves timsort is correct on all data shapes and types, preserves stability, handles edge cases without undefined behaviour, and is clean under ASAN + UBSAN.

### Story 2.1: Correctness and Stability Tests

As a C++ developer contributing to boost::sort,
I want `test/test_timsort.cpp` created with `test_correctness()` and `test_stability()` functions using Boost.Test,
So that I can verify timsort produces correctly sorted output and preserves the relative order of equal elements across representative data shapes.

**Acceptance Criteria:**

**Given** `test/test_timsort.cpp` created following the pattern of `test/test_spinsort.cpp` with `#include <boost/test/included/test_exec_monitor.hpp>` and `#include <boost/test/test_tools.hpp>`
**When** compiled with `-std=c++11 -Wall -Wextra -Wpedantic`
**Then** zero warnings; the file includes `<boost/sort/timsort/timsort.hpp>` directly

**Given** `test_correctness()` called via `test_main`
**When** run
**Then** all of the following pass with `std::is_sorted` assertion after each sort: randomly shuffled `std::vector<int>`, already-sorted `std::vector<int>`, reverse-sorted `std::vector<int>`, `std::vector<int>` with many duplicates, a single contiguous sorted segment

**Given** both `timsort(first, last)` and `timsort(first, last, comp)` overloads
**When** exercised in `test_correctness()`
**Then** both are tested (not just one overload)

**Given** `test_stability()` using `std::vector<std::pair<int,int>>` where `first` is the sort key and `second` is the original index, with deliberately repeated keys
**When** sorted by `pair.first` using a comparator on the first element only
**Then** for every group of equal keys, `pair.second` values appear in non-decreasing order (original relative order preserved)

**Given** `BOOST_REQUIRE` and `BOOST_CHECK` usage
**When** reviewing `test_correctness()` and `test_stability()`
**Then** `BOOST_REQUIRE` guards size/existence checks; `BOOST_CHECK` used for individual element and property assertions

**Given** the test binary built and run via the existing ctest or b2 test target
**When** executed
**Then** `test_correctness()` and `test_stability()` both pass; return code 0; confirm Jamfile wildcard picks up `test_timsort.cpp` (if not, document the fix needed)

---

### Story 2.2: Edge-Case, Type Coverage, and Adversarial Tests

As a C++ developer contributing to boost::sort,
I want `test_edge_cases()`, `test_type_coverage()`, and `test_adversarial()` added to `test_timsort.cpp`,
So that I can verify timsort is safe under boundary conditions, compiles for multiple value types, and correctly handles inputs that stress galloping mode and run-stack logic — with ASAN + UBSAN confirming no memory or undefined-behaviour errors.

**Acceptance Criteria:**

**Given** `test_edge_cases()` called
**When** run
**Then** passes for: empty range `[first, first)` (no-op, no crash), single element (no-op), two elements in both orderings (one comparison, correct result), all-equal elements (sorted, stable), fully reversed input (correctly sorted)

**Given** the full test binary run with ASAN enabled (`-fsanitize=address`)
**When** run
**Then** zero AddressSanitizer errors

**Given** the full test binary run with UBSAN enabled (`-fsanitize=undefined`)
**When** run
**Then** zero UndefinedBehaviorSanitizer errors (no signed overflow in index arithmetic, no out-of-bounds on merge buffer, no reads from moved-from elements)

**Given** `test_type_coverage()` called
**When** run
**Then** timsort compiles and produces correctly sorted output for: `int`, `std::string`, and a user-defined struct `Rec { int key; std::string val; }` sorted by a custom comparator on `key` (FR-13)

**Given** `test_adversarial()` called
**When** run
**Then** the following inputs are exercised without crash or incorrect output: an input designed to trigger galloping mode (one run dominates 7+ consecutive comparisons) then exit galloping, an input that simultaneously violates both run-stack invariants (requiring careful merge ordering), inputs at the minrun boundary (N=63 and N=64)

**Given** the five test functions (`test_correctness`, `test_stability`, `test_edge_cases`, `test_type_coverage`, `test_adversarial`) called in order from `test_main(int, char*[])`
**When** run
**Then** all pass; function names match P6 exactly

---

## Epic 3: Benchmarks — Demonstrated Adaptive Performance

Developers can run benchmarks comparing timsort against spinsort, flat_stable_sort, and std::stable_sort across 5 data shapes and 2 element types. The primary value proposition (SM-1: ≥2× on nearly-sorted) is empirically verifiable.

### Story 3.1: Extend `benchmark_numbers.cpp` with Timsort Column

As a C++ developer evaluating sorting algorithms,
I want timsort included as a benchmark column in `benchmark/single/benchmark_numbers.cpp` alongside spinsort, flat_stable_sort, and std::stable_sort across five data shapes for `int`,
So that I can empirically compare timsort's wall-clock performance and verify its adaptive advantage on nearly-sorted integer data.

**Acceptance Criteria:**

**Given** `benchmark/single/benchmark_numbers.cpp` is modified to add a timsort timing column
**When** compiled with `-std=c++11` and run
**Then** produces a results table with exactly 4 algorithm columns (timsort, spinsort, flat_stable_sort, std::stable_sort) × 5 data shape rows (random, already-sorted, reverse-sorted, nearly-sorted, pipe-organ) for `int`; no crash on any shape

**Given** the "nearly-sorted" row
**When** inspecting the source immediately above its timing block
**Then** a comment reads exactly: `// Nearly-sorted: sorted array, N*0.05 random adjacent swaps, mt19937(42)` and the code uses `std::mt19937` seeded with `42` performing `N * 0.05` (integer) random adjacent-pair swaps (P7, D4.3)

**Given** the "pipe-organ" data shape (ascending first half, descending second half)
**When** the benchmark runs
**Then** all 4 algorithms produce timing output without crash

**Given** the existing spinsort, flat_stable_sort, and std::stable_sort columns
**When** comparing modified output to original
**Then** existing column values are unchanged; only the timsort column is new; no existing symbol is renamed or removed

**Given** N = 1,000,000 `int` elements on the nearly-sorted shape
**When** results are observed
**Then** timsort time is expected to be significantly lower than spinsort (targeting SM-1: ≥2×); if not met, the story is still complete but a note is added to the benchmark source flagging it as a calibration issue

---

### Story 3.2: Extend `benchmark_strings.cpp` with Timsort Column

As a C++ developer evaluating sorting algorithms,
I want timsort included as a benchmark column in `benchmark/single/benchmark_strings.cpp` alongside spinsort, flat_stable_sort, and std::stable_sort across five data shapes for `std::string`,
So that I can verify timsort's performance on comparison-expensive element types and confirm the 4-column × 5-shape output required by FR-16.

**Acceptance Criteria:**

**Given** `benchmark/single/benchmark_strings.cpp` is modified to add a timsort timing column
**When** compiled with `-std=c++11` and run
**Then** produces a results table with exactly 4 algorithm columns × 5 data shape rows for `std::string`; no crash on any shape

**Given** the "nearly-sorted" row in the string benchmark
**When** inspecting the source
**Then** uses the same `mt19937(42)` / N×0.05 adjacent-swap pattern and comment as Story 3.1 (consistent across both files)

**Given** the existing string benchmark columns
**When** comparing modified output to original
**Then** existing column values are unchanged; only the timsort column is new

**Given** both benchmark files (from Story 3.1 and this story) compiled and run together
**When** reviewing the combined output
**Then** output includes separate tables for `int` (benchmark_numbers) and `std::string` (benchmark_strings), satisfying FR-16

---

## Epic 4: Documentation — Discoverable and Usable Algorithm

Developers can find timsort in the README algorithm table, understand its properties and API from the header comment, and run a working usage example to orient themselves. The feature is fully self-describing.

### Story 4.1: README Algorithm Table Entry

As a C++ developer browsing the boost::sort library,
I want a timsort row in the single-thread algorithm comparison table in `libs/sort/README.md`,
So that I can immediately see timsort's key properties alongside the other algorithms without having to read the source.

**Acceptance Criteria:**

**Given** `libs/sort/README.md` is modified
**When** the single-thread algorithm table is viewed
**Then** a `timsort` row is present with all of: Stable = yes, Additional memory = N/2, Best = N, Average = N log N, Worst = N log N, Comparison method = Comparison operator (FR-17)

**Given** the existing rows for spinsort, pdqsort, flat_stable_sort, and std::stable_sort
**When** comparing the table before and after
**Then** no existing rows are modified or removed; only the timsort row is added

**Given** the markdown table syntax of the new row
**When** rendered on GitHub
**Then** the row renders correctly within the table without breaking column alignment

---

### Story 4.2: Standalone Usage Example

As a C++ developer new to boost::sort::timsort,
I want a compilable usage example at `libs/sort/example/timsort_example.cpp`,
So that I can orient myself with the API in under a minute and immediately reproduce a working sort call.

**Acceptance Criteria:**

**Given** `libs/sort/example/timsort_example.cpp` is created with 5–10 lines of substantive code
**When** compiled with `g++ -std=c++11 -I<boost-sort-include-path> timsort_example.cpp`
**Then** compiles with zero warnings under `-std=c++11 -Wall -Wextra -Wpedantic` and runs to completion (FR-19)

**Given** the example demonstrates the `timsort(first, last)` overload
**When** viewed
**Then** sorts a `std::vector` and prints the sorted result to stdout

**Given** the example demonstrates the `timsort(first, last, comp)` overload
**When** viewed
**Then** at least one call with a custom comparator (e.g., `std::greater<int>()` or a lambda) is present

**Given** the example's `#include` list
**When** inspected
**Then** includes only `<boost/sort/sort.hpp>` (or `<boost/sort/timsort/timsort.hpp>`), `<vector>`, and `<iostream>`; no additional Boost or external headers
