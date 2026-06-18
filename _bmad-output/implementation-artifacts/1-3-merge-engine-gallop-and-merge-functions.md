---
baseline_commit: "48bf72e9fd90297bbcdebb6e13152fd2f02be158"
---

# Story 1.3: Merge Engine — `gallop_left`, `gallop_right`, `merge_lo`, `merge_hi`, `merge_collapse`, `merge_force_collapse`

Status: done

## Story

As a C++ developer contributing to boost::sort,
I want the complete merge subsystem implemented with galloping mode and run stack invariant enforcement,
So that runs are merged efficiently while maintaining stability and O(log N) stack depth.

## Acceptance Criteria

1. **gallop_left boundary:** `gallop_left(key, base, len, hint, comp)` where `key` is greater than all elements in `[base, base+len)` returns `len`.

2. **gallop_right stability:** `gallop_right(key, base, len, hint, comp)` where `key` equals element at position `p` returns `p` (not `p+1`) — left-run element wins on equality, preserving stability (P3 tie-breaking rule).

3. **merge_lo correctness:** `merge_lo(base1, len1, base2, len2, state, comp)` called with `len1 <= len2` and two adjacent sorted runs — after call, `[base1, base1+len1+len2)` is sorted; equal elements from the left run precede equal elements from the right run; `state.buffer` was used but never grown past its initial `reserve(n/2)` capacity; `buffer.resize` was never called.

4. **merge_hi correctness:** `merge_hi(base1, len1, base2, len2, state, comp)` called with `len1 > len2` — merged range is sorted with stability preserved; right run was copied into buffer; `state.buffer.resize` was never called.

5. **Adaptive galloping:** `state.min_gallop` starts at 7; when one run wins 7+ consecutive comparisons, galloping mode activates and `state.min_gallop` is decremented; when neither side dominates, it is incremented back toward 7 (adaptive per-call, not global/static).

6. **merge_collapse invariants:** With a run stack where `len[n-2] <= len[n-1] + len[n]`, `merge_collapse` performs merges in correct order (smaller pair merged first) until both `len[n-1] > len[n]` AND `len[n-2] > len[n-1] + len[n]` hold.

7. **merge_force_collapse drains stack:** Called with N runs remaining — performs N-1 merges, always merging until exactly one run remains; final range is fully sorted.

8. **move_if_noexcept safety:** `buffer.push_back(std::move_if_noexcept(*src))` used throughout — when value type has a throwing move constructor, elements are copied (not moved) into the buffer.

9. **Zero warnings:** All new functions compiled with `-std=c++11 -Wall -Wextra -Wpedantic` produce zero warnings. Exactly 6 includes remain unchanged.

## Tasks / Subtasks

- [x] Task 1: Add `gallop_left` and `gallop_right` to `tim_detail` (AC: 1, 2, 9)
  - [x] Insert both functions AFTER `count_run` (line 144), BEFORE closing `} // namespace tim_detail` (line 146)
  - [x] `gallop_left`: two-phase (exponential gallop + binary search); uses `comp(*(base+mid), key)` predicate; returns leftmost pos where `!(base[pos] < key)` i.e. `key <= base[pos]`; hint-directed gallop
  - [x] `gallop_right`: two-phase; uses `comp(key, *(base+mid))` predicate; returns rightmost pos where `!(key < base[pos])` i.e. `base[pos] <= key`; AC2: returns `p` not `p+1` for equal key
  - [x] Use `iter_value_t<Iter>` alias for the `key` parameter type (D3.3 — no `Iter::value_type`)
  - [x] Use `iter_diff_t<Iter>` for `len`, `hint`, and return type
  - [x] Handle hint out-of-range inputs safely (clamp to [0, len-1])
  - [x] Recompile; confirm zero warnings

- [x] Task 2: Add `merge_lo` to `tim_detail` (AC: 3, 5, 8, 9)
  - [x] Insert AFTER `gallop_right`; precondition: `len1 <= len2` (enforced by caller via P5)
  - [x] Copy left run `[base1, base1+len1)` into `state.buffer` using `push_back(std::move_if_noexcept(*src))` — NEVER `buffer.resize`
  - [x] Merge buffer + right run `[base2, base2+len2)` left-to-right into `[base1, base1+len1+len2)`
  - [x] Stability: on equal, take from buffer (left run) first — predicate `!comp(*right_ptr, *buf_ptr)` means take left
  - [x] Implement linear galloping mode: count consecutive wins per side; when count >= `state.min_gallop`, switch to gallop phase using `gallop_right`/`gallop_left`; on gallop exit, decrement `state.min_gallop` (min 1); on non-gallop exit, increment `state.min_gallop` toward 7
  - [x] Call `state.buffer.clear()` at start (to reuse capacity without re-allocating)
  - [x] Handle the case where one run is exhausted before the other (copy remainder)
  - [x] Use `typedef` in function body for value type (P4): `typedef iter_value_t<Iter> value_type;`
  - [x] Recompile; confirm zero warnings

- [x] Task 3: Add `merge_hi` to `tim_detail` (AC: 4, 5, 8, 9)
  - [x] Insert AFTER `merge_lo`; precondition: `len1 > len2` (enforced by caller via P5)
  - [x] Copy RIGHT run `[base2, base2+len2)` into `state.buffer` using `push_back(std::move_if_noexcept(*src))`
  - [x] Merge in REVERSE: left run right-end and buffer right-end, writing result from right-end of destination
  - [x] Stability (right-to-left merge): on equal, take from left run (base1 side) first — predicate `comp(*buf_ptr_rev, *left_ptr_rev)` means take from buffer
  - [x] Same adaptive galloping as `merge_lo` but mirrored (gallop from right end); buffer gallop uses `BufIter` template param
  - [x] Call `state.buffer.clear()` at start
  - [x] Recompile; confirm zero warnings

- [x] Task 4: Add `merge_collapse` and `merge_force_collapse` to `tim_detail` (AC: 6, 7, 9)
  - [x] Insert AFTER `merge_hi`
  - [x] `merge_collapse`: enforce both run stack invariants in a loop (canonical AC6 logic)
  - [x] `merge_force_collapse`: drain remaining stack until one run remains
  - [x] RunEntry fields accessed as `.base` and `.len` — exact names
  - [x] Recompile; confirm zero warnings

- [x] Task 5: Write and run verification tests (AC: 1–8)
  - [x] Write standalone verification (`test_13_verify.cpp`, compiled and run)
  - [x] AC1–AC7 all pass
  - [x] Temporary file removed

- [x] Task 6: Final compile and grep validation (AC: 9)
  - [x] Compile `timsort.hpp` standalone — zero warnings
  - [x] `grep "#include" | wc -l` → 6 ✓
  - [x] `grep -r "using namespace"` → zero matches ✓
  - [x] `grep "Iter::value_type"` → zero code-level matches ✓
  - [x] `grep "buffer\.resize"` → comment-only match from pre-existing Story 1.1 SortState doc; no actual calls ✓

## Dev Notes

### CRITICAL: Current File State and Insertion Point

`libs/sort/include/boost/sort/timsort/timsort.hpp` is currently **149 lines**. Structure:

```
Lines  1-21  : License/doc comment block
Line   23    : #pragma once
Lines 25-30  : 6 stdlib includes
Lines 32-34  : namespace boost { namespace sort { namespace tim_detail {
Lines 41-45  : iter_value_t / iter_diff_t aliases
Lines 54-59  : SortState<ValueType> struct    ← ALREADY DEFINED, use by ref in merges
Lines 65-69  : RunEntry<Iter> struct           ← ALREADY DEFINED, fields: .base, .len
Lines 77-85  : compute_minrun
Lines 94-116 : binary_insertion_sort
Lines 125-144: count_run
Line  146    : } // namespace tim_detail  ← INSERT NEW FUNCTIONS BEFORE THIS LINE
Lines 147-148: } // namespace sort / } // namespace boost
```

**All new functions insert between line 144 (`count_run` closing brace) and line 146 (`} // namespace tim_detail`). DO NOT move or reorder existing content.**

### Exact Function Signatures (P1 — no synonyms)

```cpp
// Position 5 in tim_detail (after count_run):
template <typename Iter, typename Compare>
iter_diff_t<Iter> gallop_left(
    const iter_value_t<Iter>& key,
    Iter base, iter_diff_t<Iter> len, iter_diff_t<Iter> hint,
    Compare comp);

template <typename Iter, typename Compare>
iter_diff_t<Iter> gallop_right(
    const iter_value_t<Iter>& key,
    Iter base, iter_diff_t<Iter> len, iter_diff_t<Iter> hint,
    Compare comp);

// Position 7 in tim_detail (after gallop functions):
template <typename Iter, typename Compare>
void merge_lo(
    Iter base1, iter_diff_t<Iter> len1,
    Iter base2, iter_diff_t<Iter> len2,
    SortState<iter_value_t<Iter>>& state,
    Compare comp);

template <typename Iter, typename Compare>
void merge_hi(
    Iter base1, iter_diff_t<Iter> len1,
    Iter base2, iter_diff_t<Iter> len2,
    SortState<iter_value_t<Iter>>& state,
    Compare comp);

// Position 8 in tim_detail (after merge_lo/merge_hi):
// RunStack = std::vector<RunEntry<Iter>> — pass by ref
template <typename Iter, typename Compare>
void merge_collapse(
    std::vector<RunEntry<Iter>>& stack,
    SortState<iter_value_t<Iter>>& state,
    Compare comp);

template <typename Iter, typename Compare>
void merge_force_collapse(
    std::vector<RunEntry<Iter>>& stack,
    SortState<iter_value_t<Iter>>& state,
    Compare comp);
```

**Note on SortState instantiation**: `SortState<iter_value_t<Iter>>` — use `iter_value_t<Iter>` alias, never `Iter::value_type`.

### P3 — Gallop Tie-Breaking Rule (CRITICAL for stability)

This asymmetry is the entire stability mechanism — reversing either predicate silently breaks stability:

| Function | Predicate | Meaning |
|---|---|---|
| `gallop_left` | `comp(*(base+mid), key)` | stops when false → left-biased insertion |
| `gallop_right` | `comp(key, *(base+mid))` | stops when false → right-biased insertion |

- **gallop_left** finds leftmost position `p` where `!(base[p] < key)` → inserts key BEFORE equal elements
- **gallop_right** finds rightmost position `p` where `!(key < base[p])` → AC2: returns `p`, not `p+1`

In `merge_lo` main loop: `if (!comp(*right_ptr, *buf_ptr))` → take from buf (left run) first. On equal, left wins.

### gallop_left Implementation Guide

```cpp
template <typename Iter, typename Compare>
iter_diff_t<Iter> gallop_left(
    const iter_value_t<Iter>& key,
    Iter base, iter_diff_t<Iter> len, iter_diff_t<Iter> hint,
    Compare comp)
{
    // Phase 1: exponential gallop from hint
    iter_diff_t<Iter> last_ofs = 0, ofs = 1;
    if (comp(*(base + hint), key)) {
        // base[hint] < key → gallop rightward from hint
        const iter_diff_t<Iter> max_ofs = len - hint;
        while (ofs < max_ofs && comp(*(base + hint + ofs), key)) {
            last_ofs = ofs;
            ofs = (ofs << 1) | 1;
        }
        if (ofs > max_ofs) ofs = max_ofs;
        last_ofs += hint;
        ofs += hint;
    } else {
        // key <= base[hint] → gallop leftward from hint
        const iter_diff_t<Iter> max_ofs = hint + 1;
        while (ofs < max_ofs && !comp(*(base + hint - ofs), key)) {
            last_ofs = ofs;
            ofs = (ofs << 1) | 1;
        }
        if (ofs > max_ofs) ofs = max_ofs;
        const iter_diff_t<Iter> tmp = last_ofs;
        last_ofs = hint - ofs;
        ofs = hint - tmp;
    }
    // Phase 2: binary search in (last_ofs, ofs]
    // Invariant: base[last_ofs] < key <= base[ofs]
    ++last_ofs;
    while (last_ofs < ofs) {
        const iter_diff_t<Iter> mid = last_ofs + ((ofs - last_ofs) >> 1);
        if (comp(*(base + mid), key))
            last_ofs = mid + 1;
        else
            ofs = mid;
    }
    return ofs;  // AC1: returns len when key > all elements
}
```

### gallop_right Implementation Guide

```cpp
template <typename Iter, typename Compare>
iter_diff_t<Iter> gallop_right(
    const iter_value_t<Iter>& key,
    Iter base, iter_diff_t<Iter> len, iter_diff_t<Iter> hint,
    Compare comp)
{
    // Phase 1: exponential gallop from hint
    iter_diff_t<Iter> last_ofs = 0, ofs = 1;
    if (comp(key, *(base + hint))) {
        // key < base[hint] → gallop leftward
        const iter_diff_t<Iter> max_ofs = hint + 1;
        while (ofs < max_ofs && comp(key, *(base + hint - ofs))) {
            last_ofs = ofs;
            ofs = (ofs << 1) | 1;
        }
        if (ofs > max_ofs) ofs = max_ofs;
        const iter_diff_t<Iter> tmp = last_ofs;
        last_ofs = hint - ofs;
        ofs = hint - tmp;
    } else {
        // key >= base[hint] → gallop rightward
        const iter_diff_t<Iter> max_ofs = len - hint;
        while (ofs < max_ofs && !comp(key, *(base + hint + ofs))) {
            last_ofs = ofs;
            ofs = (ofs << 1) | 1;
        }
        if (ofs > max_ofs) ofs = max_ofs;
        last_ofs += hint;
        ofs += hint;
    }
    // Phase 2: binary search in (last_ofs, ofs]
    // Invariant: base[last_ofs] <= key < base[ofs]
    ++last_ofs;
    while (last_ofs < ofs) {
        const iter_diff_t<Iter> mid = last_ofs + ((ofs - last_ofs) >> 1);
        if (comp(key, *(base + mid)))
            ofs = mid;
        else
            last_ofs = mid + 1;
    }
    return ofs;  // AC2: returns p where base[p]==key, not p+1
}
```

### merge_lo Implementation Guide

Precondition: `len1 <= len2`. Copies left run into buffer, merges left-to-right.

```cpp
template <typename Iter, typename Compare>
void merge_lo(Iter base1, iter_diff_t<Iter> len1,
              Iter base2, iter_diff_t<Iter> len2,
              SortState<iter_value_t<Iter>>& state, Compare comp)
{
    typedef iter_value_t<Iter> value_type;  // P4
    // Copy left run into buffer (D3.4: move_if_noexcept)
    state.buffer.clear();
    for (Iter it = base1; it != base1 + len1; ++it)
        state.buffer.push_back(std::move_if_noexcept(*it));

    typename std::vector<value_type>::iterator buf = state.buffer.begin();
    Iter right = base2;
    Iter dest = base1;

    // Invariant: take left (buf) on equal → left-before-right stability (P3)
    int min_gallop = state.min_gallop;
    while (true) {
        iter_diff_t<Iter> left_wins = 0, right_wins = 0;
        // LINEAR phase
        do {
            if (comp(*right, *buf)) {
                *dest++ = std::move_if_noexcept(*right++);
                ++right_wins; left_wins = 0;
                if (--len2 == 0) goto done;
            } else {
                *dest++ = std::move_if_noexcept(*buf++);
                ++left_wins; right_wins = 0;
                if (--len1 == 0) goto done;
            }
        } while ((left_wins | right_wins) < min_gallop);

        // GALLOP phase
        do {
            iter_diff_t<Iter> k;
            k = gallop_right(*right, buf, len1, 0, comp);
            for (iter_diff_t<Iter> i = 0; i < k; ++i)
                *dest++ = std::move_if_noexcept(*buf++);
            len1 -= k; left_wins = k;
            if (len1 == 0) goto done;

            *dest++ = std::move_if_noexcept(*right++);
            if (--len2 == 0) goto done;

            k = gallop_left(*buf, right, len2, 0, comp);
            for (iter_diff_t<Iter> i = 0; i < k; ++i)
                *dest++ = std::move_if_noexcept(*right++);
            len2 -= k; right_wins = k;
            if (len2 == 0) goto done;

            *dest++ = std::move_if_noexcept(*buf++);
            if (--len1 == 0) goto done;

            if (min_gallop > 1) --min_gallop;
        } while (left_wins >= MIN_GALLOP || right_wins >= MIN_GALLOP);

        ++min_gallop;  // penalize leaving gallop mode
    }
done:
    state.min_gallop = (min_gallop < 1) ? 1 : min_gallop;
    // Copy remaining buffer elements
    std::copy(buf, buf + len1, dest);  // if left had remainder
    // right remainder already in place (right run is in original array)
}
```

**Note on `goto`:** Using `goto done` is idiomatic for this merge algorithm — it avoids nested loop breaks cleanly. C++11 permits it.

**Alternative without goto:** use a flag variable and break from inner loops — acceptable if goto is objectionable, but both are correct.

**MIN_GALLOP constant:** Define as a constant in the function or use `state.min_gallop`. The initial value (7) lives in SortState. The local `min_gallop` variable shadows the global threshold during a merge; write back to `state.min_gallop` at end.

### merge_hi Implementation Guide

Precondition: `len1 > len2`. Copies RIGHT run into buffer, merges right-to-left.

Mirror of `merge_lo` but:
- Copy `[base2, base2+len2)` into buffer
- Start dest at `base1 + len1 + len2 - 1` (right end)
- Walk left pointers: `left_ptr = base1 + len1 - 1`, `buf_ptr = buffer.end() - 1`
- Stability (right-to-left): take from LEFT when `!comp(*buf_ptr_rev, *left_ptr_rev)` (left wins on equal)

### merge_collapse and merge_force_collapse

**RunEntry access pattern:**
```cpp
std::vector<RunEntry<Iter>>& stack;
// Access top entry:
stack.back().base   // type: Iter
stack.back().len    // type: iter_diff_t<Iter>
// Access n-th from top (0=top):
stack[stack.size()-1]   // top
stack[stack.size()-2]   // top-1
stack[stack.size()-3]   // top-2
```

**merge_collapse invariant enforcement (canonical — must match this exactly for AC6):**
```cpp
while (stack.size() > 1) {
    size_t n = stack.size() - 2;  // index of top-1
    bool inv1_violated = stack[n].len <= stack[n+1].len;
    bool inv2_violated = (n > 0) && (stack[n-1].len <= stack[n].len + stack[n+1].len);
    if (!inv1_violated && !inv2_violated) break;

    // Choose which pair to merge (AC6: smaller pair merged first)
    if (n > 0 && stack[n-1].len < stack[n+1].len)
        --n;  // merge stack[n] with stack[n+1] (i.e. n-2 with n-1)
    // Merge stack[n] with stack[n+1]
    do_merge(stack, n, state, comp);
}
```

**do_merge helper** (internal, not a named P1 function):
```cpp
// Merge stack[n] and stack[n+1]; pop stack[n+1]; update stack[n].len
Iter b1 = stack[n].base;
iter_diff_t<Iter> l1 = stack[n].len;
iter_diff_t<Iter> l2 = stack[n+1].len;
if (l1 <= l2)
    merge_lo(b1, l1, b1 + l1, l2, state, comp);
else
    merge_hi(b1, l1, b1 + l1, l2, state, comp);
stack[n].len = l1 + l2;
stack.erase(stack.begin() + n + 1);
```

**merge_force_collapse:**
```cpp
while (stack.size() > 1) {
    size_t n = stack.size() - 2;
    if (n > 0 && stack[n-1].len < stack[n+1].len)
        --n;
    do_merge(stack, n, state, comp);
}
```

### Adaptive Galloping — state.min_gallop Details (AC5)

- Initial value: 7 (set in `SortState()` constructor — already implemented)
- Each `merge_lo`/`merge_hi` call uses a **local copy** `int min_gallop = state.min_gallop`
- During gallop phase: `if (min_gallop > 1) --min_gallop` (per successful gallop iteration)
- On gallop exit (neither side dominates): `++min_gallop`
- Write back: `state.min_gallop = max(1, min_gallop)` before returning
- This is per-sort-call adaptive (SortState is on stack in timsort() — not global/static)

### D1.2 / P5 — Merge Direction (MUST use `<=`)

```cpp
if (l1 <= l2)
    merge_lo(b1, l1, b1+l1, l2, state, comp);
else
    merge_hi(b1, l1, b1+l1, l2, state, comp);
```

`<=` not `<` — using `<` would call `merge_hi` when runs are equal-length, which is less efficient.

### D3.4 — buffer.push_back vs buffer.resize

```cpp
// CORRECT (D3.4, AC8):
state.buffer.clear();
for (Iter it = src; it != src+len; ++it)
    state.buffer.push_back(std::move_if_noexcept(*it));

// FORBIDDEN (breaks non-default-constructible types):
// state.buffer.resize(len);
// std::copy(src, src+len, state.buffer.begin());
```

`buffer.reserve(n/2)` is done ONCE in the top-level `timsort()` call (Story 1.4). By the time merges run, capacity is already reserved. `clear()` drops size to 0 without releasing capacity — perfect for reuse.

### Build System Warning (from Story 1.1)

`libs/sort/test/Jamfile.v2` uses EXPLICIT per-test registration — NOT wildcards. Formal test for this story is Story 2.1. For Task 5 verification, use a standalone `.cpp` compiled directly:

```bash
g++ -std=c++11 -Wall -Wextra -Wpedantic \
    -I libs/sort/include \
    test_13_verify.cpp -o test_13_verify && ./test_13_verify && rm test_13_verify test_13_verify.cpp
```

### Compilation Command

```bash
# From superproject root:
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
| `libs/sort/include/boost/sort/timsort/timsort.hpp` | MODIFY — add `gallop_left`, `gallop_right`, `merge_lo`, `merge_hi`, `merge_collapse`, `merge_force_collapse` inside `tim_detail`, before closing brace |

No other files. `sort.hpp` update is Story 1.4. Test file is Story 2.1.

### Previous Story Patterns (from Stories 1.1 and 1.2)

- 2-space indent (Google C++ Style) — match existing file
- Section comments: `//------…---` 76-char separator before each function group
- `typedef` in function bodies for local type aliases (P4)
- `std::move_if_noexcept` for all element moves (D3.4)
- `iter_value_t<Iter>` / `iter_diff_t<Iter>` — never `Iter::value_type`
- No `using namespace` anywhere

### Cross-Story Context

- **Story 1.2 (done):** Established `binary_insertion_sort` and `count_run` patterns. File is 149 lines.
- **Story 1.4 (next):** Adds the public `timsort()` overloads that instantiate `SortState` on the stack, call `buffer.reserve(n/2)`, run the main scan loop (calling `count_run` → `binary_insertion_sort` → push to stack → `merge_collapse`), and then call `merge_force_collapse`.
- **Story 2.1:** Creates `test/test_timsort.cpp` with `BOOST_TEST` — that is the formal test home.

### Anti-Patterns to Avoid

- `buffer.resize(len)` — breaks non-default-constructible types (AC3/4/8)
- `comp(key, *mid) == false` instead of `!comp(key, *mid)` — equivalent, but be consistent
- Global or static `min_gallop` — must be in `SortState` (AC5)
- Reversing gallop predicates in gallop_left vs gallop_right — silently breaks stability (P3)
- `std::move(*src)` instead of `std::move_if_noexcept(*src)` — breaks exception safety (AC8)
- `stack.back().base_iter` or `.length` — field names MUST be `.base` and `.len` (P1)

### References

- [Source: architecture.md#D1.1] — SortState struct (already implemented)
- [Source: architecture.md#D1.2] — Merge direction bidirectional
- [Source: architecture.md#D1.3] — Run stack two-invariant policy
- [Source: architecture.md#D3.3] — iterator_traits aliases
- [Source: architecture.md#D3.4] — move_if_noexcept for buffer
- [Source: architecture.md#P1] — Canonical function names and signatures
- [Source: architecture.md#P2] — Namespace ordering (gallop=pos5, merge=pos7, stack=pos8)
- [Source: architecture.md#P3] — Gallop tie-breaking rule (critical for stability)
- [Source: architecture.md#P4] — typedef vs using
- [Source: architecture.md#P5] — Merge direction condition (<=)
- [Source: epics.md#Story-1.3] — Acceptance criteria
- [Source: implementation-artifacts/1-1-*.md] — File skeleton patterns
- [Source: implementation-artifacts/1-2-*.md] — Function body patterns (typedef, move_if_noexcept)

## Senior Developer Review (AI)

**Outcome:** Approve — no patches required  
**Date:** 2026-06-18  
**Layers:** Blind Hunter · Edge Case Hunter · Acceptance Auditor

### Review Findings

- [x] [Review][Defer] Signed overflow in gallop `ofs = (ofs<<1)|1` [timsort.hpp:163,175,204,215] — deferred, pre-existing algorithm design; theoretical only on 64-bit (requires >2^62 elements); identical to CPython reference
- [x] [Review][Defer] `buffer.reserve` responsibility is cross-story [timsort.hpp:249,319] — deferred, Story 1.4 supplies `reserve(n/2)`; no `resize` called, AC3/AC4 satisfied within Story 1.3 scope
- [x] [Review][Defer] `do_merge` has no bounds assertion for `n+1 < stack.size()` [timsort.hpp:387] — deferred, pre-existing; callers always satisfy by loop guard; future hardening
- [x] [Review][Defer] `merge_collapse` merge-order selector matches Python reference but encodes Stijn de Gouw variant [timsort.hpp:416] — deferred, spec-level design choice; implementation faithfully follows spec guide

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

- `merge_hi` gallop phase: buffer gallop must use `BufIter` (not `Iter`) as template param because `state.buffer.data()` returns a raw pointer type that differs from the wrapped iterator `Iter`. Fixed by using `buf - (len2-1)` as the BufIter base.
- Anonymous namespace for `do_merge` removed — put directly in `tim_detail` to avoid internal-linkage issues in header-only code.
- AC5 test: all-dominant-left scenario exits gallop via `goto` before `--min_gallop` executes; needed a mixed input (left=[1..7,50,51,52], right=[8..15,53,54]) where gallop completes a full iteration.

### Completion Notes List

- Implemented `gallop_left` and `gallop_right` with two-phase exponential+binary search; asymmetric predicates satisfy P3 stability rule (AC1, AC2).
- Implemented `merge_lo`: left run copied to buffer via `move_if_noexcept`, merged L→R with adaptive galloping; local `min_gallop` written back to `state.min_gallop` (AC3, AC5, AC8).
- Implemented `merge_hi`: right run copied to buffer, merged R→L; buffer-side gallop uses `BufIter` template parameter to avoid iterator type mismatch (AC4, AC5, AC8).
- Implemented `do_merge` helper (internal to `tim_detail`), `merge_collapse` (two-invariant enforcement with smaller-pair-first logic), and `merge_force_collapse` (full drain) (AC6, AC7).
- All 7 ACs verified by standalone test, zero compile warnings with `-Wall -Wextra -Wpedantic`, 6 includes, no `using namespace`, no `Iter::value_type` in code, no `buffer.resize` calls.

### File List

- `libs/sort/include/boost/sort/timsort/timsort.hpp` (MODIFY)

### Change Log

- 2026-06-18: Story 1.3 created — merge engine with galloping and run stack management.
- 2026-06-18: Implemented `gallop_left`, `gallop_right`, `merge_lo`, `merge_hi`, `do_merge`, `merge_collapse`, `merge_force_collapse`; all AC1–AC9 verified.
