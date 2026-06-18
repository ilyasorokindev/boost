---
baseline_commit: "1b63a5ca0f7efe23fedb352cf639ed59c4ff0874"
---

# Story 4.1: README Algorithm Table Entry

Status: done

## Story

As a C++ developer browsing the boost::sort library,
I want a timsort row in the single-thread algorithm comparison table in `libs/sort/README.md`,
so that I can immediately see timsort's key properties alongside the other algorithms without having to read the source.

## Acceptance Criteria

1. **timsort row present (FR-17):** `libs/sort/README.md` single-thread algorithm table contains a `timsort` row with all of:
   - Stable = `yes`
   - Additional memory = `N / 2`
   - Best, average, and worst case = `N, N LogN, N LogN`
   - Comparison method = `Comparison operator`

2. **Existing rows untouched:** The rows for `spreadsort`, `pdqsort`, `spinsort`, and `flat_stable_sort` are not modified, removed, or reordered.

3. **Table renders correctly:** The new row uses the same markdown column widths as existing rows so GitHub renders the table without breaking alignment.

## Tasks / Subtasks

- [x] Task 1: Add timsort row to single-thread algorithm table (AC: 1, 2, 3)
  - [x] Open `libs/sort/README.md`
  - [x] Locate the single-thread algorithm table (lines 16–22 in the current file); the table ends with the `flat_stable_sort` row
  - [x] Insert the following row **after** the `flat_stable_sort` row and **before** the closing blank line of the table:
    ```
      | timsort           |  yes  |      N / 2                 | N, N LogN, N LogN             | Comparison operator |
    ```
  - [x] Verify existing rows are byte-for-byte unchanged
  - [x] View the final table to confirm alignment looks correct

## Dev Notes

### File to Modify

- **`libs/sort/README.md`** — single file, single-table edit; no other file is touched by this story.

### Current Table State (as of last git commit)

```markdown
  | Algorithm         |Stable |   Additional memory        |Best, average, and worst case  | Comparison method   |
  |-------------------|-------|----------------------------|-------------------------------|---------------------|
  | spreadsort        |  no   |      key_length            | N, N sqrt(LogN),              | Hybrid radix sort   |
  |                   |       |                            | min(N logN, N key_length)     |                     |
  | pdqsort           |  no   |      Log N                 | N, N LogN, N LogN             | Comparison operator |
  | spinsort          |  yes  |      N / 2                 | N, N LogN, N LogN             | Comparison operator |
  | flat_stable_sort  |  yes  |size of the data / 256 + 8K | N, N LogN, N LogN             | Comparison operator |
```

The `spreadsort` row spans **two markdown lines** (the algorithm name + wrapped complexity cell). Do not collapse or reformat these two rows.

### Exact Row to Insert

```
  | timsort           |  yes  |      N / 2                 | N, N LogN, N LogN             | Comparison operator |
```

Column alignment notes (match existing row widths exactly):
- `Algorithm` column: 17 chars padded with trailing spaces to match `flat_stable_sort  ` width
- `Stable` column: `  yes  ` (two spaces each side)
- `Additional memory` column: `      N / 2                 ` — 6 leading spaces, `N / 2` then trailing spaces to width 26
- `Best, average, and worst case` column: `N, N LogN, N LogN             ` — trailing spaces to width 31
- `Comparison method` column: `Comparison operator ` — trailing space to width 19

The exact row string above already has correct whitespace; copy it verbatim.

### Insertion Point

Insert **after** line ending with `| Comparison operator |` that belongs to the `flat_stable_sort` row. There is a blank line after the table — insert before it, not after.

### Anti-Patterns to Avoid

- **Do not add** a prose description bullet for timsort below the table (that's Epic 4.2's scope).
- **Do not add** a `std::stable_sort` row — it does not appear in the current single-thread table.
- **Do not modify** the parallel algorithms table.
- **Do not change** any header or separator rows.

### No Build System Changes

This story is pure markdown. No CMakeLists.txt, Jamfile, or `.cpp` changes.

### Testing / Verification

There are no automated tests for README changes. Verification is:

1. `grep -n "timsort" libs/sort/README.md` — must return exactly one line.
2. Visual inspection of the table in the file to confirm alignment.
3. Confirm existing rows are unchanged: `git diff libs/sort/README.md` should show only the one new `timsort` row added.

### Previous Story Context

Stories 1–3 are complete. The timsort header is at `libs/sort/include/boost/sort/timsort/timsort.hpp` and is included via `libs/sort/include/boost/sort/sort.hpp`. The header-file top-of-file comment (FR-18) was added in Story 1.4 — this story only adds the README table row.

### References

- FR-17 definition: [epics.md — Story 4.1 AC](../_bmad-output/planning-artifacts/epics.md)
- Current README: `libs/sort/README.md` lines 13–22
- project-context.md §"What's Next — Epic 4 — Story 4.1"

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

- Added timsort row to single-thread algorithm table in `libs/sort/README.md` after the `flat_stable_sort` row. One line inserted, all existing rows byte-for-byte unchanged. `grep -n "timsort" README.md` returns exactly one match. `git diff` inside submodule confirms only the single row addition.

### File List

- libs/sort/README.md

### Review Findings

- [x] [Review][Defer] Missing prose description bullet for timsort [README.md] — deferred, out of scope for Story 4.1 per spec anti-patterns; belongs in a future documentation story
- [x] [Review][Defer] Missing timsort entry in `doc/single_thread.qbk` algorithm table [doc/single_thread.qbk] — deferred, .qbk files are Boost rendered docs; not in Story 4.1 scope
- [x] [Review][Defer] Missing timsort entry in `doc/introduction.qbk` algorithm table [doc/introduction.qbk] — deferred, same as above
- [x] [Review][Defer] timsort author/copyright omitted from README footer — deferred, not in Story 4.1 spec
- [x] [Review][Defer] Missing algorithm description link and paper reference for timsort [README.md] — deferred, belongs with prose description in a future story

### Change Log

- 2026-06-18: Added timsort row to README single-thread algorithm table (AC: 1, 2, 3 — FR-17)
- 2026-06-18: Code review complete — 0 patches, 5 deferred, 3 dismissed; status → done
