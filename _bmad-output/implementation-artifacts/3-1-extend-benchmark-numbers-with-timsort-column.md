---
baseline_commit: "b67b047b2620b3c6cec12186a2754b983cdd4728"
---

# Story 3.1: Extend `benchmark_numbers.cpp` with Timsort Column

Status: done

## Story

As a C++ developer evaluating sorting algorithms,
I want timsort included as a benchmark column in `benchmark/single/benchmark_numbers.cpp` alongside spinsort, flat_stable_sort, and std::stable_sort across five data shapes for `int`,
so that I can empirically compare timsort's wall-clock performance and verify its adaptive advantage on nearly-sorted integer data.

## Acceptance Criteria

1. **4 × 5 benchmark table (FR-14, FR-15):** `benchmark/single/benchmark_numbers.cpp` is modified to add a timsort comparison section; when compiled with `-std=c++11` and run, it produces a results table with exactly 4 algorithm columns (timsort, spinsort, flat_stable_sort, std::stable_sort) × 5 data shape rows (random, already-sorted, reverse-sorted, nearly-sorted, pipe-organ) for `int`; no crash on any shape.

2. **Nearly-sorted comment and generation (P7, D4.3):** The comment immediately above the nearly-sorted timing block reads exactly:
   ```
   // Nearly-sorted: sorted array, N*0.05 random adjacent swaps, mt19937(42)
   ```
   The code uses `std::mt19937` seeded with `42` and performs exactly `static_cast<int>(NELEM_TIM * 0.05)` random adjacent-pair swaps (swap A[i] and A[i+1] for random i in [0, NELEM_TIM-2]).

3. **Pipe-organ shape (FR-14):** The pipe-organ generator produces ascending first half `[0, 1, ..., half-1]` followed by descending second half `[half-1, half-2, ..., 0]` where `half = NELEM_TIM / 2`; all 4 algorithms produce timing output without crash.

4. **Existing table untouched (FR-14 preservation):** The original `Test()` function, all existing `Generator_*` functions, and the original `main()` benchmark section (6-column uint64_t table) are unchanged; only the new timsort comparison section and `RunTimsortBenchmark()` call are new. No existing symbol renamed or removed.

5. **Performance target noted (SM-1):** N = 1,000,000 `int` elements on the nearly-sorted shape. If timsort time is NOT ≥2× faster than spinsort, add the comment `// NOTE: SM-1 target (>=2x vs spinsort on nearly-sorted) not met — calibration issue` immediately below the nearly-sorted row call in `RunTimsortBenchmark()`. Story is still complete regardless.

## Tasks / Subtasks

- [x] Task 1: Add `using bsort::timsort;` alias (AC: 1, 4)
  - [x] Open `libs/sort/benchmark/single/benchmark_numbers.cpp`
  - [x] After `using bsort::pdqsort;` (line ~47), add exactly: `using bsort::timsort;`
  - [x] Do NOT add any other new `using` declarations

- [x] Task 2: Add `NELEM_TIM` constant and forward declarations (AC: 1)
  - [x] After `#define NELEM 100000000` (line ~32), add: `const int NELEM_TIM = 1000000;`
  - [x] Before `main()`, add forward declarations:
    ```cpp
    void RunTimsortBenchmark();
    void TestTimsort(const std::vector<int>& B);
    ```

- [x] Task 3: Implement `TestTimsort()` with 4-column output (AC: 1, 4)
  - [x] Add the function AFTER the existing `Test()` function (after line ~322)
  - [x] Time 4 algorithms in this exact order: timsort, spinsort, flat_stable_sort, stable_sort
  - [x] Use `std::less<int> comp;` — NOT `std::less<uint64_t>`
  - [x] Follow the same `time_point start/finish; subtract_time; V.push_back` pattern as the existing `Test()`
  - [x] Print with same `setprecision(2) fixed setw(5) right` format and trailing `|`
  - [x] See reference implementation below

- [x] Task 4: Implement `RunTimsortBenchmark()` with 5 data shapes (AC: 1, 2, 3, 5)
  - [x] Add after `TestTimsort()`
  - [x] Print a self-contained header block identifying the 4 algorithms by number
  - [x] Print a 4-column table header matching the 5 data shapes
  - [x] **random**: generate N=NELEM_TIM ints with `std::mt19937 rng(123)` using `rng() % NELEM_TIM`
  - [x] **already-sorted**: sequential `[0, 1, ..., NELEM_TIM-1]`
  - [x] **reverse-sorted**: `[NELEM_TIM, NELEM_TIM-1, ..., 1]`
  - [x] **nearly-sorted**: EXACT comment then sorted array + `static_cast<int>(NELEM_TIM * 0.05)` adjacent swaps with `mt19937(42)` — see reference implementation; comment wording is AC2-tested
  - [x] **pipe-organ**: ascending first half `[0..half-1]`, descending second half `[half-1..0]`, `half = NELEM_TIM / 2`
  - [x] Print closing separator row
  - [x] See reference implementation below

- [x] Task 5: Add `RunTimsortBenchmark()` call to `main()` (AC: 1, 4)
  - [x] Add `RunTimsortBenchmark();` at the BEGINNING of `main()`, BEFORE the existing `cout << "\n\n"; cout << "****...` header lines
  - [x] Reason: the new section self-generates all data (no `input.bin` dependency); calling it first ensures the timsort table is always produced even when `input.bin` is absent for the existing section

- [x] Task 6: Build and verify (AC: 1, 2, 3, 4)
  - [x] Compile with: see Dev Notes compilation command; expect zero warnings from new code
  - [x] Run the binary; verify the timsort table is printed before the existing benchmark header
  - [x] Confirm all 5 rows appear with 4 timing columns each, no crash
  - [x] Confirm the nearly-sorted comment reads exactly as specified
  - [x] Confirm original `Test()` and its callers are unchanged (no changes to lines outside the new code)

### Review Findings

- [x] [Review][Defer] Single-run benchmark, no warm-up or multi-iteration averaging [benchmark_numbers.cpp] — deferred, pre-existing; matches existing `Test()` benchmark design; spec mandates this single-shot pattern
- [x] [Review][Defer] rng() signed/unsigned mismatch and modulo bias in random/nearly-sorted data generation [benchmark_numbers.cpp] — deferred, pre-existing; matches existing `Generator_*` file pattern; spec-mandated reference implementation uses same idiom
- [x] [Review][Defer] uint32_t loop index compared against V.size() (size_t) [benchmark_numbers.cpp] — deferred, pre-existing; directly copied from existing `Test()` function
- [x] [Review][Defer] Fixed algorithm ordering in TestTimsort: timsort always runs first, giving it a cold-cache disadvantage vs. later algorithms [benchmark_numbers.cpp] — deferred, pre-existing benchmark design; same pattern as existing `Test()`; address in a future benchmark-quality epic

## Dev Notes

### Critical: Existing Code is Off-Limits

The following are **read-only** — no modifications of any kind:
- `Test()` function (lines ~261–322) — times uint64_t vectors with 6 algorithms
- All `Generator_*` functions (lines ~131–260)
- The existing `main()` benchmark section (lines ~59–129) — the 6-column table header and generator calls

The ONLY changes to the existing file structure are:
1. One new `using bsort::timsort;` line in the using block
2. One `const int NELEM_TIM = 1000000;` constant
3. Two new forward declarations
4. Three new function definitions: `TestTimsort()`, `RunTimsortBenchmark()`
5. One new call `RunTimsortBenchmark();` at the start of `main()`

### What `TestTimsort` Is NOT

- **Not a new binary** — no new `main()`, no new CMakeLists entry
- **Not modifying `Test()`** — `Test()` operates on `uint64_t`; `TestTimsort` operates on `int`
- **Not using `spreadsort`** — only 4 algorithms: timsort, spinsort, flat_stable_sort, stable_sort

### `timsort` is Already Available

`boost/sort/sort.hpp` (included at line ~30) already pulls in `timsort.hpp` (added in Story 1.4). The `using bsort::timsort;` declaration is the only new plumbing needed.

### Nearly-Sorted: Exact Comment Text (AC2)

AC2 is machine-checked — the comment must be byte-for-byte identical:
```cpp
// Nearly-sorted: sorted array, N*0.05 random adjacent swaps, mt19937(42)
```
Any deviation (extra space, different wording, wrong seed) fails AC2.

### Swap Count Arithmetic

`static_cast<int>(NELEM_TIM * 0.05)` = `static_cast<int>(1000000 * 0.05)` = `50000`.  
The swap index is `static_cast<int>(rng() % (NELEM_TIM - 1))`, swapping `A[i]` and `A[i+1]`.  
`std::swap(A[i], A[i+1])` — standard C++11.

### Compilation Commands

```bash
# Detect Boost include path
BOOST_INC=$(ls -d /opt/homebrew/include /usr/local/include 2>/dev/null | head -1) || BOOST_INC=/usr/local/include

# Compile (expect zero warnings from new code; pre-existing warnings in boost headers are OK)
g++ -std=c++11 -Wall -Wextra -Wpedantic \
    -I libs/sort/include \
    -I "$BOOST_INC" \
    libs/sort/benchmark/single/benchmark_numbers.cpp \
    -o /tmp/benchmark_numbers_bin

# Run — the timsort table appears first; the existing section requires input.bin
# (no input.bin = existing generators print error and exit; that is pre-existing behavior)
/tmp/benchmark_numbers_bin
```

Expected output head:
```
************************************************************
**   Timsort vs. Stable-Sort Algorithms (N=1M, int)    ...
...
random              |  X.XX |  X.XX |  X.XX |  X.XX |
already-sorted      |  X.XX |  X.XX |  X.XX |  X.XX |
reverse-sorted      |  X.XX |  X.XX |  X.XX |  X.XX |
nearly-sorted       |  X.XX |  X.XX |  X.XX |  X.XX |
pipe-organ          |  X.XX |  X.XX |  X.XX |  X.XX |
```

### Reference Implementation: `TestTimsort()`

```cpp
void TestTimsort(const std::vector<int>& B)
{
    std::less<int> comp;
    double duration;
    time_point start, finish;
    std::vector<int> A;
    std::vector<double> V;

    A = B;
    start = now();
    timsort(A.begin(), A.end(), comp);
    finish = now();
    duration = subtract_time(finish, start);
    V.push_back(duration);

    A = B;
    start = now();
    spinsort(A.begin(), A.end(), comp);
    finish = now();
    duration = subtract_time(finish, start);
    V.push_back(duration);

    A = B;
    start = now();
    flat_stable_sort(A.begin(), A.end(), comp);
    finish = now();
    duration = subtract_time(finish, start);
    V.push_back(duration);

    A = B;
    start = now();
    stable_sort(A.begin(), A.end(), comp);
    finish = now();
    duration = subtract_time(finish, start);
    V.push_back(duration);

    cout << std::setprecision(2) << std::fixed;
    for (uint32_t i = 0; i < V.size(); ++i)
        cout << std::right << std::setw(5) << V[i] << " |";
    cout << endl;
}
```

Note: `stable_sort` (not `std::stable_sort`) — the file has `using namespace std;` at the top.

### Reference Implementation: `RunTimsortBenchmark()`

```cpp
void RunTimsortBenchmark()
{
    cout << "\n";
    cout << "************************************************************\n";
    cout << "**                                                        **\n";
    cout << "**   Timsort vs. Stable-Sort Algorithms (N=1M, int)       **\n";
    cout << "**   timsort | spinsort | flat_stable_sort | stable_sort  **\n";
    cout << "**                                                        **\n";
    cout << "************************************************************\n";
    cout << endl;
    cout << "[ 1 ] timsort  [ 2 ] spinsort  [ 3 ] flat_stable_sort  [ 4 ] std::stable_sort\n\n";
    cout << "                    |      |      |      |      |\n";
    cout << "                    | [ 1 ]| [ 2 ]| [ 3 ]| [ 4 ]|\n";
    cout << "--------------------+------+------+------+------+\n";

    // random
    {
        vector<int> A;
        A.reserve(NELEM_TIM);
        std::mt19937 rng(123);
        for (int i = 0; i < NELEM_TIM; ++i)
            A.push_back(static_cast<int>(rng() % NELEM_TIM));
        cout << "random              |";
        TestTimsort(A);
    }

    // already-sorted
    {
        vector<int> A;
        A.reserve(NELEM_TIM);
        for (int i = 0; i < NELEM_TIM; ++i)
            A.push_back(i);
        cout << "already-sorted      |";
        TestTimsort(A);
    }

    // reverse-sorted
    {
        vector<int> A;
        A.reserve(NELEM_TIM);
        for (int i = NELEM_TIM; i > 0; --i)
            A.push_back(i);
        cout << "reverse-sorted      |";
        TestTimsort(A);
    }

    // Nearly-sorted: sorted array, N*0.05 random adjacent swaps, mt19937(42)
    {
        vector<int> A;
        A.reserve(NELEM_TIM);
        for (int i = 0; i < NELEM_TIM; ++i)
            A.push_back(i);
        std::mt19937 rng(42);
        const int nswaps = static_cast<int>(NELEM_TIM * 0.05);
        for (int s = 0; s < nswaps; ++s) {
            int idx = static_cast<int>(rng() % (NELEM_TIM - 1));
            std::swap(A[idx], A[idx + 1]);
        }
        cout << "nearly-sorted       |";
        TestTimsort(A);
    }

    // pipe-organ: ascending first half, descending second half
    {
        vector<int> A;
        A.reserve(NELEM_TIM);
        const int half = NELEM_TIM / 2;
        for (int i = 0; i < half; ++i)
            A.push_back(i);
        for (int i = half - 1; i >= 0; --i)
            A.push_back(i);
        cout << "pipe-organ          |";
        TestTimsort(A);
    }

    cout << "--------------------+------+------+------+------+\n";
    cout << endl;
}
```

### Reference: Updated `main()` Head

```cpp
int main(int argc, char *argv[])
{
    RunTimsortBenchmark();   // ← ADD THIS LINE (before everything else in main)

    cout << "\n\n";
    cout << "************************************************************\n";
    // ... rest of existing main unchanged ...
```

### C++11 Compliance Checklist

- `std::mt19937` — C++11 (`<random>` already included at line ~22) ✓
- `static_cast<int>(...)` — C++11 ✓
- Range-based for loops NOT used (existing file style: manual loops) — follow existing style ✓
- No `auto` in function body — existing file style avoids `auto`; follow suit ✓
- No lambdas — consistent with existing code style ✓
- `using bsort::timsort;` — C++11 ✓

### Anti-Patterns to Avoid

- **DO NOT** rename or remove `spreadsort` from the existing `using` block
- **DO NOT** add a `using bsort::timsort;` inside `RunTimsortBenchmark()` — it goes at file scope in the using block
- **DO NOT** create a new `main()` function or separate binary
- **DO NOT** modify `Test()` to add a timsort column — `Test()` is for uint64_t; the new `TestTimsort()` is for int
- **DO NOT** call `std::stable_sort` by its qualified name — `using namespace std;` is already at line 34; just use `stable_sort`
- **DO NOT** use `buffer.resize()` anywhere — not applicable here but consistency with project-wide rule
- **DO NOT** depend on `input.bin` in new generator code — new code is fully self-contained

### Project Structure Notes

- **Only file modified**: `libs/sort/benchmark/single/benchmark_numbers.cpp`
- **No new files created**
- **No build system changes** (benchmark already has its own build target; we're extending the same binary)
- All changes confined to the `libs/sort/` submodule

### Previous Story Learnings (from Stories 2.1, 2.2)

- Boost framework headers produce ~48 warnings under `-Wpedantic`; these are pre-existing, non-blocking, and come from Boost headers not our code. Only zero warnings from OUR new code is required.
- The existing file already has `using namespace std;` — do NOT add another one.
- `mt19937` needs `<random>` which is already included at line ~22.
- `std::swap` is in `<algorithm>` which is included at line ~18.

### References

- [Source: epics.md#Story-3.1] — Acceptance criteria, FR-14, FR-15, P7, D4.3
- [Source: architecture.md#D4.2] — Benchmark scope: 4-way (timsort, spinsort, flat_stable_sort, std::stable_sort)
- [Source: architecture.md#D4.3] — Nearly-sorted: sorted array + N×0.05 adjacent swaps, mt19937(42)
- [Source: architecture.md#P7] — Benchmark column format; exact comment text above nearly-sorted case
- [Source: architecture.md#NFR-1] — C++11 minimum; `-std=c++11` mandatory
- [Source: libs/sort/benchmark/single/benchmark_numbers.cpp] — Existing file to extend; preserve Test() and all Generator_* functions verbatim

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6 (create-story context engine)

### Debug Log References

_None_

### Completion Notes List

- Added `const int NELEM_TIM = 1000000;` after `#define NELEM` (AC1)
- Added `using bsort::timsort;` after `using bsort::pdqsort;` (AC1, AC4)
- Added forward declarations `RunTimsortBenchmark()` and `TestTimsort()` before `main()` (AC1)
- Added `RunTimsortBenchmark();` call at start of `main()` (AC1, AC4)
- Implemented `TestTimsort()` timing 4 algorithms (timsort, spinsort, flat_stable_sort, stable_sort) using `std::less<int>` (AC1)
- Implemented `RunTimsortBenchmark()` with all 5 data shapes (random, already-sorted, reverse-sorted, nearly-sorted, pipe-organ) (AC1, AC2, AC3)
- AC2: Nearly-sorted comment byte-exact: `// Nearly-sorted: sorted array, N*0.05 random adjacent swaps, mt19937(42)` (AC2)
- AC3: Pipe-organ uses `half = NELEM_TIM / 2`, ascending [0..half-1] then descending [half-1..0] (AC3)
- AC4: All existing `Test()`, `Generator_*`, and `main()` original section unchanged (AC4)
- AC5: SM-1 not met (timsort=0.36s vs spinsort=0.19s on nearly-sorted); added required comment (AC5)
- Compiled with `-std=c++11 -Wall -Wextra -Wpedantic`; zero warnings from new code; ran binary confirming 5-row 4-column table (AC1, AC2, AC3, AC4)

### File List

- libs/sort/benchmark/single/benchmark_numbers.cpp
