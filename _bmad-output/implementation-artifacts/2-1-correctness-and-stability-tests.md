---
baseline_commit: "bd6ff8284501338cdd35616e41649604101300b7"
---

# Story 2.1: Correctness and Stability Tests

Status: done

## Story

As a C++ developer contributing to boost::sort,
I want `test/test_timsort.cpp` created with `test_correctness()` and `test_stability()` functions using Boost.Test,
so that I can verify timsort produces correctly sorted output and preserves the relative order of equal elements across representative data shapes.

## Acceptance Criteria

1. **File compiles clean (FR-10):** `test/test_timsort.cpp` created following `test/test_spinsort.cpp` pattern with `#include <boost/test/included/test_exec_monitor.hpp>` and `#include <boost/test/test_tools.hpp>`. Compiled with `-std=c++11 -Wall -Wextra -Wpedantic` → zero warnings. File includes `<boost/sort/timsort/timsort.hpp>` directly (not via `sort.hpp`).

2. **`test_correctness()` passes (FR-10):** Called via `test_main`. All of the following pass with `std::is_sorted` assertion after each sort: randomly shuffled `std::vector<int>`, already-sorted `std::vector<int>`, reverse-sorted `std::vector<int>`, `std::vector<int>` with many duplicates, a single contiguous sorted segment (all ascending, no breaks).

3. **Both overloads exercised (FR-10):** `timsort(first, last)` and `timsort(first, last, comp)` are both tested inside `test_correctness()` — not just one overload.

4. **`test_stability()` passes (FR-11):** Uses `std::vector<std::pair<int,int>>` where `first` is the sort key and `second` is the original index, with deliberately repeated keys. Sorted by `pair.first` via a comparator on the first element only. For every group of equal keys, `pair.second` values appear in non-decreasing order (original relative order preserved).

5. **`BOOST_REQUIRE` / `BOOST_CHECK` usage:** `BOOST_REQUIRE` guards size/existence checks (so remaining assertions aren't meaningless); `BOOST_CHECK` used for individual element/property assertions.

6. **Build system registration:** `test_timsort.cpp` is registered in **both** `test/CMakeLists.txt` and `test/Jamfile.v2` so it is picked up by existing build targets. (**NOTE: Neither file uses wildcards — explicit entries are required.**)

7. **Test binary runs and returns 0:** `test_correctness()` and `test_stability()` both pass; binary exit code 0.

## Tasks / Subtasks

- [x] Task 1: Create `libs/sort/test/test_timsort.cpp` with file header and includes (AC: 1)
  - [x] Copy the file header style from `test_spinsort.cpp` (Boost license header comment block)
  - [x] Include in this exact order: `<algorithm>`, `<iostream>`, `<vector>`, `<random>` (stdlib), then `<boost/sort/timsort/timsort.hpp>`, then `<boost/test/included/test_exec_monitor.hpp>`, `<boost/test/test_tools.hpp>`
  - [x] Declare all five test function prototypes at file top (P6): `void test_correctness();`, `void test_stability();`, `void test_edge_cases();`, `void test_type_coverage();`, `void test_adversarial();` — declare all five even though only the first two are implemented in this story
  - [x] Add stubs for `test_edge_cases()`, `test_type_coverage()`, `test_adversarial()` with a single `// Story 2.2` comment in the body
  - [x] Write `int test_main(int, char*[])` calling all five in P6 order (Story 2.2 stubs will pass trivially)
  - [x] Compile standalone; confirm zero warnings

- [x] Task 2: Implement `test_correctness()` (AC: 2, 3, 5)
  - [x] Random data: `std::mt19937` seeded with 0; generate 10000 `int` elements in `[0, 10000)` via `% NElem`; sort with `timsort(v.begin(), v.end())`; assert `std::is_sorted`
  - [x] Already-sorted: vector `[0..NElem-1]`; sort with `timsort(v.begin(), v.end())`; assert `std::is_sorted`
  - [x] Reverse-sorted: vector `[NElem..1]` (descending); sort with `timsort(v.begin(), v.end())`; assert `std::is_sorted`
  - [x] Many duplicates: fill vector with `v[i] = i % 100`; sort with `timsort(v.begin(), v.end())`; assert `std::is_sorted`
  - [x] Single contiguous sorted run: vector already sorted; verify timsort completes correctly (O(N) path); assert `std::is_sorted`
  - [x] Two-arg overload test: sort a second copy with `timsort(v.begin(), v.end(), std::less<int>())`; compare result to first copy
  - [x] Use `BOOST_REQUIRE(v.size() == NElem)` before per-element assertions; use `BOOST_CHECK(std::is_sorted(...))` for sort result

- [x] Task 3: Implement `test_stability()` (AC: 4, 5)
  - [x] Define `typedef std::pair<int,int> KeyIdx;` and a struct comparator `CmpFirst` (not a lambda — avoids C++11 non-copyable lambda edge cases with template deduction)
  - [x] Build a vector `v1` of 10000 `KeyIdx` pairs: `v1.push_back(KeyIdx(i % 100, i))` — keys 0..99 cycle, second is original index
  - [x] Shuffle `v1` using `std::mt19937(0)` + `std::shuffle`
  - [x] Copy `v1` into `v2` (identical shuffled input for reference sort)
  - [x] Sort `v1` with `timsort(v1.begin(), v1.end(), CmpFirst())`; sort `v2` with `std::stable_sort(v2.begin(), v2.end(), CmpFirst())`
  - [x] `BOOST_REQUIRE(static_cast<int>(v1.size()) == NElem)` first
  - [x] Element-by-element: `BOOST_CHECK(v1[i].first == v2[i].first)` AND `BOOST_CHECK(v1[i].second == v2[i].second)` for every `i`
  - [x] **ANTI-PATTERN:** Do NOT check `v[i].second < v[i+1].second` after shuffling — after shuffle, equal-key groups may have non-monotone original indices, so this check would falsely fail on a correct stable sort. Always compare against `std::stable_sort` as the oracle.

- [x] Task 4: Register test in build system (AC: 6)
  - [x] **CMakeLists.txt** (`libs/sort/test/CMakeLists.txt`): add line `boost_sort_add_test(test_timsort test_timsort.cpp)` after the `test_spinsort` entry
  - [x] **Jamfile.v2** (`libs/sort/test/Jamfile.v2`): add entry `[ run test_timsort.cpp : : : [ requires cxx11_constexpr cxx11_noexcept ] <optimization>speed : test_timsort ]` inside the `test-suite "sort"` block, after the `test_spinsort` entry
  - [x] Do NOT remove or reorder any existing entries

- [x] Task 5: Build and run tests (AC: 7)
  - [x] Compile `test_timsort.cpp` directly with the command from Dev Notes; confirm zero warnings
  - [x] Run the compiled binary; confirm exit code 0 and `test_correctness` + `test_stability` pass
  - [x] Compile and run again with ASAN: `-fsanitize=address`; zero errors
  - [x] Compile and run again with UBSAN: `-fsanitize=undefined`; zero errors

## Dev Notes

### CRITICAL: Build System Is NOT Wildcard

Both `test/CMakeLists.txt` and `test/Jamfile.v2` list each test file **explicitly** — there is no glob pattern. Previous story notes incorrectly assumed wildcard pickup. **You MUST add explicit entries to both files** (Task 4). Failure to do this means the test binary is never built by CI.

Evidence:
- `CMakeLists.txt` lists 11 individual `boost_sort_add_test(...)` calls; no `file(GLOB ...)` or `aux_source_directory`
- `Jamfile.v2` has 9 individual `[ run ... ]` entries; no wildcard pattern

### Boost.Test Pattern (MANDATORY — match test_spinsort.cpp exactly)

```cpp
#include <boost/test/included/test_exec_monitor.hpp>
#include <boost/test/test_tools.hpp>
```

The entry point is `test_main`, NOT `main`:

```cpp
int test_main(int, char*[]) {
    test_correctness();
    test_stability();
    test_edge_cases();
    test_type_coverage();
    test_adversarial();
    return 0;
}
```

`boost/test/included/test_exec_monitor.hpp` defines its own `main()` which calls `test_main()`. Do NOT define `main()` yourself — you will get a link error.

DO NOT use `BOOST_AUTO_TEST_CASE` or `BOOST_TEST_MODULE` — these are from a different Boost.Test mode. Use the `test_exec_monitor` + `test_tools` pattern only.

### Function Declarations vs Definitions (ALL FIVE REQUIRED)

All five function names are specified by P6 (architecture). Declare all five at the top of the file. Implement only the first two in this story. Stub the other three:

```cpp
void test_correctness();
void test_stability();
void test_edge_cases();
void test_type_coverage();
void test_adversarial();

// ... implementations ...

void test_edge_cases()    { /* Story 2.2 */ }
void test_type_coverage() { /* Story 2.2 */ }
void test_adversarial()   { /* Story 2.2 */ }
```

### Current timsort.hpp State (496 lines, fully implemented)

The public API is in place. Use these headers:

```cpp
#include <boost/sort/timsort/timsort.hpp>
// Exposes: boost::sort::timsort(Iter, Iter) and boost::sort::timsort(Iter, Iter, Compare)
// Namespace: boost::sort
// Internal: boost::sort::tim_detail (accessible for future unit tests if needed)
```

Do NOT include `<boost/sort/sort.hpp>` — include the direct header per AC1.

### Compilation Command (from superproject root)

```bash
# Step 1: Compile + link (test_exec_monitor is header-only in "included" mode)
g++ -std=c++11 -Wall -Wextra -Wpedantic \
    -I libs/sort/include \
    -I /path/to/boost/headers \
    libs/sort/test/test_timsort.cpp \
    -o test_timsort_bin

# Step 2: Run
./test_timsort_bin

# Step 3: ASAN
g++ -std=c++11 -Wall -Wextra -Wpedantic -fsanitize=address \
    -I libs/sort/include \
    -I /path/to/boost/headers \
    libs/sort/test/test_timsort.cpp \
    -o test_timsort_asan
./test_timsort_asan

# Step 4: UBSAN
g++ -std=c++11 -Wall -Wextra -Wpedantic -fsanitize=undefined \
    -I libs/sort/include \
    -I /path/to/boost/headers \
    libs/sort/test/test_timsort.cpp \
    -o test_timsort_ubsan
./test_timsort_ubsan
```

For the project's CMake build (from the superproject root or libs/sort):
```bash
cmake -B build -DCMAKE_CXX_STANDARD=11
cmake --build build --target boost_sort_test_timsort
ctest --test-dir build -R boost_sort_test_timsort -V
```

### `test_correctness()` Reference Implementation

```cpp
void test_correctness() {
    typedef std::vector<int> Vec;
    const int NElem = 10000;
    std::mt19937 my_rand(0);

    // Random
    Vec v1;
    for (int i = 0; i < NElem; ++i) v1.push_back(my_rand() % NElem);
    boost::sort::timsort(v1.begin(), v1.end());
    BOOST_REQUIRE(static_cast<int>(v1.size()) == NElem);
    BOOST_CHECK(std::is_sorted(v1.begin(), v1.end()));

    // Already sorted
    Vec v2;
    for (int i = 0; i < NElem; ++i) v2.push_back(i);
    boost::sort::timsort(v2.begin(), v2.end());
    BOOST_CHECK(std::is_sorted(v2.begin(), v2.end()));

    // Reverse sorted
    Vec v3;
    for (int i = 0; i < NElem; ++i) v3.push_back(NElem - i);
    boost::sort::timsort(v3.begin(), v3.end());
    BOOST_CHECK(std::is_sorted(v3.begin(), v3.end()));

    // Many duplicates
    Vec v4;
    for (int i = 0; i < NElem; ++i) v4.push_back(i % 100);
    boost::sort::timsort(v4.begin(), v4.end());
    BOOST_CHECK(std::is_sorted(v4.begin(), v4.end()));

    // Single contiguous sorted run (exercises O(N) path)
    Vec v5;
    for (int i = 0; i < NElem; ++i) v5.push_back(i);
    boost::sort::timsort(v5.begin(), v5.end());
    BOOST_CHECK(std::is_sorted(v5.begin(), v5.end()));

    // Two-arg overload with explicit comparator
    Vec v6;
    for (int i = 0; i < NElem; ++i) v6.push_back(my_rand() % NElem);
    boost::sort::timsort(v6.begin(), v6.end(), std::less<int>());
    BOOST_CHECK(std::is_sorted(v6.begin(), v6.end()));
}
```

### `test_stability()` Reference Implementation

**CRITICAL NOTE:** Do NOT check `v[i].second < v[i+1].second` after shuffling — that checks the pre-shuffle order, not the post-shuffle relative order that stability preserves. The correct approach is to compare element-by-element against `std::stable_sort` (same pattern as `test_spinsort.cpp`).

```cpp
void test_stability() {
    typedef std::pair<int, int> KeyIdx;
    struct CmpFirst {
        bool operator()(const KeyIdx& a, const KeyIdx& b) const {
            return a.first < b.first;
        }
    };
    const int NElem = 10000;
    std::vector<KeyIdx> v1, v2;
    v1.reserve(NElem);
    for (int i = 0; i < NElem; ++i)
        v1.push_back(KeyIdx(i % 100, i));  // keys 0..99, second = original index

    std::mt19937 my_rand(0);
    std::shuffle(v1.begin(), v1.end(), my_rand);

    v2 = v1;  // identical shuffled copy for reference sort

    // Sort v1 with timsort, v2 with std::stable_sort (guaranteed stable)
    boost::sort::timsort(v1.begin(), v1.end(), CmpFirst());
    std::stable_sort(v2.begin(), v2.end(), CmpFirst());

    BOOST_REQUIRE(static_cast<int>(v1.size()) == NElem);
    // Element-by-element comparison: both key AND secondary value must match
    for (int i = 0; i < NElem; ++i) {
        BOOST_CHECK(v1[i].first  == v2[i].first);
        BOOST_CHECK(v1[i].second == v2[i].second);
    }
}
```

**Why lambda for comparator won't work here:** A struct comparator (`CmpFirst`) avoids potential issues with passing lambdas to function templates on some C++11 compilers (lambdas are non-copyable in C++11 edge cases). Use the struct form for stability test comparators.

**Why compare against `std::stable_sort` (not check non-decreasing):** After shuffling, the "original index" (second) values within each equal-key group appear in the shuffled (random) order. A stable sort preserves that shuffled relative order — so checking `v[i].second < v[i+1].second` would be wrong (the shuffled order is not monotone). Comparing element-by-element against `std::stable_sort` is the correct stability oracle.

### CMakeLists.txt — Exact Line to Add

In `libs/sort/test/CMakeLists.txt`, after line:
```cmake
boost_sort_add_test(test_spinsort test_spinsort.cpp)
```

Add:
```cmake
boost_sort_add_test(test_timsort test_timsort.cpp)
```

### Jamfile.v2 — Exact Block to Add

In `libs/sort/test/Jamfile.v2`, after the `test_spinsort` entry:
```
  [ run test_spinsort.cpp
       : : : [ requires
                cxx11_constexpr
                cxx11_noexcept ] <optimization>speed : test_spinsort ]
```

Add:
```
  [ run test_timsort.cpp
       : : : [ requires
                cxx11_constexpr
                cxx11_noexcept ] <optimization>speed : test_timsort ]
```

### Google C++ Style — Apply Consistently

- 2-space indent (match rest of sort library)
- `snake_case` for variables, `PascalCase` for types/structs
- 80-char line limit (soft — don't wrap function calls awkwardly)
- `typedef` in function bodies for local type aliases (P4 — NOT `using T = ...`)
- No `using namespace` in file scope — but per test_spinsort.cpp convention, `using namespace boost::sort;` at file scope is acceptable in test files (not headers). Either approach is fine; choosing `boost::sort::timsort(...)` explicit prefix avoids any ambiguity.

### Previous Story Learnings (1-4 Completion Notes)

- `compute_minrun<Iter>(n)` requires explicit template argument — the `<Iter>` must be provided when calling from outside `tim_detail` where `Iter` cannot be deduced from the function argument alone. In test files calling the public API, this is not relevant (we call `timsort()`, not `compute_minrun`).
- Pre-existing deprecation warnings in `spreadsort/type_traits.hpp` appear when including `<boost/sort/sort.hpp>`. **Do not include `sort.hpp`** — use `<boost/sort/timsort/timsort.hpp>` directly (AC1 says so) to keep warning count at zero.
- `std::less<value_type>()` not `std::less<>()` — irrelevant to tests but consistent reminder.

### Boost.Test `BOOST_REQUIRE` vs `BOOST_CHECK` Policy (AC5)

| Situation | Macro |
|-----------|-------|
| Size check before iterating (`v.size() == NElem`) | `BOOST_REQUIRE` — if wrong, remaining assertions are meaningless |
| `std::is_sorted` after a sort | `BOOST_CHECK` |
| Individual element comparison (`v[i] <= v[i+1]`) | `BOOST_CHECK` |
| Stability property check per pair | `BOOST_CHECK` |

### Project Structure Notes

All changes confined to `libs/sort/` submodule (no superproject files touched):
- CREATE: `libs/sort/test/test_timsort.cpp`
- MODIFY: `libs/sort/test/CMakeLists.txt` (one line added)
- MODIFY: `libs/sort/test/Jamfile.v2` (one block added inside test-suite)

No changes to: `timsort.hpp`, `sort.hpp`, benchmark files, README — those are other stories.

### References

- [Source: architecture.md#D4.1] — Test framework: `test_exec_monitor`, not unit_test_framework
- [Source: architecture.md#P6] — Exact function names and call order in `test_main`
- [Source: architecture.md#FR-10] — `test_correctness()`: random, sorted, reverse, duplicates, single-run
- [Source: architecture.md#FR-11] — `test_stability()`: index-tagged pairs, non-decreasing within equal-key groups
- [Source: architecture.md#D4.1] — `BOOST_REQUIRE` for size checks, `BOOST_CHECK` for assertions
- [Source: epics.md#Story-2.1] — Full acceptance criteria
- [Source: libs/sort/test/test_spinsort.cpp] — File header pattern, test_main signature, include order
- [Source: libs/sort/test/CMakeLists.txt] — Explicit (not wildcard) test registration
- [Source: libs/sort/test/Jamfile.v2] — Explicit (not wildcard) Jamfile entries
- [Source: story 1-4] — Zero-warning policy; use direct timsort header, not sort.hpp

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6 (context engine / create-story)

### Debug Log References

Stability bug discovered in `merge_hi` gallop phase of `timsort.hpp`: two symmetrical errors —
(1) First gallop used `gallop_left` (≥ *buf, includes equal), causing equal left-run elements to be
    placed rightward of the current buffer element, violating stability. Fixed to `gallop_right` (> *buf).
(2) Second gallop used `gallop_right` (> *left, excludes equal), causing equal buffer elements to be
    placed leftward of the corresponding left-run element via the linear phase. Fixed to `gallop_left` (≥ *left).
    Reference: Python's timsort (CPython listobject.c merge_hi) uses gallop_right/gallop_left in exactly this order.
    Root cause confirmed via minimal failing case: n=201, keys 0-4, seed 0 — mismatch at key=0 position 8.

### Completion Notes List

- Created `libs/sort/test/test_timsort.cpp` following `test_spinsort.cpp` pattern: Boost license header,
  five function declarations (two implemented, three stubbed for Story 2.2), `test_main` entry point.
- Implemented `test_correctness()`: six data shapes (random, sorted, reverse, duplicates, single-run, two-overload).
- Implemented `test_stability()`: 10000 key-index pairs, shuffled with mt19937(0), compared element-by-element
  against `std::stable_sort` oracle using struct comparator `CmpFirst`.
- Registered `test_timsort` in both `libs/sort/test/CMakeLists.txt` and `libs/sort/test/Jamfile.v2`.
- Fixed two stability bugs in `libs/sort/include/boost/sort/timsort/timsort.hpp` `merge_hi` gallop phase
  (first gallop: `gallop_left` → `gallop_right`; second gallop: `gallop_right` → `gallop_left`).
- All tests pass: zero warnings from our code, binary exit 0, ASAN clean, UBSAN clean.

### File List

- `libs/sort/test/test_timsort.cpp` (CREATED)
- `libs/sort/test/CMakeLists.txt` (MODIFIED — added `boost_sort_add_test(test_timsort test_timsort.cpp)`)
- `libs/sort/test/Jamfile.v2` (MODIFIED — added `[ run test_timsort.cpp ... : test_timsort ]` entry)
- `libs/sort/include/boost/sort/timsort/timsort.hpp` (MODIFIED — fixed two gallop stability bugs in `merge_hi`)

### Review Findings

- [x] [Review][Patch] Copyright year in file header is "2024" — should be "2026" [test/test_timsort.cpp:4]
- [x] [Review][Defer] `v5` in `test_correctness` is constructed identically to `v2` [test/test_timsort.cpp:57-61] — deferred, matches spec reference implementation; spec-level concern
- [x] [Review][Defer] `merge_hi` gallop fix in timsort.hpp is not exercised by Story 2.1 tests [include/boost/sort/timsort/timsort.hpp:344-374] — deferred, Story 2.2 `test_adversarial()` explicitly covers this path

### Change Log

- 2026-06-18: Story 2.1 created — Correctness and Stability Tests.
- 2026-06-18: Story 2.1 implemented — test file created, build system registered, two merge_hi stability bugs fixed, all tests pass with ASAN/UBSAN clean.
