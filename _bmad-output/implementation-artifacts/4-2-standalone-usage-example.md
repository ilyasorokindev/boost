---
baseline_commit: "59277272dc5516838a687db78f4b87f428381dc4"
---

# Story 4.2: Standalone Usage Example

Status: done

## Story

As a C++ developer new to boost::sort::timsort,
I want a compilable usage example at `libs/sort/example/timsort_example.cpp`,
so that I can orient myself with the API in under a minute and immediately reproduce a working sort call.

## Acceptance Criteria

1. **Compiles clean (FR-19, NFR-1, NFR-3):** `libs/sort/example/timsort_example.cpp` exists, compiles
   with `-std=c++11 -Wall -Wextra -Wpedantic` producing zero warnings, and runs to completion.

2. **Default overload demonstrated:** The example calls `boost::sort::timsort(first, last)` on a
   `std::vector`, and prints the sorted result to stdout.

3. **Custom comparator overload demonstrated:** The example calls `boost::sort::timsort(first, last, comp)`
   with at least one explicit comparator (e.g., `std::greater<int>()` or a lambda).

4. **Minimal includes:** The `#include` list contains only `<boost/sort/sort.hpp>` (or
   `<boost/sort/timsort/timsort.hpp>`), `<vector>`, and `<iostream>` — no additional Boost or
   external headers.

5. **Substantive code:** The example body is 5–10 lines of substantive code (excluding blank lines
   and comment lines).

## Tasks / Subtasks

- [x] Task 1: Create `libs/sort/example/timsort_example.cpp` (AC: 1, 2, 3, 4, 5)
  - [x] Write the file with the Boost license header comment
  - [x] Include only `<boost/sort/timsort/timsort.hpp>`, `<vector>`, `<iostream>`
  - [x] Demonstrate `timsort(first, last)` on a `std::vector<int>` and print sorted output
  - [x] Demonstrate `timsort(first, last, comp)` with a custom comparator and print output
  - [x] Verify substantive code is 5–10 lines (blank lines / comments excluded)

- [x] Task 2: Register the example in `libs/sort/example/Jamfile.v2` (build system)
  - [x] Add `exe timsort_example : timsort_example.cpp ;` to `Jamfile.v2` after the last `exe` line
  - [x] Confirm the new line follows the same format as existing entries

- [x] Task 3: Compile and verify (AC: 1, 2, 3)
  - [x] Run: `g++ -std=c++11 -Wall -Wextra -Wpedantic -I libs/sort/include timsort_example.cpp -o /tmp/timsort_example`
    from the superproject root (or equivalent path to the source and include)
  - [x] Confirm zero warnings and zero errors
  - [x] Run `/tmp/timsort_example` and confirm sorted output appears on stdout

## Dev Notes

### File to Create

- **`libs/sort/example/timsort_example.cpp`** — the only source file created by this story.

### File to Modify

- **`libs/sort/example/Jamfile.v2`** — add one `exe` line. This is necessary because the example
  Jamfile uses **explicit per-target registration** (no wildcard); unlike the test Jamfile, new
  files are NOT picked up automatically. See Jamfile.v2 current structure below.

### No CMakeLists.txt Change Needed

The CMake build system for examples is not wired into the main build (`BOOSTINCLUDE_LIBRARIES=sort`
only builds the test suite). There is no `example/CMakeLists.txt`. Jamfile.v2 is the only build
registration needed.

### Current Jamfile.v2 State (as of Story 4.1 commit)

```jamfile
exe spreadsort : sample.cpp ;
exe alreadysorted : alreadysorted.cpp ;
exe mostlysorted : mostlysorted.cpp ;
exe rightshift : rightshiftsample.cpp ;
exe reverseintsort : reverseintsample.cpp ;
exe int64 : int64.cpp ;
exe floatsort : floatsample.cpp ;
exe shiftfloatsort : shiftfloatsample.cpp ;
exe floatfunctorsort : floatfunctorsample.cpp ;
exe double : double.cpp ;
exe stringsort : stringsample.cpp ;
exe wstringsort : wstringsample.cpp ;
exe reversestringsort : reversestringsample.cpp ;
exe charstringsort : charstringsample.cpp ;
exe stringfunctorsort : stringfunctorsample.cpp ;
exe reversestringfunctorsort : reversestringfunctorsample.cpp ;
exe keyplusdata : keyplusdatasample.cpp ;
exe randomgen : randomgen.cpp ;
exe boostrandomgen : boostrandomgen.cpp ;
exe alrbreaker : alrbreaker.cpp ;
exe binaryalrbreaker : binaryalrbreaker.cpp ;
exe caseinsensitive : caseinsensitive.cpp ;
exe generalizedstruct : generalizedstruct.cpp ;
```

Add after the last `exe` line:
```jamfile
exe timsort_example : timsort_example.cpp ;
```

### Target File Template

The example must be concise and self-explanatory. Suggested structure (adapt as needed but
**stay within 5–10 substantive lines**):

```cpp
// boost::sort::timsort usage example
//
// Copyright (c) 2026 Ilya Sorokin
// Distributed under the Boost Software License, Version 1.0.
//     (See accompanying file LICENSE_1_0.txt or copy at
//      http://www.boost.org/LICENSE_1_0.txt)

#include <boost/sort/sort.hpp>
#include <iostream>
#include <vector>

int main() {
    // Default overload: sort ascending using operator<
    std::vector<int> v = {5, 3, 1, 4, 2};
    boost::sort::timsort(v.begin(), v.end());
    for (int x : v) std::cout << x << ' ';
    std::cout << '\n';

    // Custom comparator overload: sort descending
    std::vector<int> w = {5, 3, 1, 4, 2};
    boost::sort::timsort(w.begin(), w.end(), std::greater<int>());
    for (int x : w) std::cout << x << ' ';
    std::cout << '\n';

    return 0;
}
```

The substantive lines (excluding blank lines, comment block, and `return 0`) count as:
1. `std::vector<int> v = {...};`
2. `boost::sort::timsort(v.begin(), v.end());`
3. `for (int x : v) std::cout << x << ' ';`
4. `std::cout << '\n';`
5. `std::vector<int> w = {...};`
6. `boost::sort::timsort(w.begin(), w.end(), std::greater<int>());`
7. `for (int x : w) std::cout << x << ' ';`
8. `std::cout << '\n';`

= 8 substantive lines — within the 5–10 bound.

### Compile Verification Command

Run from the superproject root:

```bash
g++ -std=c++11 -Wall -Wextra -Wpedantic \
    -I libs/sort/include \
    libs/sort/example/timsort_example.cpp \
    -o /tmp/timsort_example
/tmp/timsort_example
```

Expected output:
```
1 2 3 4 5
5 4 3 2 1
```

### C++11 Compatibility Notes

- Use `std::greater<int>()` NOT `std::greater<>()` (transparent comparator requires C++14).
- Use range-for loop with explicit type `int x` NOT `auto x` — both are fine in C++11, but explicit
  is clearer and avoids any edge-case warnings on older compilers.
- Do NOT use initializer-list brace syntax for vectors if compiler warns — `{5, 3, 1, 4, 2}` is
  valid C++11, confirmed in use throughout the existing test suite.

### Previous Story Context

Stories 1–4.1 are complete:
- `libs/sort/include/boost/sort/timsort/timsort.hpp` — full implementation
- `libs/sort/include/boost/sort/sort.hpp` — includes timsort.hpp via one added `#include` line
- `libs/sort/README.md` — timsort row added to algorithm table

This story only adds `timsort_example.cpp` and registers it in Jamfile.v2. No changes to the
timsort.hpp header, sort.hpp, or test files.

### Anti-Patterns to Avoid

- **Do not** `#include <boost/sort/timsort/timsort.hpp>` directly — use `<boost/sort/sort.hpp>`
  (the cumulative header) to demonstrate the public integration point (AC: 4).
  If the cumulative header include path causes any compiler issue, fall back to the direct header
  and note why in the Completion Notes.
- **Do not** add more than 10 substantive code lines — this is a minimal orientation example, not
  a tutorial (AC: 5).
- **Do not** use `using namespace boost::sort;` — call `boost::sort::timsort` fully qualified
  (consistent with architecture P2 guideline: no `using namespace` in headers; example code should
  model best practice).
- **Do not** touch any other file beyond `timsort_example.cpp` and `Jamfile.v2`.

### References

- FR-19: `example/timsort_example.cpp` usage example
- NFR-1: C++11 minimum
- NFR-3: Zero warnings under `-std=c++11 -Wall -Wextra -Wpedantic`
- Architecture D2.2: default comparator is `std::less<value_type>()` — example verifies both overloads
- project-context.md §"What's Next — Epic 4 — Story 4.2"
- Epic 4.2 AC source: `_bmad-output/planning-artifacts/epics.md`

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

- Used `<boost/sort/timsort/timsort.hpp>` (direct header) instead of `<boost/sort/sort.hpp>` (cumulative).
  Reason: `sort.hpp` transitively pulls in `spreadsort/spreadsort.hpp` which triggers 7 deprecation
  warnings from system Boost type_traits headers under `-Wpedantic` on this compiler (clang on macOS).
  These warnings are in Boost internals, not in our code. The direct header produces zero warnings.
  AC 4 explicitly permits both options.

### Completion Notes List

- Created `libs/sort/example/timsort_example.cpp` (25 lines total: license header + 2 blank lines +
  `#include` block + `main()` with 8 substantive lines). Demonstrates both overloads.
- Added `exe timsort_example : timsort_example.cpp ;` to `libs/sort/example/Jamfile.v2`.
- Compiled with `g++ -std=c++11 -Wall -Wextra -Wpedantic -I libs/sort/include` — zero warnings, zero errors.
- Output: `1 2 3 4 5 ` (ascending) and `5 4 3 2 1 ` (descending). All ACs satisfied.

### File List

- libs/sort/example/timsort_example.cpp
- libs/sort/example/Jamfile.v2

### Change Log

- 2026-06-18: Story created — ready-for-dev
- 2026-06-18: Implemented — created timsort_example.cpp and registered in Jamfile.v2; compiled zero-warning, ran correctly; status → review
- 2026-06-18: Code review — 1 finding (4-space vs 2-space indent); fixed; status → done
