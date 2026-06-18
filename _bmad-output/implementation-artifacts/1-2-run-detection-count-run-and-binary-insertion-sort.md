---
baseline_commit: 48bf72e9fd90297bbcdebb6e13152fd2f02be158
---

# Story 1.2: Run Detection — `count_run` and `binary_insertion_sort`

Status: done

## Story

As a C++ developer contributing to boost::sort,
I want `tim_detail::count_run` and `tim_detail::binary_insertion_sort` implemented,
so that the algorithm can detect natural sorted order in the input and extend short runs to minrun length before merging.

## Acceptance Criteria

1. **count_run ascending:** `count_run(first, last, comp)` called on strictly ascending `[1, 2, 3, 4, 5]` returns 5; range is unmodified.

2. **count_run descending:** `count_run` called on strictly descending `[5, 4, 3, 2, 1]` returns 5 and reverses the range in-place to `[1, 2, 3, 4, 5]`.

3. **count_run mixed:** `count_run` called on `[3, 1, 4, 1, 5]` returns 2 (the descending run `[3, 1]` is detected, reversed in-place to `[1, 3]`, and the run length 2 is returned). *(See Dev Notes — AC 3 Discrepancy.)*

4. **count_run equal prefix:** `count_run` called on `[3, 3, 3, 2, 1]` returns 3 (equal-element prefix `[3, 3, 3]` is treated as non-descending/ascending, NOT reversed; the run stops where `comp(next, prev)` is first true).

5. **binary_insertion_sort basic:** `binary_insertion_sort(first, last, start, comp)` where `[first, start)` is already sorted and `start < last` — after call, all elements in `[first, last)` are sorted correctly per `comp`.

6. **binary_insertion_sort stability:** `binary_insertion_sort` called on a range of `std::pair<int,int>` sorted by first element, with duplicate keys — pairs with equal first elements retain their original relative order (stability preserved by the upper-bound binary search).

7. **Zero warnings:** Both functions compiled with `-std=c++11 -Wall -Wextra -Wpedantic` produce zero warnings.

## Tasks / Subtasks

- [x] Task 1: Add `binary_insertion_sort` to `tim_detail` (AC: 5, 6, 7)
  - [x] Add function after `compute_minrun` in `timsort.hpp`, before `count_run` (P2 ordering: position 3 in namespace)
  - [x] Use `typedef` in function body for `value_type` alias (P4 rule)
  - [x] Implement upper-bound-style binary search: `comp(pivot, *mid)` as the search predicate — finds first position where pivot < *mid, so equal elements from the left stay to the left of the new element (stability)
  - [x] Shift elements right using a backward walk (do NOT use `std::copy_backward` on overlapping ranges with move semantics)
  - [x] Use `std::move_if_noexcept(*src)` when saving the pivot element and when storing it back (D3.4)
  - [x] Handle the trivial case: if `start == first`, advance start by one before the main loop (first element is already a sorted run of 1)
  - [x] Recompile; confirm zero warnings

- [x] Task 2: Add `count_run` to `tim_detail` (AC: 1, 2, 3, 4, 7)
  - [x] Add function after `binary_insertion_sort` in `timsort.hpp` (P2 ordering: position 4)
  - [x] Handle edge cases: if range has 0 or 1 element, return the size immediately
  - [x] Check `comp(*(first+1), *first)`: if true → strictly descending branch; else → non-descending branch
  - [x] Descending branch: scan forward while `comp(*cur, *(cur-1))` is true, increment run length; then call `std::reverse(first, cur)` to produce an ascending sequence in-place
  - [x] Non-descending branch: scan forward while `!comp(*cur, *(cur-1))` is true (equal elements are non-descending — NOT reversed; this preserves stability)
  - [x] Return the run length as `iter_diff_t<Iter>`
  - [x] Recompile; confirm zero warnings

- [x] Task 3: Write and run verification tests (AC: 1, 2, 3, 4, 5, 6)
  - [x] Write a standalone `test_main()` in a temporary `#if 0` block OR a small separate `.cpp` file verifying all ACs
  - [x] Verify `count_run` ACs 1–4 with exact values and post-condition checks
  - [x] Verify `binary_insertion_sort` AC 5 with int range; AC 6 with `std::pair<int,int>` range checking `second` is non-decreasing within equal-key groups
  - [x] Run all verifications; confirm all pass
  - [x] Remove any temporary test main before marking complete (formal tests are Story 2.1)

- [x] Task 4: Final compile and grep validation (AC: 7)
  - [x] Compile `timsort.hpp` standalone with `-std=c++11 -Wall -Wextra -Wpedantic` — zero warnings
  - [x] Confirm `grep "Iter::value_type" libs/sort/include/boost/sort/timsort/timsort.hpp` returns zero code-level matches
  - [x] Confirm `grep -r "using namespace" libs/sort/include/boost/sort/timsort/` returns zero matches
  - [x] Confirm `grep "#include" libs/sort/include/boost/sort/timsort/timsort.hpp` still shows exactly 6 lines (no new includes added)

## Dev Notes

### AC 3 Discrepancy — Must Read

The epics document states `count_run([3, 1, 4, 1, 5])` returns 1 with comment "the ascending run is length 1". **This appears to be an error in the spec.** The canonical timsort `count_run` detects `[3, 1]` as a strictly descending run (len=2), reverses it in-place to `[1, 3]`, and returns 2. The story ACs above use 2 (canonical behavior). The epics comment was describing the ascending-run perspective, not the descending-detection result. Implement canonical behavior.

### Where to Add the Functions (CRITICAL — P2 Ordering)

Current `timsort.hpp` has 89 lines. The closing `} // namespace tim_detail` is at line 87. Insert new functions BEFORE that line in this order:

```
// After compute_minrun (line ~85), before closing } // namespace tim_detail:
// 3. binary_insertion_sort
// 4. count_run
```

Do NOT reorder existing content. The P2 canonical ordering inside `tim_detail`:
1. `iter_value_t` / `iter_diff_t` aliases (lines ~41-45) ← already present
2. `SortState` struct (lines ~54-59) ← already present
3. `RunEntry` struct (lines ~65-69) ← already present  
4. `compute_minrun` (lines ~77-85) ← already present
5. **`binary_insertion_sort`** ← ADD HERE (Task 1)
6. **`count_run`** ← ADD HERE (Task 2)

Story 1.3 will add gallop functions and merge functions after these.

### Exact Function Signatures (P1 — no synonyms)

```cpp
// Position 5 in tim_detail (after compute_minrun):
template <typename Iter, typename Compare>
void binary_insertion_sort(Iter first, Iter last, Iter start, Compare comp);

// Position 6 in tim_detail (after binary_insertion_sort):
template <typename Iter, typename Compare>
iter_diff_t<Iter> count_run(Iter first, Iter last, Compare comp);
```

### canonical `binary_insertion_sort` Implementation Guide

```cpp
template <typename Iter, typename Compare>
void binary_insertion_sort(Iter first, Iter last, Iter start, Compare comp) {
    typedef typename std::iterator_traits<Iter>::value_type value_type;
    if (first == start)
        ++start;
    for (; start != last; ++start) {
        // 1. Save pivot using move_if_noexcept (D3.4)
        value_type pivot = std::move_if_noexcept(*start);
        // 2. Upper-bound binary search: find first pos where comp(pivot, *pos)
        //    Inserts AFTER all equal elements → stability preserved
        Iter left = first, right = start;
        while (left < right) {
            Iter mid = left + (right - left) / 2;
            if (comp(pivot, *mid))
                right = mid;
            else
                left = mid + 1;
        }
        // left is the insertion point
        // 3. Shift [left, start) right by 1 using backward walk
        for (Iter ptr = start; ptr != left; --ptr)
            *ptr = std::move_if_noexcept(*(ptr - 1));
        // 4. Place pivot
        *left = std::move_if_noexcept(pivot);
    }
}
```

**Why upper-bound (`left = mid + 1` on equal):** For stability, when `pivot` equals `*mid`, we prefer inserting AFTER `*mid`. Moving `left` past `mid` achieves this. If we moved `right` to `mid` on equal, we'd insert BEFORE equal elements, reversing their order — a stability bug.

**Do NOT use `std::copy_backward`** for the shift: it works but requires care with move semantics. The explicit backward walk is clearer and avoids accidentally mixing move and copy paths.

### Canonical `count_run` Implementation Guide

```cpp
template <typename Iter, typename Compare>
iter_diff_t<Iter> count_run(Iter first, Iter last, Compare comp) {
    if (first == last) return 0;
    Iter cur = first;
    ++cur;
    if (cur == last) return 1;

    iter_diff_t<Iter> n = 2;
    if (comp(*cur, *first)) {
        // Strictly descending: scan while each element < previous
        while (++cur != last && comp(*cur, *(cur - 1)))
            ++n;
        std::reverse(first, cur);   // make ascending in-place
    } else {
        // Non-descending (ascending OR equal): scan while each element >= previous
        while (++cur != last && !comp(*cur, *(cur - 1)))
            ++n;
    }
    return n;
}
```

**Why `comp(*cur, *first)` not `comp(*(cur-1), *first)`:** We compare the first two elements at entry to decide ascending vs. descending. Both `*cur` and `*first` are valid at this point since `n=2` and we haven't advanced `cur` yet in the loop.

**Stability of descending detection:** Equal elements satisfy `!comp(b, a)` AND `!comp(a, b)`. At the decision point, `comp(*cur, *first)` is `false` for equal elements (equal is not strictly less). So equal-prefix inputs take the non-descending branch — they are NOT reversed. This preserves stability for equal elements that appeared in original order.

### P4 — typedef vs using in Function Bodies

Inside both function bodies, use `typedef` (NOT `using`):
```cpp
// CORRECT (C++11 compat):
typedef typename std::iterator_traits<Iter>::value_type value_type;

// AVOID in function bodies (may warn on some C++11 compilers):
// using value_type = typename std::iterator_traits<Iter>::value_type;
```

`using` aliases are fine at namespace scope (the existing `iter_value_t` and `iter_diff_t`).

### D3.3 — iterator_traits aliases

Never write `Iter::value_type` directly anywhere. Always use `iter_value_t<Iter>` (at namespace/class scope) or `typedef typename std::iterator_traits<Iter>::value_type value_type;` (in function bodies). The existing aliases at lines ~41-45 are already in scope.

### D3.4 — move_if_noexcept

Use `std::move_if_noexcept(*src)` when copying elements in `binary_insertion_sort`. `<utility>` is already included from Story 1.1. Falls back to copy when the move constructor may throw, preserving source range integrity.

### SortState NOT passed to these functions

Per D1.1: "Run detection and insertion sort do not receive [SortState]." Do NOT add a `SortState&` parameter to either function. SortState is only passed to merge functions (Stories 1.3+).

### Verification Pattern (same as Story 1.1 Task 6)

No test framework exists until Story 2.1. Use a standalone verification:

```bash
# From the boost superproject root:
g++ -std=c++11 -Wall -Wextra -Wpedantic \
    -I libs/sort/include \
    test_12_verify.cpp -o test_12_verify && ./test_12_verify
```

Or inline in a `#if 0` block inside a temporary `.cpp`. Remove before marking complete.

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

### Files Modified

| File | Action |
|------|--------|
| `libs/sort/include/boost/sort/timsort/timsort.hpp` | MODIFY — add `binary_insertion_sort` and `count_run` inside `tim_detail` |

No other files touched. `sort.hpp` update is Story 1.4. Test file is Story 2.1.

### Cross-Story Context

- **Story 1.1 (done):** Created the skeleton and types. `timsort.hpp` is 89 lines; `compute_minrun` is the last function before closing namespace braces.
- **Story 1.3 (next):** Will add `gallop_left`, `gallop_right`, `merge_lo`, `merge_hi`, `merge_collapse`, `merge_force_collapse` — all require `SortState`. Run detection and insertion sort do NOT.
- **Story 1.4:** Public API overloads. The main sort loop calls `count_run` then `binary_insertion_sort` (extend to minrun), then pushes to the run stack.
- **Story 2.1:** Creates `test/test_timsort.cpp` using Boost.Test — this is the formal test home.

### Build System Warning (from Story 1.1 Dev Notes)

The Jamfile and CMakeLists in `libs/sort/test/` use EXPLICIT file listings (not wildcards). `test_timsort.cpp` will require explicit registration in both files — that work belongs to Story 2.1, NOT this story.

### References

- [Source: architecture.md#P1] — Canonical function names and signatures
- [Source: architecture.md#P2] — Namespace ordering (`binary_insertion_sort` before `count_run`)
- [Source: architecture.md#P3] — Gallop tie-breaking (not this story, but stability principle carries through)
- [Source: architecture.md#P4] — typedef vs using
- [Source: architecture.md#D1.1] — SortState not passed to run detection/insertion sort
- [Source: architecture.md#D3.3] — iterator_traits alias enforcement
- [Source: architecture.md#D3.4] — move_if_noexcept in merge buffer (applies here too)
- [Source: epics.md#Story-1.2] — Acceptance criteria (note AC 3 discrepancy documented above)
- [Source: implementation-artifacts/1-1-header-file-skeleton-foundational-types-and-compute-minrun.md] — Previous story patterns

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6 (context engine / create-story)

### Debug Log References

### Completion Notes List

- `binary_insertion_sort`: upper-bound binary search (left = mid+1 on equal) preserves stability; `typedef` for value_type in function body (P4); `std::move_if_noexcept` for pivot save/restore and element shifting (D3.4); `if (first == start) ++start` handles trivial case.
- `count_run`: detects strictly descending runs via `comp(*cur, *first)` at entry; equal elements go to non-descending branch and are NOT reversed (stability); `std::reverse` used for in-place ascending correction; edge cases (n=0, n=1) handled before main logic.
- AC 3 clarification: `count_run([3,1,4,1,5])` returns 2 (canonical behavior — detects descending run [3,1], reverses to [1,3]); the epics note "returns 1" was a spec error as documented in Dev Notes.
- All 9 verification tests passed (4 for binary_insertion_sort, 5 for count_run); zero warnings under `-std=c++11 -Wall -Wextra -Wpedantic`; 6 includes unchanged; no `Iter::value_type` in code; no `using namespace`.
- Temporary `test_12_verify.cpp` removed before completion.

### File List

- `libs/sort/include/boost/sort/timsort/timsort.hpp` (MODIFY)

### Change Log

- 2026-06-17: Story 1.2 created — run detection and binary insertion sort.
- 2026-06-17: Story 1.2 implemented — `binary_insertion_sort` (P2 pos 3) and `count_run` (P2 pos 4) added to `tim_detail`; all ACs verified; zero warnings.
