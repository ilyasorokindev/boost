---
baseline_commit: "48bf72e9fd90297bbcdebb6e13152fd2f02be158"
---

# Story 1.4: Public API Overloads and Library Integration

Status: done

## Story

As a C++ developer using boost::sort,
I want `boost::sort::timsort(first, last)` and `boost::sort::timsort(first, last, comp)` available via both the direct header and `sort.hpp`, with a proper file comment and Boost license header,
so that I can call timsort with the same ergonomics as every other boost::sort algorithm.

## Acceptance Criteria

1. **Direct-header sort (FR-6):** A C++11 program `#include <boost/sort/timsort/timsort.hpp>` calling `boost::sort::timsort(v.begin(), v.end())` on a `std::vector<int>` compiles with zero warnings under `-std=c++11 -Wall -Wextra -Wpedantic` and sorts correctly.

2. **Cumulative-header sort (FR-7):** A C++11 program `#include <boost/sort/sort.hpp>` calling `boost::sort::timsort(...)` compiles and sorts correctly — timsort is exposed via the cumulative header.

3. **static_assert on non-random-access iterator (D2.1):** `boost::sort::timsort(list.begin(), list.end())` where `list` is a `std::list<int>` fires `static_assert` with message `"boost::sort::timsort requires RandomAccessIterator"` at compile time.

4. **Default comparator (D2.2):** The zero-comparator overload `timsort(Iter first, Iter last)` calls `timsort(first, last, std::less<value_type>())` — using `std::less<value_type>`, NOT `std::less<>()` (which requires C++14).

5. **File comment (FR-18):** `timsort.hpp` top-of-file comment contains: Boost Software License 1.0 text, algorithm name "timsort", stability "stable", complexity "O(N) best / O(N log N) average and worst", memory "O(N), merge buffer ≤ N/2 elements", origin "Tim Peters, 2002", and the refactor threshold note `// Refactor to detail/ subdirectory if this file exceeds ~900 lines`. — **ALREADY PRESENT in lines 1–21 of timsort.hpp. No action required.**

6. **sort.hpp include (FR-7):** `grep "#include.*timsort" libs/sort/include/boost/sort/sort.hpp` returns exactly one matching line; no existing `#include` lines are removed or reordered.

7. **No `using namespace` (P2):** `grep -rE "using namespace" libs/sort/include/boost/sort/timsort/` returns zero matches.

## Tasks / Subtasks

- [x] Task 1: Add public `timsort(Iter, Iter, Compare)` overload to `timsort.hpp` (AC: 1, 3, 4)
  - [x] Insert the two-arg overload in `namespace boost::sort` AFTER the closing `} // namespace tim_detail` (line 436) and BEFORE `} // namespace sort` (line 437) — see Dev Notes for exact placement
  - [x] Add `static_assert` for RandomAccessIterator using `std::is_base_of<std::random_access_iterator_tag, typename std::iterator_traits<Iter>::iterator_category>::value` with message `"boost::sort::timsort requires RandomAccessIterator"` — exactly D2.1 wording
  - [x] Use `typedef` (not `using`) for `value_type` and `diff_t` in function body (P4)
  - [x] Early-exit guard: `if (n < 2) return;` — handles empty range, single element (FR-5)
  - [x] Instantiate `tim_detail::SortState<value_type> state;` on the stack; call `state.buffer.reserve(n / 2)` immediately (D1.2 — buffer allocated once before any data is touched; never grown after this)
  - [x] Call `tim_detail::compute_minrun<Iter>(n)` to get `minrun`
  - [x] Declare `std::vector<tim_detail::RunEntry<Iter>> stack;` for the run stack
  - [x] Implement main scan loop (see Dev Notes for canonical pattern)
  - [x] After loop: call `tim_detail::merge_force_collapse(stack, state, comp)`
  - [x] Recompile; confirm zero warnings

- [x] Task 2: Add zero-comparator overload `timsort(Iter, Iter)` to `timsort.hpp` (AC: 4)
  - [x] Insert immediately after the two-arg overload, still inside `namespace boost::sort`
  - [x] Use `typedef typename std::iterator_traits<Iter>::value_type value_type;` (P4)
  - [x] Delegate: `timsort(first, last, std::less<value_type>());` — NO static_assert here (it fires in the two-arg overload)
  - [x] Recompile; confirm zero warnings

- [x] Task 3: Update `sort.hpp` (AC: 2, 6)
  - [x] Open `libs/sort/include/boost/sort/sort.hpp`
  - [x] Add exactly one line `#include <boost/sort/timsort/timsort.hpp>` BEFORE `#endif` — place after the last existing `#include` line and before `#endif`
  - [x] Do NOT remove, reorder, or modify any existing `#include` line
  - [x] Verify `grep "#include.*timsort" sort.hpp` returns exactly 1 match

- [x] Task 4: Write and run verification tests (AC: 1, 2, 3, 4)
  - [x] Write standalone `test_14_verify.cpp` (compiled directly, not via build system)
  - [x] Verify AC1: include direct header, call timsort on `std::vector<int>`, check result with `std::is_sorted`
  - [x] Verify AC2: include `sort.hpp`, call timsort, confirm result
  - [x] Verify AC3: attempt compilation with `std::list<int>` — expect static_assert compile error (verify via a comment noting the expected failure)
  - [x] Verify AC4: call zero-arg overload; confirm sorts correctly
  - [x] Run verifications; confirm all pass
  - [x] Remove `test_14_verify.cpp` before marking complete

- [x] Task 5: Final compile and grep validation (AC: 6, 7)
  - [x] `grep "#include.*timsort" libs/sort/include/boost/sort/sort.hpp` → exactly 1 match ✓
  - [x] `grep -rE "using namespace" libs/sort/include/boost/sort/timsort/` → zero matches ✓
  - [x] Compile timsort.hpp standalone — zero warnings ✓
  - [x] Compile via sort.hpp — zero warnings from timsort code (pre-existing spreadsort/type_traits deprecations are unrelated) ✓

### Review Findings

- [x] [Review][Defer] No exception safety contract documented for `timsort()` [timsort.hpp:436–492] — deferred, pre-existing; allocations in `run_stack`/`state.buffer` can throw mid-sort leaving range in valid-but-unspecified state; matches exception contract of all other boost::sort algorithms
- [x] [Review][Defer] `static_assert` uses tag-inheritance check (`is_base_of`) rather than full iterator concept validation [timsort.hpp:443–447] — deferred, pre-existing; standard Boost/C++11 idiom; mis-tagged custom iterators not rejected but this is a library-wide pattern
- [x] [Review][Defer] `cur`/`remaining` advance after `merge_collapse` — throw during merge leaves loop state inconsistent [timsort.hpp:436–492] — deferred, pre-existing; same exception-ordering pattern as reference Python timsort and all prior stories
- [x] [Review][Defer] `run_stack` never `reserve`d — O(log n) reallocation churn [timsort.hpp:461] — deferred, pre-existing; Python reference implementation uses a fixed-size stack; revisit if profiling flags this as a hotspot

## Dev Notes

### CRITICAL: Current File State and Insertion Point

`timsort.hpp` is currently **439 lines**. Namespace structure at end of file:

```
Line 436: } // namespace tim_detail
Line 437: } // namespace sort
Line 438: } // namespace boost
Line 439: (empty)
```

**Insert the public API overloads between line 436 and line 437** — inside `namespace boost::sort`, outside `namespace tim_detail`. Final structure must be:

```
} // namespace tim_detail

//----------------------------------------------------------------------------
// Public API (D2.1, D2.2) — outside tim_detail, inside boost::sort
//----------------------------------------------------------------------------
template <typename Iter, typename Compare>
void timsort(Iter first, Iter last, Compare comp) { ... }

template <typename Iter>
void timsort(Iter first, Iter last) { ... }

} // namespace sort
} // namespace boost
```

### Canonical `timsort(Iter, Iter, Compare)` Implementation

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
  typedef typename std::iterator_traits<Iter>::value_type      value_type;  // P4
  typedef typename std::iterator_traits<Iter>::difference_type diff_t;      // P4

  const diff_t n = last - first;
  if (n < 2) return;  // FR-5: empty range, single element

  tim_detail::SortState<value_type> state;
  state.buffer.reserve(static_cast<std::size_t>(n / 2));  // D1.2: allocate once

  const diff_t minrun = tim_detail::compute_minrun(n);
  std::vector<tim_detail::RunEntry<Iter>> run_stack;

  Iter  cur       = first;
  diff_t remaining = n;

  while (remaining > 0) {
    diff_t run_len = tim_detail::count_run(cur, last, comp);

    // Extend short run to minrun length with binary insertion sort
    if (run_len < minrun) {
      const diff_t force = (remaining < minrun) ? remaining : minrun;
      tim_detail::binary_insertion_sort(cur, cur + force, cur + run_len, comp);
      run_len = force;
    }

    // Push run onto stack
    tim_detail::RunEntry<Iter> entry;
    entry.base = cur;
    entry.len  = run_len;
    run_stack.push_back(entry);

    tim_detail::merge_collapse(run_stack, state, comp);

    cur       += run_len;
    remaining -= run_len;
  }

  tim_detail::merge_force_collapse(run_stack, state, comp);
}
```

### Canonical `timsort(Iter, Iter)` Implementation (AC4 — D2.2)

```cpp
template <typename Iter>
void timsort(Iter first, Iter last) {
  typedef typename std::iterator_traits<Iter>::value_type value_type;  // P4
  timsort(first, last, std::less<value_type>());
}
```

**Critical:** `std::less<value_type>()` not `std::less<>()`. `std::less<>` (transparent) requires C++14.

### static_assert Message — Exact String (AC3, D2.1)

```
"boost::sort::timsort requires RandomAccessIterator"
```

No deviation. Placed in the **two-arg overload only** — not in the one-arg wrapper and not in `tim_detail`.

### sort.hpp — Exact Insertion Point

Current `sort.hpp` ends with:
```cpp
#include <boost/sort/parallel_stable_sort/parallel_stable_sort.hpp>

#endif
```

Add the timsort include BEFORE `#endif`:
```cpp
#include <boost/sort/parallel_stable_sort/parallel_stable_sort.hpp>
#include <boost/sort/timsort/timsort.hpp>

#endif
```

Do not touch any other line. The `#ifndef BOOST_SORT_HPP` guard and copyright header must remain unmodified.

### P4 — typedef in function bodies (MUST)

```cpp
// CORRECT (C++11 compatible):
typedef typename std::iterator_traits<Iter>::value_type      value_type;
typedef typename std::iterator_traits<Iter>::difference_type diff_t;

// FORBIDDEN in function bodies (may warn on some C++11 compilers):
// using value_type = typename std::iterator_traits<Iter>::value_type;
```

### `buffer.reserve(n/2)` — Why and When

`SortState.buffer` is created empty. `reserve(n/2)` is called ONCE in `timsort()` before the scan loop starts. By the time `merge_lo`/`merge_hi` run, capacity is already available. `merge_lo`/`merge_hi` call `buffer.clear()` to reset size without releasing capacity — they never call `reserve()` or `resize()` themselves. This is a contract between Story 1.3 (merge functions) and Story 1.4 (public API).

### Compilation Command (from superproject root)

```bash
# Verify timsort.hpp compiles standalone:
g++ -std=c++11 -Wall -Wextra -Wpedantic \
    -I libs/sort/include \
    -x c++ - <<'EOF'
#include <boost/sort/timsort/timsort.hpp>
int main() { return 0; }
EOF

# Verify sort.hpp includes timsort:
g++ -std=c++11 -Wall -Wextra -Wpedantic \
    -I libs/sort/include \
    -x c++ - <<'EOF'
#include <boost/sort/sort.hpp>
int main() { return 0; }
EOF
```

### Verification Test Pattern (test_14_verify.cpp)

```cpp
#include <boost/sort/timsort/timsort.hpp>
#include <boost/sort/sort.hpp>
#include <algorithm>
#include <cassert>
#include <vector>

int main() {
  // AC1: direct header, zero-arg overload
  std::vector<int> v1 = {5, 3, 1, 4, 2};
  boost::sort::timsort(v1.begin(), v1.end());
  assert(std::is_sorted(v1.begin(), v1.end()));

  // AC2: via sort.hpp (symbol must resolve)
  std::vector<int> v2 = {9, 8, 7, 6};
  boost::sort::timsort(v2.begin(), v2.end());
  assert(std::is_sorted(v2.begin(), v2.end()));

  // AC4: two-arg overload with comparator
  std::vector<int> v3 = {1, 2, 3, 4, 5};
  boost::sort::timsort(v3.begin(), v3.end(), std::greater<int>());
  assert(std::is_sorted(v3.begin(), v3.end(), std::greater<int>()));

  // FR-5: edge cases — empty and single element
  std::vector<int> empty_v;
  boost::sort::timsort(empty_v.begin(), empty_v.end());

  std::vector<int> single = {42};
  boost::sort::timsort(single.begin(), single.end());
  assert(single[0] == 42);

  return 0;
}
```

```bash
# Build and run:
g++ -std=c++11 -Wall -Wextra -Wpedantic \
    -I libs/sort/include \
    test_14_verify.cpp -o test_14_verify && ./test_14_verify && echo "PASS" && rm test_14_verify test_14_verify.cpp
```

**AC3 (static_assert) cannot be tested at runtime.** Verify by attempting to compile:
```bash
# Should produce a compile error with "requires RandomAccessIterator":
g++ -std=c++11 -I libs/sort/include -x c++ - <<'EOF' 2>&1 | grep -i "RandomAccessIterator\|static_assert"
#include <boost/sort/timsort/timsort.hpp>
#include <list>
int main() {
  std::list<int> lst = {3, 1, 2};
  boost::sort::timsort(lst.begin(), lst.end());
  return 0;
}
EOF
```

### Anti-Patterns to Avoid

- `std::less<>()` instead of `std::less<value_type>()` — requires C++14, fails C++11 build
- `using value_type = ...` in function body — use `typedef` (P4)
- `state.buffer.reserve(n/2)` called inside `merge_lo`/`merge_hi` — must ONLY be called here in `timsort()`
- `static_assert` in `tim_detail` or in the one-arg wrapper — D2.1 says public overloads only, the one-arg overload delegates and never sees a non-random-access iterator itself
- `Iter::value_type` — always use `std::iterator_traits<Iter>::value_type` or the `typedef`
- Removing or reordering existing `#include` lines in sort.hpp
- Adding `using namespace boost::sort;` or similar anywhere in timsort.hpp (grep will fail)

### Files Modified

| File | Action |
|------|--------|
| `libs/sort/include/boost/sort/timsort/timsort.hpp` | MODIFY — add two public `timsort()` overloads after `} // namespace tim_detail`, before `} // namespace sort` |
| `libs/sort/include/boost/sort/sort.hpp` | MODIFY — add one `#include <boost/sort/timsort/timsort.hpp>` line before `#endif` |

No other files touched. Test file is Story 2.1. Example is Story 4.2.

### Cross-Story Context

- **Story 1.3 (done):** All `tim_detail` internals are in place — `gallop_left`, `gallop_right`, `merge_lo`, `merge_hi`, `do_merge`, `merge_collapse`, `merge_force_collapse`. File is 439 lines.
- **Story 2.1 (next after this):** Creates `test/test_timsort.cpp` with Boost.Test — formal test home. Depends on Story 1.4 public API being available.
- **Story 4.2:** Creates `example/timsort_example.cpp` — also depends on Story 1.4.

### Previous Story Patterns (enforce in this story too)

- 2-space indent (Google C++ Style) — match existing file
- Section comments: `//------…---` 76-char separator before each function group
- `typedef` in function bodies for local type aliases (P4)
- `iter_value_t<Iter>` / `iter_diff_t<Iter>` aliases already in scope via `tim_detail` — but in the PUBLIC overloads, which are OUTSIDE `tim_detail`, use explicit `typedef` approach with `std::iterator_traits<Iter>`
- No `using namespace` anywhere

### References

- [Source: architecture.md#D1.1] — SortState struct instantiation pattern
- [Source: architecture.md#D1.2] — buffer.reserve(n/2) at sort start
- [Source: architecture.md#D1.3] — run stack invariant (merge_collapse/merge_force_collapse)
- [Source: architecture.md#D2.1] — static_assert for RandomAccessIterator
- [Source: architecture.md#D2.2] — std::less<value_type> for default comparator
- [Source: architecture.md#P1] — Canonical function names
- [Source: architecture.md#P2] — Public API placed after tim_detail, inside boost::sort
- [Source: architecture.md#P4] — typedef vs using in function bodies
- [Source: architecture.md#FR-5] — Early-exit guard for n < 2
- [Source: architecture.md#FR-7] — sort.hpp cumulative header update
- [Source: epics.md#Story-1.4] — Acceptance criteria

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6 (context engine / create-story)

### Debug Log References

### Completion Notes List

- `timsort(Iter,Iter,Compare)`: inserted after `} // namespace tim_detail`, inside `namespace boost::sort`; `static_assert` with exact D2.1 message; `typedef` for `value_type` and `diff_t` (P4); early-exit `n < 2`; `state.buffer.reserve(n/2)` before loop (D1.2); `compute_minrun<Iter>(n)` explicit template arg required for deduction; canonical scan loop with `count_run` + `binary_insertion_sort` extension + `merge_collapse`; final `merge_force_collapse`.
- `timsort(Iter,Iter)`: delegates to two-arg overload with `std::less<value_type>()` (D2.2 — NOT `std::less<>()` which requires C++14); no redundant `static_assert`.
- `sort.hpp`: one line added before `#endif`; no existing includes modified or reordered.
- All ACs verified: AC1 (direct header + ascending sort), AC2 (sort.hpp include), AC3 (static_assert fires for `std::list<int>::iterator`), AC4 (default comparator sorts ascending), stability (pairs with equal keys preserve relative order). FR-5 edge cases (empty, single-element) verified. All 6 runtime assertions passed.
- Zero warnings from timsort code under `-std=c++11 -Wall -Wextra -Wpedantic`. Pre-existing deprecation warnings in `spreadsort/type_traits.hpp` (system Boost) are unrelated to this story.

### File List

- `libs/sort/include/boost/sort/timsort/timsort.hpp` (MODIFY)
- `libs/sort/include/boost/sort/sort.hpp` (MODIFY)

### Change Log

- 2026-06-18: Story 1.4 created — public API overloads and library integration.
- 2026-06-18: Story 1.4 implemented — `timsort(Iter,Iter,Compare)` and `timsort(Iter,Iter)` added to `namespace boost::sort`; `sort.hpp` updated with timsort include; all ACs verified; zero timsort warnings.
