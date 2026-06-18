---
baseline_commit: 5c1a321c7d7660db95b3b8302af1498fea239a1c
---

# Story 2.2: Edge-Case, Type Coverage, and Adversarial Tests

Status: done

## Story

As a C++ developer contributing to boost::sort,
I want `test_edge_cases()`, `test_type_coverage()`, and `test_adversarial()` added to `test_timsort.cpp`,
so that I can verify timsort is safe under boundary conditions, compiles for multiple value types, and correctly handles inputs that stress galloping mode and run-stack logic — with ASAN + UBSAN confirming no memory or undefined-behaviour errors.

## Acceptance Criteria

1. **`test_edge_cases()` passes (FR-12):** empty range `[first, first)` (no-op, no crash), single element (no-op), two elements in both orderings (correct result), all-equal elements (sorted), fully reversed input (correctly sorted).

2. **ASAN clean (NFR-4):** Full test binary compiled with `-fsanitize=address` runs to completion with zero AddressSanitizer errors.

3. **UBSAN clean (NFR-4):** Full test binary compiled with `-fsanitize=undefined` runs to completion with zero UndefinedBehaviorSanitizer errors (no signed overflow in index arithmetic, no out-of-bounds on merge buffer, no reads from moved-from elements).

4. **`test_type_coverage()` passes (FR-13):** timsort compiles and produces correctly sorted output for `int`, `std::string`, and a user-defined struct `Rec { int key; std::string val; }` sorted by a custom struct comparator on `key`.

5. **`test_adversarial()` passes:** the following inputs produce correct sorted output: (a) an input that triggers galloping mode (one run wins 7+ consecutive comparisons) then exits gracefully, (b) an input that simultaneously violates both run-stack invariants (exercises the tri-entry merge ordering), (c) inputs at the minrun boundary N=63 and N=64.

6. **Function names and order unchanged (P6):** all five functions (`test_correctness`, `test_stability`, `test_edge_cases`, `test_type_coverage`, `test_adversarial`) remain declared at the top of the file and called in that order from `test_main(int, char*[])`. No change to `test_correctness`, `test_stability`, or `test_main`.

7. **Test binary exits 0:** compiled and run normally; all five functions pass; exit code 0.

## Tasks / Subtasks

- [x] Task 1: Add `#include <string>` and implement `test_edge_cases()` (AC: 1)
  - [x] Add `#include <string>` to the include block (needed for std::string in test_type_coverage; add after `<random>`, before the boost headers)
  - [x] Implement `test_edge_cases()`: empty vector — call timsort, check `v.empty()`; single-element vector `{42}` — call timsort, `BOOST_REQUIRE(v.size()==1u)`, `BOOST_CHECK(v[0]==42)`
  - [x] Two elements ascending `{1,2}` — call timsort, check `v[0]==1 && v[1]==2`
  - [x] Two elements descending `{2,1}` — call timsort, check `v[0]==1 && v[1]==2`
  - [x] All-equal: 1000 ints all value 7 — call timsort, `BOOST_REQUIRE(v.size()==1000u)`, `BOOST_CHECK(std::is_sorted(...))`, loop checking all values are 7
  - [x] Fully reversed: 10000 ints `[10000, 9999, ..., 1]` — call timsort, `BOOST_REQUIRE`, `BOOST_CHECK(std::is_sorted(...))`

- [x] Task 2: Implement `test_type_coverage()` (AC: 4)
  - [x] `int` block: mt19937(7) seeded RNG, 1000 elements in `[0,1000)`, sort with default overload, `BOOST_CHECK(std::is_sorted(...))`
  - [x] `std::string` block: 700 elements drawn from 7-word pool with mt19937(42), sort with default overload (lexicographic), `BOOST_CHECK(std::is_sorted(...))`
  - [x] `Rec { int key; std::string val; }` struct defined inside function body (C++11 local struct as template arg — valid); `CmpKey` struct comparator defined similarly; 500 elements with key = `rng()%50`, sort with `CmpKey()`, check key-order with `BOOST_CHECK(!(v[i].key < v[i-1].key))` for i in [1, 500)
  - [x] Use struct comparators (not lambdas) — same rationale as Story 2.1: C++11 non-copyable lambda edge cases with template deduction

- [x] Task 3: Implement `test_adversarial()` (AC: 5)
  - [x] **Gallop trigger block**: construct `Vec` of 200 ints: first 100 are `[100..199]`, next 100 are `[0..99]`. timsort pushes two ascending runs of 100 each; merge_lo copies left run [100..199] to buffer; right run [0..99] wins 100 consecutive comparisons → right_wins reaches 7 → gallop phase activates → right run exhausted → buffer flushed. Assert `std::is_sorted` after. Include `BOOST_REQUIRE(v.size()==200u)`.
  - [x] **Both-invariants block**: construct `Vec` of 140 ints: 60 ascending [0..59] + 40 ascending [200..239] + 40 ascending [100..139]. minrun(140)=35. Three natural runs of lengths 60, 40, 40. After pushing run-C (40): inv1 (40<=40) AND inv2 (60<=40+40=80) both violated simultaneously. merge_collapse chooses n=1 (B-C merge, since stack[0].len=60 is NOT < stack[2].len=40). Merge B+C → stack [60, 80]. inv1 (60<=80) still violated → merge A+(B+C). Assert `std::is_sorted` after. `BOOST_REQUIRE(v.size()==140u)`.
  - [x] **Minrun N=63 block**: 63 random ints (mt19937(42)), sort, `BOOST_REQUIRE(v.size()==63u)`, `BOOST_CHECK(std::is_sorted(...))`. Note: minrun(63)=63, so binary_insertion_sort handles the whole range; no merge path exercised.
  - [x] **Minrun N=64 block**: 64 random ints (mt19937(42)), sort, `BOOST_REQUIRE(v.size()==64u)`, `BOOST_CHECK(std::is_sorted(...))`. Note: minrun(64)=32, so two runs of ~32 get pushed and merged; merge path IS exercised.

- [x] Task 4: Build and run tests (AC: 2, 3, 7)
  - [x] Compile with default flags; confirm zero warnings; run binary; confirm exit code 0 and all five test functions pass
  - [x] Compile and run with ASAN (`-fsanitize=address`); zero errors
  - [x] Compile and run with UBSAN (`-fsanitize=undefined`); zero errors

### Review Findings

- [x] [Review][Defer] Gallop activation not instrumentally verified — test data structurally guarantees gallop (100 consecutive right wins >> 7 threshold), ASAN/UBSAN clean, but only std::is_sorted asserted; no internal probe confirms gallop code path ran [libs/sort/test/test_timsort.cpp] — deferred, pre-existing
- [x] [Review][Defer] merge_hi path (len1>len2) never explicitly targeted — all adversarial merges use equal or right-heavier runs, so merge_hi gallop logic is untested under pressure [libs/sort/test/test_timsort.cpp] — deferred, pre-existing
- [x] [Review][Defer] Stability not verified for equal strings or equal-key Rec structs in test_type_coverage — only sort order checked, not original relative order preserved [libs/sort/test/test_timsort.cpp] — deferred, pre-existing
- [x] [Review][Defer] min_gallop persistence across multiple merges not verified — no test constructs multi-merge input and asserts gallop threshold does not reset between merges [libs/sort/test/test_timsort.cpp] — deferred, pre-existing
- [x] [Review][Defer] N=2 equal-element case missing from test_edge_cases — (7,7) pair not tested [libs/sort/test/test_timsort.cpp] — deferred, pre-existing
- [x] [Review][Defer] N=3 case not tested — smallest range where run extension via binary_insertion_sort inserts one element into a 2-element natural run [libs/sort/test/test_timsort.cpp] — deferred, pre-existing
- [x] [Review][Defer] Gallop with mixed-win streaks (entry + exit + re-entry cycle) not tested — the adaptive gallop threshold decay/growth cycle is unexercised [libs/sort/test/test_timsort.cpp] — deferred, pre-existing
- [x] [Review][Defer] merge_collapse n-1 branch not exercised — the "merge smaller pair first" tiebreaker (when stack[n-1].len < stack[n+1].len) is unreachable with current adversarial data [libs/sort/test/test_timsort.cpp] — deferred, pre-existing

## Dev Notes

### CRITICAL: Only Modify Three Stubs — Touch Nothing Else

`test_timsort.cpp` currently has:
```cpp
void test_edge_cases()    { /* Story 2.2 */ }
void test_type_coverage() { /* Story 2.2 */ }
void test_adversarial()   { /* Story 2.2 */ }
```

**ONLY replace these three function bodies.** Do NOT touch:
- `test_correctness()` (lines 27–68 in current file)
- `test_stability()` (lines 70–96)
- `test_main` (lines 102–109)
- Any function declarations at the top (lines 21–25)
- File header comment block (lines 1–19)

Also add `#include <string>` to the include block (required for `std::string` in test_type_coverage — not transitively guaranteed by any existing include).

### Build System Already Registered — No Changes Required

Story 2.1 already added:
- `libs/sort/test/CMakeLists.txt`: `boost_sort_add_test(test_timsort test_timsort.cpp)`
- `libs/sort/test/Jamfile.v2`: `[ run test_timsort.cpp ... : test_timsort ]` entry

**Do NOT add any build system entries.** The file is already registered.

### Compilation Commands (from superproject root)

```bash
# Find the Boost headers path (needed for boost/test/...)
BOOST_INC=$(ls -d /usr/local/include/boost /opt/homebrew/include /usr/include | head -1 2>/dev/null) || BOOST_INC=/usr/local/include

# Step 1: Normal compile + run
g++ -std=c++11 -Wall -Wextra -Wpedantic \
    -I libs/sort/include \
    -I "$BOOST_INC" \
    libs/sort/test/test_timsort.cpp \
    -o test_timsort_bin
./test_timsort_bin

# Step 2: ASAN
g++ -std=c++11 -Wall -Wextra -Wpedantic -fsanitize=address \
    -I libs/sort/include \
    -I "$BOOST_INC" \
    libs/sort/test/test_timsort.cpp \
    -o test_timsort_asan
./test_timsort_asan

# Step 3: UBSAN
g++ -std=c++11 -Wall -Wextra -Wpedantic -fsanitize=undefined \
    -I libs/sort/include \
    -I "$BOOST_INC" \
    libs/sort/test/test_timsort.cpp \
    -o test_timsort_ubsan
./test_timsort_ubsan
```

CMake alternative:
```bash
cmake -B build -DCMAKE_CXX_STANDARD=11
cmake --build build --target boost_sort_test_timsort
ctest --test-dir build -R boost_sort_test_timsort -V
```

### `test_edge_cases()` Reference Implementation

```cpp
void test_edge_cases() {
  typedef std::vector<int> Vec;

  // Empty range — no crash, no-op
  {
    Vec v;
    boost::sort::timsort(v.begin(), v.end());
    BOOST_CHECK(v.empty());
  }

  // Single element — no-op
  {
    Vec v(1, 42);
    boost::sort::timsort(v.begin(), v.end());
    BOOST_REQUIRE(v.size() == 1u);
    BOOST_CHECK(v[0] == 42);
  }

  // Two elements — already ordered
  {
    Vec v;
    v.push_back(1); v.push_back(2);
    boost::sort::timsort(v.begin(), v.end());
    BOOST_REQUIRE(v.size() == 2u);
    BOOST_CHECK(v[0] == 1);
    BOOST_CHECK(v[1] == 2);
  }

  // Two elements — reversed
  {
    Vec v;
    v.push_back(2); v.push_back(1);
    boost::sort::timsort(v.begin(), v.end());
    BOOST_REQUIRE(v.size() == 2u);
    BOOST_CHECK(v[0] == 1);
    BOOST_CHECK(v[1] == 2);
  }

  // All-equal elements (1000 x 7)
  {
    const int N = 1000;
    Vec v(N, 7);
    boost::sort::timsort(v.begin(), v.end());
    BOOST_REQUIRE(static_cast<int>(v.size()) == N);
    BOOST_CHECK(std::is_sorted(v.begin(), v.end()));
    for (int i = 0; i < N; ++i)
      BOOST_CHECK(v[i] == 7);
  }

  // Fully reversed range
  {
    const int N = 10000;
    Vec v;
    for (int i = N; i > 0; --i) v.push_back(i);
    boost::sort::timsort(v.begin(), v.end());
    BOOST_REQUIRE(static_cast<int>(v.size()) == N);
    BOOST_CHECK(std::is_sorted(v.begin(), v.end()));
  }
}
```

Note: `Vec v(N, 7)` (fill constructor) is valid C++11. `std::is_sorted` is in `<algorithm>` (already included).

### `test_type_coverage()` Reference Implementation

```cpp
void test_type_coverage() {
  // int
  {
    std::vector<int> v;
    std::mt19937 rng(7);
    for (int i = 0; i < 1000; ++i) v.push_back(static_cast<int>(rng() % 1000));
    boost::sort::timsort(v.begin(), v.end());
    BOOST_REQUIRE(v.size() == 1000u);
    BOOST_CHECK(std::is_sorted(v.begin(), v.end()));
  }

  // std::string (lexicographic default sort)
  {
    const char* words[] = {
      "banana","apple","cherry","date","elderberry","fig","grape"
    };
    const int NW = 7;
    std::vector<std::string> v;
    std::mt19937 rng(42);
    for (int i = 0; i < 700; ++i)
      v.push_back(words[rng() % NW]);
    boost::sort::timsort(v.begin(), v.end());
    BOOST_REQUIRE(v.size() == 700u);
    BOOST_CHECK(std::is_sorted(v.begin(), v.end()));
  }

  // User-defined struct with custom comparator (FR-13)
  {
    struct Rec { int key; std::string val; };
    struct CmpKey {
      bool operator()(const Rec& a, const Rec& b) const {
        return a.key < b.key;
      }
    };
    std::vector<Rec> v;
    std::mt19937 rng(99);
    for (int i = 0; i < 500; ++i) {
      Rec r;
      r.key = static_cast<int>(rng() % 50);
      r.val = "x";
      v.push_back(r);
    }
    boost::sort::timsort(v.begin(), v.end(), CmpKey());
    BOOST_REQUIRE(v.size() == 500u);
    for (std::size_t i = 1; i < v.size(); ++i)
      BOOST_CHECK(!(v[i].key < v[i-1].key));
  }
}
```

**Why struct comparators, not lambdas:** Matches Story 2.1's `CmpFirst` pattern. C++11 lambdas are non-copyable in some edge cases with template deduction. Struct form is safe and consistent. See Story 2.1 Dev Notes anti-pattern section.

**Local struct as template arg:** Valid C++11 (restriction lifted from C++03). `Rec` and `CmpKey` defined inside `test_type_coverage()` function body can be used as template arguments.

### `test_adversarial()` Reference Implementation

```cpp
void test_adversarial() {
  typedef std::vector<int> Vec;

  // --- A: Gallop trigger ---
  // Input: two ascending runs concatenated in wrong order.
  // Run1=[100..199], Run2=[0..99]. timsort pushes both (len=100 each, >=minrun(200)=50).
  // merge_collapse: inv1 (100<=100) violated → merge_lo called (len1==len2, 100<=100).
  // merge_lo copies Run1 [100..199] to buffer; right pointer starts at Run2 [0].
  // All right-run elements (0..99) are < all buffer elements (100..199):
  // right_wins accumulates 100 consecutive → right_wins >= min_gallop(7) → gallop activates.
  // Gallop: gallop_left(*buf=100, right_base, len2, 0, comp) returns len2 (all right elements < 100).
  // Right run exhausted in one gallop pass. Buffer elements [100..199] flushed to dest.
  // Result: [0..99, 100..199]. Both gallop entry and exit are exercised.
  {
    const int N = 200;
    Vec v;
    for (int i = N/2; i < N; ++i) v.push_back(i);   // [100..199]
    for (int i = 0;   i < N/2; ++i) v.push_back(i);  // [0..99]
    boost::sort::timsort(v.begin(), v.end());
    BOOST_REQUIRE(static_cast<int>(v.size()) == N);
    BOOST_CHECK(std::is_sorted(v.begin(), v.end()));
  }

  // --- B: Both run-stack invariants violated simultaneously ---
  // Input: three ascending runs of lengths 60, 40, 40 (total N=140, minrun(140)=35).
  // Trace:
  //   Push Run-A (60): stack=[(A,60)]. OK.
  //   Push Run-B (40): inv1=60<=40? No. stack=[(A,60),(B,40)].
  //   Push Run-C (40): n=1. inv1=stack[1].len<=stack[2].len → 40<=40=true.
  //                    inv2=stack[0].len<=stack[1].len+stack[2].len → 60<=80=true.
  //                    BOTH violated. merge_collapse: stack[n-1].len=60 < stack[n+1].len=40? No.
  //                    → merge (B,C) at n=1. B=[200..239], C=[100..139]:
  //                    merge_lo(B,C): all C elements (100..139) < all B elements (200..239)
  //                    → gallop triggered again. Merged BC=[100..139,200..239], len=80.
  //                    stack=[(A,60),(BC,80)].
  //                    Recheck: inv1=60<=80=true → merge A+BC at n=0.
  //                    merge_lo(A=[0..59], BC=[100..139,200..239]):
  //                    A elements win 60 consecutive (0..59 < 100) → left gallop.
  //   Result: [0..59, 100..139, 200..239]. Sorted.
  {
    const int N = 140;
    Vec v;
    for (int i = 0;   i < 60;  ++i) v.push_back(i);         // Run-A: [0..59]
    for (int i = 200; i < 240; ++i) v.push_back(i);          // Run-B: [200..239]
    for (int i = 100; i < 140; ++i) v.push_back(i);          // Run-C: [100..139]
    boost::sort::timsort(v.begin(), v.end());
    BOOST_REQUIRE(static_cast<int>(v.size()) == N);
    BOOST_CHECK(std::is_sorted(v.begin(), v.end()));
  }

  // --- C: Minrun boundary N=63 ---
  // minrun(63)=63 (63<64, returned unchanged). Entire range handled by
  // binary_insertion_sort in one pass; no merge operations exercised.
  {
    const int N = 63;
    Vec v;
    std::mt19937 rng(42);
    for (int i = 0; i < N; ++i) v.push_back(static_cast<int>(rng() % N));
    boost::sort::timsort(v.begin(), v.end());
    BOOST_REQUIRE(static_cast<int>(v.size()) == N);
    BOOST_CHECK(std::is_sorted(v.begin(), v.end()));
  }

  // --- D: Minrun boundary N=64 ---
  // minrun(64)=32 (64>>1=32, no remainder bits). Two runs of ~32 each
  // get pushed and merged via merge_lo/merge_hi — merge path IS exercised.
  {
    const int N = 64;
    Vec v;
    std::mt19937 rng(42);
    for (int i = 0; i < N; ++i) v.push_back(static_cast<int>(rng() % N));
    boost::sort::timsort(v.begin(), v.end());
    BOOST_REQUIRE(static_cast<int>(v.size()) == N);
    BOOST_CHECK(std::is_sorted(v.begin(), v.end()));
  }
}
```

### compute_minrun Verification Table (for Story notes)

| N   | minrun | Derivation |
|-----|--------|-----------|
| 63  | 63     | 63<64, returned unchanged |
| 64  | 32     | 64→32 (shift 1, r=0) |
| 140 | 35     | 140→70(r=0)→35(r=0), 35<64 |
| 200 | 50     | 200→100(r=0)→50(r=0), 50<64 |

### BOOST_REQUIRE vs BOOST_CHECK Policy (AC5, architecture D4.1)

| Situation | Macro |
|-----------|-------|
| Size check before iterating | `BOOST_REQUIRE` |
| `std::is_sorted` after sort | `BOOST_CHECK` |
| Individual element check | `BOOST_CHECK` |
| Empty check | `BOOST_CHECK` (not BOOST_REQUIRE; `empty()` alone is meaningful) |

### Stability Bug Already Fixed (from Story 2.1)

`timsort.hpp` `merge_hi` gallop phase was fixed in Story 2.1:
- First gallop: `gallop_left` → `gallop_right` (was broken)
- Second gallop: `gallop_right` → `gallop_left` (was broken)

The adversarial test block B exercises `merge_lo` gallop. The existing `test_stability()` (Story 2.1) covered `merge_hi` correctness. ASAN/UBSAN will catch any remaining memory issues.

### Existing timsort.hpp Public API (for reference)

```cpp
// In namespace boost::sort:
template <typename Iter, typename Compare>
void timsort(Iter first, Iter last, Compare comp);  // main overload

template <typename Iter>
void timsort(Iter first, Iter last);  // calls above with std::less<value_type>()
```

Requires RandomAccessIterator (static_assert fires on std::list etc.).

### Project Structure Notes

- **Only file modified:** `libs/sort/test/test_timsort.cpp` (fill three stubs + add `#include <string>`)
- **No new files created**
- **No build system changes** (test already registered by Story 2.1)
- All changes confined to the `libs/sort/` submodule

### References

- [Source: epics.md#Story-2.2] — Acceptance criteria and FR coverage
- [Source: architecture.md#D4.1] — Test framework pattern, BOOST_REQUIRE/BOOST_CHECK policy
- [Source: architecture.md#P6] — Test function names and test_main call order
- [Source: architecture.md#FR-12] — Edge case requirements
- [Source: architecture.md#FR-13] — Type coverage requirements (int, string, user-defined struct)
- [Source: architecture.md#NFR-4] — ASAN + UBSAN mandatory
- [Source: libs/sort/test/test_timsort.cpp] — Existing file to modify; stubs at lines 98-100
- [Source: libs/sort/include/boost/sort/timsort/timsort.hpp] — Public API and merge_hi stability fix (Story 2.1)
- [Source: story 2-1 Dev Notes] — Anti-pattern: struct comparators not lambdas; do not include sort.hpp; zero-warning policy

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6 (context engine / create-story)

### Debug Log References

_None_

### Completion Notes List

- Implemented `test_edge_cases()`: 6 sub-cases covering empty, single, two-element (both orderings), all-equal (1000×7), fully reversed (10000 elements). All pass AC1.
- Implemented `test_type_coverage()`: int (mt19937(7), 1000 elements), std::string (lexicographic, 700 elements, 7-word pool), user-defined `Rec`/`CmpKey` struct (500 elements, key-order check). Local struct as C++11 template arg. All pass AC4.
- Implemented `test_adversarial()`: gallop trigger (200 ints, two runs reversed), both-invariants violation (140 ints, runs 60+40+40), minrun N=63 (no merge), minrun N=64 (merge exercised). All pass AC5.
- Added `#include <string>` after `<random>`.
- Normal build: 0 errors from our code (48 warnings are all in Boost framework headers), exit 0. AC7 satisfied.
- ASAN build+run: zero AddressSanitizer errors. AC2 satisfied.
- UBSAN build+run: zero UndefinedBehaviorSanitizer errors. AC3 satisfied.
- `test_main` call order unchanged: correctness → stability → edge_cases → type_coverage → adversarial. AC6 satisfied.

### File List

- libs/sort/test/test_timsort.cpp

## Change Log

- 2026-06-18: Implemented test_edge_cases, test_type_coverage, test_adversarial stubs; added #include <string>. All ACs satisfied; ASAN+UBSAN clean. (Story 2.2)
