---
baseline_commit: 48bf72e9fd90297bbcdebb6e13152fd2f02be158
---

# Story 1.1: Header File Skeleton, Foundational Types, and `compute_minrun`

Status: done

## Story

As a C++ developer contributing to boost::sort,
I want the `timsort.hpp` file created with its full structural skeleton, foundational data structures (`SortState`, `RunEntry`, `RunStack`, iterator aliases), and the `compute_minrun` function,
so that all subsequent implementation stories have a compilable, correctly-structured base to build upon.

## Acceptance Criteria

1. **File structure compiles:** `libs/sort/include/boost/sort/timsort/timsort.hpp` exists with `#pragma once`, exactly 6 stdlib includes (`<algorithm>`, `<cstddef>`, `<functional>`, `<iterator>`, `<utility>`, `<vector>`), and namespace structure `boost::sort::tim_detail`. Compiles standalone with `-std=c++11 -Wall -Wextra -Wpedantic` with zero warnings. No Boost headers present.

2. **Iterator aliases correct:** `iter_value_t<Iter>` and `iter_diff_t<Iter>` alias templates defined in `tim_detail` using `std::iterator_traits`. `Iter::value_type` is never used directly anywhere in the file.

3. **SortState correct:** `template <typename V> struct SortState` with `std::vector<V> buffer` and `int min_gallop`. Default-constructed `SortState<int>` has `min_gallop == 7` and `buffer.empty() == true`.

4. **RunEntry and RunStack correct:** `RunEntry` struct with fields named exactly `base` (type `Iter`) and `len` (type `iter_diff_t<Iter>`). `RunStack` defined as `std::vector<RunEntry>` (or equivalent typedef). Field names must be exactly `base` and `len` — no synonyms.

5. **compute_minrun correct:** `tim_detail::compute_minrun(n)` using canonical Python-derived formula (shift right until n < 64, OR-ing in any shifted-off bits). Verified: n∈[1,63] → returns n unchanged; n=64 → returns 32; n=65 → returns 33; all n≥64 → result in [32,64] inclusive.

## Tasks / Subtasks

- [x] Task 1: Create directory and file skeleton (AC: 1)
  - [x] Create directory `libs/sort/include/boost/sort/timsort/`
  - [x] Create `timsort.hpp` with `#pragma once` guard (NOT `#ifndef`)
  - [x] Add license/file comment block (Boost Software License 1.0 header; algorithm name "timsort"; stability "stable"; complexity "O(N) best / O(N log N) average and worst"; memory "O(N), merge buffer ≤ N/2 elements"; origin "Tim Peters, 2002"; refactor note `// Refactor to detail/ subdirectory if this file exceeds ~900 lines`)
  - [x] Add exactly 6 stdlib includes in alphabetical order: `<algorithm>`, `<cstddef>`, `<functional>`, `<iterator>`, `<utility>`, `<vector>`
  - [x] Add namespace skeleton: `namespace boost { namespace sort { namespace tim_detail { } } }`
  - [x] Compile standalone and confirm zero warnings under `-std=c++11 -Wall -Wextra -Wpedantic`

- [x] Task 2: Add iterator_traits aliases (AC: 2)
  - [x] Define `iter_value_t<Iter>` as `using` alias at namespace scope inside `tim_detail`
  - [x] Define `iter_diff_t<Iter>` as `using` alias at namespace scope inside `tim_detail`
  - [x] Verify no direct `Iter::value_type` usage exists anywhere in the file
  - [x] Recompile; confirm zero warnings

- [x] Task 3: Implement SortState struct (AC: 3)
  - [x] Define `template <typename V> struct SortState` inside `tim_detail`
  - [x] Add member `std::vector<V> buffer`
  - [x] Add member `int min_gallop` initialized to 7 in a constructor or default member initializer
  - [x] Ensure default-constructed `SortState<int>` has `min_gallop == 7` and `buffer.empty() == true`
  - [x] Recompile; confirm zero warnings

- [x] Task 4: Add RunEntry struct and RunStack typedef (AC: 4)
  - [x] Define `RunEntry` as a template struct (templated on `Iter`) with fields `base` and `len` — exact names, no synonyms
  - [x] `base` field type: `Iter`
  - [x] `len` field type: `iter_diff_t<Iter>`
  - [x] Define `RunStack` as `typedef std::vector<RunEntry<Iter>> RunStack;` or equivalent (P4: use typedef in function bodies, using at namespace scope — RunStack is at namespace scope so use `using` or ensure it is accessible as needed)
  - [x] Recompile; confirm zero warnings

- [x] Task 5: Implement compute_minrun (AC: 5)
  - [x] Define `tim_detail::compute_minrun(iter_diff_t<Iter> n) -> iter_diff_t<Iter>` (or a standalone template function with the same signature)
  - [x] Implement canonical Python-derived formula: shift n right until < 64, accumulate any shifted-off bits into a remainder flag; return shifted n + (remainder ? 1 : 0)
  - [x] Recompile; confirm zero warnings

- [x] Task 6: Write and run verification assertions for compute_minrun (AC: 5)
  - [x] Write a `static_assert` or runtime assertion block (can be in a `#if 0`-guarded block or a small temporary main) verifying: `compute_minrun(1)==1`, `compute_minrun(63)==63`, `compute_minrun(64)==32`, `compute_minrun(65)==33`, and all n in [1,256] produce result in [1,64]
  - [x] Run the verification; confirm all assertions hold
  - [x] Remove any temporary test main before marking complete — the assertions are for development validation only; the Boost.Test file is created in Story 2.1

- [x] Task 7: Final compile validation (AC: 1, 2, 3, 4, 5)
  - [x] Compile with `-std=c++11 -Wall -Wextra -Wpedantic` — zero warnings
  - [x] Confirm `grep "#include" libs/sort/include/boost/sort/timsort/timsort.hpp` shows exactly 6 lines, all stdlib headers
  - [x] Confirm `grep -r "using namespace" libs/sort/include/boost/sort/timsort/` returns zero matches
  - [x] Confirm `grep "Iter::value_type" libs/sort/include/boost/sort/timsort/timsort.hpp` returns zero matches (comment line only — no code-level usage)

## Dev Notes

### Critical Architecture Requirements (MUST follow exactly)

**File guard:** Use `#pragma once` (NOT `#ifndef/#define`). This is explicitly specified in D3.1 and the architecture. Spinsort uses `#ifndef` — do NOT copy that pattern for timsort.

**Namespace structure (P2 — mandatory layout):**
```cpp
namespace boost {
namespace sort {
namespace tim_detail {
  // 1. iter_value_t / iter_diff_t aliases
  // 2. compute_minrun
  // (stories 1.2–1.3 add more here)
} // namespace tim_detail

// Public API goes here (outside tim_detail, inside boost::sort)
// (stories 1.4 adds the public timsort() overloads)
} // namespace sort
} // namespace boost
```

**No `using namespace` anywhere in the header** (P2). Not even `using namespace std`. This is an anti-pattern for headers.

**Iterator aliases (D3.3 — enforced throughout ALL stories):**
```cpp
// At namespace scope inside tim_detail — use `using` alias (P4):
template <typename Iter>
using iter_value_t = typename std::iterator_traits<Iter>::value_type;
template <typename Iter>
using iter_diff_t  = typename std::iterator_traits<Iter>::difference_type;
```
Every iterator-derived type in ALL stories must use these aliases. `Iter::value_type` is explicitly forbidden.

**SortState (D1.1):**
```cpp
template <typename ValueType>
struct SortState {
    std::vector<ValueType> buffer;  // merge buffer, capacity <= N/2
    int min_gallop;                  // adaptive gallop threshold, starts at 7
    SortState() : min_gallop(7) {}
};
```
- `buffer` and `min_gallop` are the EXACT member names — no synonyms
- Instantiated ONCE on the stack in the top-level timsort() call (Story 1.4); passed by reference to all merge functions
- `buffer.reserve(n/2)` at sort start; `buffer.clear()` between merges; NEVER grow after initial reserve
- **NEVER use `buffer.resize(n)`** — this breaks non-default-constructible types (critical anti-pattern)

**RunEntry and RunStack:**
```cpp
template <typename Iter>
struct RunEntry {
    Iter base;
    iter_diff_t<Iter> len;
};
// RunStack is used as: std::vector<RunEntry<Iter>> in function scope
```
Field names are `base` and `len` — exact, no synonyms. These names are used by merge_collapse and merge_force_collapse in Stories 1.3+.

**compute_minrun formula (canonical Python-derived):**
```cpp
template <typename Iter>
iter_diff_t<Iter> compute_minrun(iter_diff_t<Iter> n) {
    iter_diff_t<Iter> r = 0;  // flag: any bits shifted off?
    while (n >= 64) {
        r |= (n & 1);
        n >>= 1;
    }
    return n + r;
}
```
Result is always in [32, 64] for any n ≥ 64. For n < 64, returns n unchanged.

**Include order (architecture):** own header first → Boost headers → stdlib headers (alphabetically within groups). Since timsort.hpp is self-contained with no Boost headers, the 6 stdlib headers appear alphabetically.

**typedef vs using (P4):**
- At namespace scope: use `using` aliases (e.g., `iter_value_t`, `iter_diff_t`)
- In function bodies: use `typedef` (e.g., `typedef typename std::iterator_traits<Iter>::value_type value_type;`)
- Do NOT use `using value_type = ...` in function bodies — may warn on some C++11 compilers

**std::move_if_noexcept (D3.4):** `<utility>` is included for this — it's used in Stories 1.2–1.3 for merge buffer operations. Include it here in the skeleton so subsequent stories don't need to modify the include list.

### File to Create

| File | Action | Path |
|------|--------|------|
| `timsort.hpp` | CREATE | `libs/sort/include/boost/sort/timsort/timsort.hpp` |

No other files are modified in this story. `sort.hpp` update is Story 1.4. Test file is Epic 2.

### Build System Warning — No Wildcards (CRITICAL)

⚠️ **Architecture assumption was wrong.** The architecture doc says "test_timsort.cpp picked up by existing wildcard build rules." The actual `test/Jamfile.v2` and `test/CMakeLists.txt` use EXPLICIT file listings:

- `Jamfile.v2`: each test is individually listed with `[ run test_spinsort.cpp ... ]`
- `CMakeLists.txt`: each test is individually registered with `boost_sort_add_test(test_spinsort test_spinsort.cpp)`

**Story 2.1 (test file creation) must add explicit entries to both build files.** This is not in scope for story 1.1, but the developer must know this constraint.

For verification in this story (Task 6), use a standalone compilation or assertion-based test — NOT the Jamfile/CMake build. The Boost.Test harness setup belongs to Story 2.1.

### Header Comment Block Template (FR-18)

The file comment block for timsort.hpp must contain:

```
//----------------------------------------------------------------------------
/// @file timsort.hpp
/// @brief Timsort — adaptive stable sort for random-access iterator ranges
///
/// Algorithm:   timsort
/// Stability:   stable
/// Complexity:  O(N) best / O(N log N) average and worst
/// Memory:      O(N), merge buffer <= N/2 elements
/// Origin:      Tim Peters, 2002 (Python's sort algorithm)
///
/// @author      [your name]
/// Copyright (c) [year]
/// Distributed under the Boost Software License, Version 1.0.
///     (See accompanying file LICENSE_1_0.txt or copy at
///      http://www.boost.org/LICENSE_1_0.txt)
///
/// @remarks
/// Precondition: comparator must not throw. If comp throws during a merge,
/// the range is left in a valid but unspecified state.
///
/// Refactor to detail/ subdirectory if this file exceeds ~900 lines.
//----------------------------------------------------------------------------
```

### Analogous Reference File

`libs/sort/include/boost/sort/spinsort/spinsort.hpp` — but note key differences from timsort:
- Spinsort uses `#ifndef` guard; timsort MUST use `#pragma once`
- Spinsort includes Boost headers (`boost/sort/insert_sort/insert_sort.hpp`, etc.); timsort is stdlib-only
- Spinsort uses `namespace spin_detail`; timsort uses `namespace tim_detail`
- Spinsort has a `range<>` helper from `boost/sort/common/`; timsort uses raw iterators only

### Compilation Test Command

```bash
# From the boost superproject root:
g++ -std=c++11 -Wall -Wextra -Wpedantic \
    -I libs/sort/include \
    -x c++ - <<'EOF'
#include <boost/sort/timsort/timsort.hpp>
int main() { return 0; }
EOF
```

Or via CMake if available:
```bash
cd build && cmake .. -DCMAKE_CXX_STANDARD=11 && cmake --build . --target boost_sort_...
```

### Project Structure Notes

- All changes confined to `libs/sort/` (git submodule at `libs/sort`)
- New directory: `libs/sort/include/boost/sort/timsort/`
- Pattern: follows `pdqsort` and `spinsort` subdirectory layout exactly
- Google C++ Style Guide: 2-space indent, snake_case for functions, PascalCase for types, 80-char line limit

### References

- [Source: _bmad-output/planning-artifacts/architecture.md#D1.1] — SortState struct design
- [Source: _bmad-output/planning-artifacts/architecture.md#D3.1] — tim_detail namespace
- [Source: _bmad-output/planning-artifacts/architecture.md#D3.2] — Dependency policy (stdlib-only)
- [Source: _bmad-output/planning-artifacts/architecture.md#D3.3] — iterator_traits aliases
- [Source: _bmad-output/planning-artifacts/architecture.md#P1] — Canonical function names
- [Source: _bmad-output/planning-artifacts/architecture.md#P2] — Namespace & file layout
- [Source: _bmad-output/planning-artifacts/architecture.md#P4] — typedef vs using
- [Source: _bmad-output/planning-artifacts/epics.md#Story-1.1] — Acceptance criteria
- [Source: libs/sort/test/Jamfile.v2] — Explicit (not wildcard) test registration
- [Source: libs/sort/test/CMakeLists.txt] — Explicit (not wildcard) test registration

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

- Created `libs/sort/include/boost/sort/timsort/timsort.hpp` with `#pragma once`, 6 stdlib includes, full license/doc comment block.
- `iter_value_t<Iter>` and `iter_diff_t<Iter>` aliases defined via `using` at namespace scope in `tim_detail` — no `Iter::value_type` in code.
- `SortState<V>` struct: `buffer` (empty) + `min_gallop` (7) on default construction. Verified via runtime assertion.
- `RunEntry<Iter>` struct with exact field names `base` (Iter) and `len` (iter_diff_t<Iter>). Verified field access.
- `compute_minrun` canonical formula verified: n∈[1,63]→n; n=64→32; n=65→33; all n∈[64,1024]→[32,64]. 100% pass.
- All 7 tasks complete. Zero warnings under `-std=c++11 -Wall -Wextra -Wpedantic`. Zero `using namespace`. Zero code-level `Iter::value_type`.

### File List

- `libs/sort/include/boost/sort/timsort/timsort.hpp` (CREATE)

### Change Log

- 2026-06-17: Story 1.1 implemented — timsort.hpp skeleton, iter aliases, SortState, RunEntry, compute_minrun. All ACs verified.
