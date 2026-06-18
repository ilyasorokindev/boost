---
baseline_commit: "b67b047b2620b3c6cec12186a2754b983cdd4728"
---

# Story 3.2: Extend `benchmark_strings.cpp` with Timsort Column

Status: done

## Story

As a C++ developer evaluating sorting algorithms,
I want timsort included as a benchmark column in `benchmark/single/benchmark_strings.cpp` alongside spinsort, flat_stable_sort, and std::stable_sort across five data shapes for `std::string`,
so that I can verify timsort's performance on comparison-expensive element types and confirm the 4-column × 5-shape output required by FR-16.

## Acceptance Criteria

1. **4 × 5 benchmark table (FR-14, FR-15, FR-16):** `benchmark/single/benchmark_strings.cpp` is modified to add a timsort comparison section; when compiled with `-std=c++11` and run, it produces a results table with exactly 4 algorithm columns (timsort, spinsort, flat_stable_sort, std::stable_sort) × 5 data shape rows (random, already-sorted, reverse-sorted, nearly-sorted, pipe-organ) for `std::string`; no crash on any shape.

2. **Nearly-sorted comment (P7, D4.3):** The comment immediately above the nearly-sorted timing block reads exactly:
   ```
   // Nearly-sorted: sorted array, N*0.05 random adjacent swaps, mt19937(42)
   ```
   The code uses `std::mt19937` seeded with `42` and performs exactly `static_cast<int>(NELEM_STR * 0.05)` random adjacent-pair swaps (swap A[i] and A[i+1] for random i in [0, NELEM_STR-2]).

3. **Pipe-organ shape (FR-14):** The pipe-organ generator produces an ascending first half followed by a descending second half (same strings reversed), where `half = NELEM_STR / 2`; all 4 algorithms produce timing output without crash.

4. **Existing table untouched (FR-14 preservation):** The original `Test()` function (lines 284–345), all existing `Generator_*` functions (lines 136–282), and the original `main()` benchmark section (lines 62–133 — 6-column std::string table) are unchanged; only the new timsort comparison section and `RunTimsortStringBenchmark()` call are new. No existing symbol renamed or removed.

5. **Self-contained generation:** The new benchmark section generates all string data programmatically without reading `input.bin`; it works correctly even when `input.bin` is absent.

## Tasks / Subtasks

- [x] Task 1: Add `using bsort::timsort;` alias (AC: 1, 4)
  - [x] Open `libs/sort/benchmark/single/benchmark_strings.cpp`
  - [x] After `using bsort::pdqsort;` (line 47), add exactly: `using bsort::timsort;`
  - [x] Do NOT add any other new `using` declarations

- [x] Task 2: Add `NELEM_STR` constant and forward declarations (AC: 1, 5)
  - [x] After `#define NMAXSTRING 10000000` (line 32), add: `const int NELEM_STR = 100000;`
  - [x] Before `main()` (before line 57), add forward declarations:
    ```cpp
    void RunTimsortStringBenchmark();
    void TestTimsortStr(const std::vector<std::string>& B);
    ```

- [x] Task 3: Implement `TestTimsortStr()` with 4-column output (AC: 1, 4)
  - [x] Add the function AFTER the existing `Test()` function (after line 345)
  - [x] Time 4 algorithms in this exact order: timsort, spinsort, flat_stable_sort, stable_sort
  - [x] Use `std::less<std::string> comp;`
  - [x] Follow the same `time_point start/finish; subtract_time; V.push_back` pattern as the existing `Test()`
  - [x] Print with same `setprecision(2) fixed setw(5) right` format and trailing `|`
  - [x] See reference implementation below

- [x] Task 4: Implement `RunTimsortStringBenchmark()` with 5 data shapes (AC: 1, 2, 3, 5)
  - [x] Add after `TestTimsortStr()`
  - [x] Print a self-contained header block identifying the 4 algorithms by number
  - [x] Print a 4-column table header and separator
  - [x] **random**: generate N=NELEM_STR random 8-char lowercase strings with `std::mt19937 rng(123)` and `charset[rng() % 26]`
  - [x] **already-sorted**: same generation (rng(123)) + `std::sort(A.begin(), A.end())`
  - [x] **reverse-sorted**: same generation (rng(123)) + sort + `std::reverse(A.begin(), A.end())`
  - [x] **nearly-sorted**: same generation (rng(123)) + sort + `static_cast<int>(NELEM_STR * 0.05)` adjacent swaps with `mt19937(42)` — EXACT comment required (AC2); use separate `rng2(42)` for swaps
  - [x] **pipe-organ**: generate NELEM_STR strings (rng(123)) + sort into `sorted_base`; first half of output = `sorted_base[0..half-1]`; second half = `sorted_base[half-1..0]`; `half = NELEM_STR / 2`
  - [x] Print closing separator row
  - [x] See reference implementation below

- [x] Task 5: Add `RunTimsortStringBenchmark()` call to `main()` (AC: 1, 4)
  - [x] Add `RunTimsortStringBenchmark();` at the BEGINNING of `main()`, BEFORE the existing `cout << "\n\n"; ...` header lines (before line 62 content)
  - [x] Reason: new section is fully self-contained; calling it first ensures the string timsort table is produced even when `input.bin` is absent

- [x] Task 6: Build and verify (AC: 1, 2, 3, 4, 5)
  - [x] Compile with the command in Dev Notes; expect zero warnings from new code
  - [x] Run the binary; verify the timsort string table is printed before the existing benchmark header
  - [x] Confirm all 5 rows appear with 4 timing columns each, no crash
  - [x] Confirm the nearly-sorted comment reads exactly as specified in AC2
  - [x] Confirm original `Test()` and its callers are unchanged (no changes to lines outside the new code)

### Review Findings

- [x] [Review][Defer] Single-run benchmark, no warm-up or multi-iteration averaging [benchmark_strings.cpp] — deferred, pre-existing; matches existing `Test()` benchmark design and Story 3.1 pattern
- [x] [Review][Defer] rng() signed/unsigned mismatch and modulo bias (int data and charset[rng()%26]) [benchmark_strings.cpp] — deferred, pre-existing; same class as Story 3.1 deferred finding; matches existing `Generator_*` pattern
- [x] [Review][Defer] uint32_t loop index compared against V.size() (size_t) [benchmark_strings.cpp] — deferred, pre-existing; directly copied from existing `Test()` function idiom
- [x] [Review][Defer] Fixed algorithm ordering in TestTimsortStr: timsort always runs first, giving it a cold-cache disadvantage vs. later algorithms [benchmark_strings.cpp] — deferred, pre-existing benchmark design; same pattern as existing `Test()` and Story 3.1; address in a future benchmark-quality epic

## Dev Notes

### Critical: Existing Code is Off-Limits

The following are **read-only** — no modifications of any kind:
- `Test()` function (lines 284–345) — times 6 algorithms (std::sort, pdqsort, std::stable_sort, spinsort, flat_stable_sort, spreadsort) on `std::vector<std::string>` loaded from `input.bin`
- All `Generator_*` functions (lines 136–282) — all depend on `input.bin`; do not touch
- The existing `main()` benchmark section (lines 62–133) — the 6-column table header and all Generator_* calls

The ONLY changes to the existing file structure are:
1. One new `using bsort::timsort;` line in the using block (after line 47)
2. One `const int NELEM_STR = 100000;` constant (after line 32)
3. Two new forward declarations (before `main()`)
4. Two new function definitions: `TestTimsortStr()` and `RunTimsortStringBenchmark()` (after existing `Test()`)
5. One new call `RunTimsortStringBenchmark();` at the start of `main()`

### What `TestTimsortStr` Is NOT

- **Not a new binary** — no new `main()`, no new CMakeLists entry
- **Not modifying `Test()`** — `Test()` benchmarks 6 algorithms on `input.bin`-loaded strings; `TestTimsortStr` benchmarks 4 algorithms on self-generated strings
- **Not using `spreadsort` or `pdqsort`** — only 4 algorithms: timsort, spinsort, flat_stable_sort, stable_sort

### `timsort` is Already Available

`boost/sort/sort.hpp` (line 29) already includes `timsort.hpp` (added in Story 1.4). The `using bsort::timsort;` declaration is the only new plumbing needed.

### Nearly-Sorted: Exact Comment Text (AC2)

AC2 is machine-checked — the comment must be byte-for-byte identical:
```cpp
// Nearly-sorted: sorted array, N*0.05 random adjacent swaps, mt19937(42)
```
Any deviation (extra space, different wording, wrong seed) fails AC2.

### String Generation Strategy

All five shapes are generated from `std::string` vectors — no `input.bin` required:
- **random**: NELEM_STR 8-char lowercase strings, `mt19937(123)`, `charset[rng() % 26]`
- **already-sorted**: same random strings → `std::sort`
- **reverse-sorted**: same random strings → `std::sort` → `std::reverse`
- **nearly-sorted**: same random strings → `std::sort` → `static_cast<int>(NELEM_STR * 0.05)` adjacent swaps using a separate `mt19937(42)`
- **pipe-organ**: same random strings → `std::sort` into `sorted_base`; output = `sorted_base[0..half-1]` + `sorted_base[half-1..0]`

The `charset` array is: `"abcdefghijklmnopqrstuvwxyz"` (26 chars).

### Swap Count for Nearly-Sorted

`static_cast<int>(NELEM_STR * 0.05)` = `static_cast<int>(100000 * 0.05)` = `5000`.
Swap index: `static_cast<int>(rng2() % (NELEM_STR - 1))`, swapping `A[idx]` and `A[idx+1]`.

### Required Headers (Already Present)

All needed headers are already included in the file:
- `<algorithm>` (line 18) — `std::sort`, `std::reverse`, `std::swap`
- `<random>` (line 21) — `std::mt19937`
- `<vector>` (line 23) — `std::vector`
- `<iomanip>` (line 20) — `std::setprecision`, `std::setw`, `std::fixed`, `std::right`

No new `#include` directives are needed.

### Compilation Commands

```bash
# Detect Boost include path
BOOST_INC=$(ls -d /opt/homebrew/include /usr/local/include 2>/dev/null | head -1) || BOOST_INC=/usr/local/include

# Compile (expect zero warnings from new code; pre-existing warnings in boost headers are OK)
g++ -std=c++11 -Wall -Wextra -Wpedantic \
    -I libs/sort/include \
    -I "$BOOST_INC" \
    libs/sort/benchmark/single/benchmark_strings.cpp \
    -o /tmp/benchmark_strings_bin

# Run — the timsort string table appears first; the existing section requires input.bin
# (no input.bin = existing generators print "Error in the input file" and exit; pre-existing behavior)
/tmp/benchmark_strings_bin
```

Expected output head:
```
************************************************************
**                                                        **
**  Timsort vs. Stable-Sort Algorithms (N=100K, string)   **
**   timsort | spinsort | flat_stable_sort | stable_sort  **
**                                                        **
************************************************************

[ 1 ] timsort  [ 2 ] spinsort  [ 3 ] flat_stable_sort  [ 4 ] std::stable_sort

                    |      |      |      |      |
                    | [ 1 ]| [ 2 ]| [ 3 ]| [ 4 ]|
--------------------+------+------+------+------+
random              |  X.XX |  X.XX |  X.XX |  X.XX |
already-sorted      |  X.XX |  X.XX |  X.XX |  X.XX |
reverse-sorted      |  X.XX |  X.XX |  X.XX |  X.XX |
nearly-sorted       |  X.XX |  X.XX |  X.XX |  X.XX |
pipe-organ          |  X.XX |  X.XX |  X.XX |  X.XX |
--------------------+------+------+------+------+
```

### Reference Implementation: `TestTimsortStr()`

```cpp
void TestTimsortStr(const std::vector<std::string>& B)
{
    std::less<std::string> comp;
    double duration;
    time_point start, finish;
    std::vector<std::string> A;
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

Note: `stable_sort` (not `std::stable_sort`) — the file has `using namespace std;`.

### Reference Implementation: `RunTimsortStringBenchmark()`

```cpp
void RunTimsortStringBenchmark()
{
    cout << "\n";
    cout << "************************************************************\n";
    cout << "**                                                        **\n";
    cout << "**  Timsort vs. Stable-Sort Algorithms (N=100K, string)   **\n";
    cout << "**   timsort | spinsort | flat_stable_sort | stable_sort  **\n";
    cout << "**                                                        **\n";
    cout << "************************************************************\n";
    cout << endl;
    cout << "[ 1 ] timsort  [ 2 ] spinsort  [ 3 ] flat_stable_sort  [ 4 ] std::stable_sort\n\n";
    cout << "                    |      |      |      |      |\n";
    cout << "                    | [ 1 ]| [ 2 ]| [ 3 ]| [ 4 ]|\n";
    cout << "--------------------+------+------+------+------+\n";

    const char charset[] = "abcdefghijklmnopqrstuvwxyz";

    // random
    {
        vector<std::string> A;
        A.reserve(NELEM_STR);
        std::mt19937 rng(123);
        for (int i = 0; i < NELEM_STR; ++i) {
            std::string s(8, ' ');
            for (int j = 0; j < 8; ++j)
                s[j] = charset[rng() % 26];
            A.push_back(s);
        }
        cout << "random              |";
        TestTimsortStr(A);
    }

    // already-sorted
    {
        vector<std::string> A;
        A.reserve(NELEM_STR);
        std::mt19937 rng(123);
        for (int i = 0; i < NELEM_STR; ++i) {
            std::string s(8, ' ');
            for (int j = 0; j < 8; ++j)
                s[j] = charset[rng() % 26];
            A.push_back(s);
        }
        std::sort(A.begin(), A.end());
        cout << "already-sorted      |";
        TestTimsortStr(A);
    }

    // reverse-sorted
    {
        vector<std::string> A;
        A.reserve(NELEM_STR);
        std::mt19937 rng(123);
        for (int i = 0; i < NELEM_STR; ++i) {
            std::string s(8, ' ');
            for (int j = 0; j < 8; ++j)
                s[j] = charset[rng() % 26];
            A.push_back(s);
        }
        std::sort(A.begin(), A.end());
        std::reverse(A.begin(), A.end());
        cout << "reverse-sorted      |";
        TestTimsortStr(A);
    }

    // Nearly-sorted: sorted array, N*0.05 random adjacent swaps, mt19937(42)
    {
        vector<std::string> A;
        A.reserve(NELEM_STR);
        std::mt19937 rng(123);
        for (int i = 0; i < NELEM_STR; ++i) {
            std::string s(8, ' ');
            for (int j = 0; j < 8; ++j)
                s[j] = charset[rng() % 26];
            A.push_back(s);
        }
        std::sort(A.begin(), A.end());
        std::mt19937 rng2(42);
        const int nswaps = static_cast<int>(NELEM_STR * 0.05);
        for (int k = 0; k < nswaps; ++k) {
            int idx = static_cast<int>(rng2() % (NELEM_STR - 1));
            std::swap(A[idx], A[idx + 1]);
        }
        cout << "nearly-sorted       |";
        TestTimsortStr(A);
    }

    // pipe-organ: ascending first half, descending second half
    {
        vector<std::string> sorted_base;
        sorted_base.reserve(NELEM_STR);
        std::mt19937 rng(123);
        for (int i = 0; i < NELEM_STR; ++i) {
            std::string s(8, ' ');
            for (int j = 0; j < 8; ++j)
                s[j] = charset[rng() % 26];
            sorted_base.push_back(s);
        }
        std::sort(sorted_base.begin(), sorted_base.end());
        const int half = NELEM_STR / 2;
        vector<std::string> A;
        A.reserve(NELEM_STR);
        for (int i = 0; i < half; ++i)
            A.push_back(sorted_base[i]);
        for (int i = half - 1; i >= 0; --i)
            A.push_back(sorted_base[i]);
        cout << "pipe-organ          |";
        TestTimsortStr(A);
    }

    cout << "--------------------+------+------+------+------+\n";
    cout << endl;
}
```

### Reference: Updated `main()` Head

```cpp
int main (int argc, char *argv [])
{
    RunTimsortStringBenchmark();   // ← ADD THIS LINE (before everything else in main)

    cout << "\n\n";
    cout << "************************************************************\n";
    // ... rest of existing main unchanged ...
```

### C++11 Compliance Checklist

- `std::mt19937` — C++11 (`<random>` at line 21) ✓
- `static_cast<int>(...)` — C++11 ✓
- `std::sort`, `std::reverse` — C++11 (`<algorithm>` at line 18) ✓
- `std::string(8, ' ')` + char assignment — C++11 ✓
- No `auto` in function body — consistent with existing file style ✓
- No lambdas — consistent with existing file style ✓
- `using bsort::timsort;` — C++11 ✓

### Anti-Patterns to Avoid

- **DO NOT** rename or remove `spreadsort`, `pdqsort`, or any existing `using` declaration
- **DO NOT** add `using bsort::timsort;` inside `RunTimsortStringBenchmark()` — it belongs at file scope
- **DO NOT** create a new `main()` function or separate binary
- **DO NOT** modify `Test()` — it already benchmarks 6 algorithms; `TestTimsortStr` is a separate function for 4 algorithms
- **DO NOT** use `buffer.resize()` anywhere
- **DO NOT** call `std::stable_sort` by qualified name — `using namespace std;` is already at line 34
- **DO NOT** depend on `input.bin` in new generator code — new code is fully self-contained
- **DO NOT** change the function name to `RunTimsortBenchmark` — that name is already used in `benchmark_numbers.cpp` (separate translation unit; use `RunTimsortStringBenchmark` for clarity)

### Project Structure Notes

- **Only file modified**: `libs/sort/benchmark/single/benchmark_strings.cpp`
- **No new files created**
- **No build system changes**
- All changes confined to the `libs/sort/` submodule

### Previous Story Learnings (from Stories 3.1, 2.1, 2.2)

- Boost framework headers produce ~48 warnings under `-Wpedantic`; these are pre-existing, non-blocking. Only zero warnings from OUR new code is required.
- The existing file already has `using namespace std;` (line 34) — do NOT add another one.
- `mt19937` needs `<random>` which is already included at line 21.
- `std::swap` is in `<algorithm>` which is already included at line 18.
- `std::reverse` is also in `<algorithm>` — already included.
- When using two `mt19937` instances in the same scope (base generation + swaps), use distinct variable names (`rng` and `rng2`) to avoid confusion.
- The `uint32_t` loop index over `V.size()` is a pre-existing idiom from `Test()`; carry it into `TestTimsortStr`.

### References

- [Source: epics.md#Story-3.2] — Acceptance criteria, FR-14, FR-15, FR-16
- [Source: architecture.md#D4.2] — Benchmark scope: 4-way (timsort, spinsort, flat_stable_sort, std::stable_sort)
- [Source: architecture.md#D4.3] — Nearly-sorted: sorted array + N×0.05 adjacent swaps, mt19937(42)
- [Source: architecture.md#P7] — Benchmark column format; exact comment text above nearly-sorted case
- [Source: architecture.md#NFR-1] — C++11 minimum; `-std=c++11` mandatory
- [Source: libs/sort/benchmark/single/benchmark_strings.cpp] — Existing file to extend; preserve Test() and all Generator_* functions verbatim
- [Source: Story 3.1 impl] — Pattern for TestTimsort/RunTimsortBenchmark; nearly-sorted comment text; pipe-organ shape definition

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6 (create-story context engine)

### Debug Log References

_None_

### Completion Notes List

- Added `const int NELEM_STR = 100000;` after `#define NMAXSTRING` (AC1, AC5)
- Added `using bsort::timsort;` after `using bsort::pdqsort;` (AC1, AC4)
- Added forward declarations `RunTimsortStringBenchmark()` and `TestTimsortStr()` before `main()` (AC1)
- Added `RunTimsortStringBenchmark();` call at start of `main()` (AC1, AC4)
- Implemented `TestTimsortStr()` timing 4 algorithms (timsort, spinsort, flat_stable_sort, stable_sort) using `std::less<std::string>` (AC1)
- Implemented `RunTimsortStringBenchmark()` with all 5 data shapes; all shapes generated from 8-char lowercase strings via mt19937(123) + charset (AC1, AC5)
- AC2: Nearly-sorted comment byte-exact: `// Nearly-sorted: sorted array, N*0.05 random adjacent swaps, mt19937(42)` (AC2)
- AC3: Pipe-organ uses `half = NELEM_STR / 2`, ascending `sorted_base[0..half-1]` then descending `sorted_base[half-1..0]` (AC3)
- AC4: All existing `Test()`, `Generator_*`, and `main()` original section unchanged (AC4)
- Compiled with `-std=c++11 -Wall -Wextra -Wpedantic`; zero warnings from new code; 9 pre-existing Boost header + unused-param warnings (AC1)
- Ran binary: 5-row 4-column timsort string table printed first; existing benchmark header follows (AC1, AC4)

### File List

- libs/sort/benchmark/single/benchmark_strings.cpp

## Change Log

- Extend benchmark_strings.cpp with timsort 4×5 string benchmark (Story 3.2, 2026-06-18)
