# Deferred Work Ledger

## Deferred from: code review of 4-1-readme-algorithm-table-entry (2026-06-18)

- Missing prose description bullet for timsort in README.md: no `- **timsort** is a ...` paragraph below the table; every other algorithm has one; belongs in a future documentation story (not Story 4.1 scope per spec anti-patterns)
- Missing timsort entry in `doc/single_thread.qbk`: the Boost rendered docs (not README) come from .qbk files; `doc/single_thread.qbk` contains an identical algorithm table that does not include timsort
- Missing timsort entry in `doc/introduction.qbk`: same as above; `doc/introduction.qbk` has a third copy of the single-thread table without timsort
- timsort author/copyright omitted from README footer: README lines 76–80 list copyright holders for spreadsort/spinsort/pdqsort; timsort implementation (`timsort.hpp` line 11) carries "Copyright (c) 2026 Ilya Sorokin" which is absent from the README
- Missing algorithm description link and paper reference: all other algorithms link to Wikipedia or arXiv; Tim Peters' 2002 description and the Wikipedia Timsort article are standard references; belongs with the prose description bullet above

## Deferred from: code review of 1-4-public-api-overloads-and-library-integration (2026-06-18)

- Exception safety contract (timsort.hpp public API): `timsort()` allocates `run_stack` and `state.buffer`; if either throws mid-sort the range is left in a valid-but-unspecified state with no documented guarantee; add a contract note when exception-safety audit is done across the library
- Iterator concept check (timsort.hpp:443): `static_assert` uses `is_base_of<random_access_iterator_tag, category>` which rejects non-RA iterators but cannot reject mis-tagged custom iterators that claim RA without satisfying the concept; acceptable C++11 idiom but revisit with C++20 concepts
- Scan-loop exception ordering (timsort.hpp): `cur` and `remaining` advance after `merge_collapse`; a throwing comparator or allocator during merge leaves the run-stack and loop variables in an inconsistent state; same concern as all prior stories — track in a future exception-safety epic
- `run_stack` not pre-reserved (timsort.hpp:461): Python reference timsort uses a fixed 85-entry stack to avoid heap churn; `std::vector` auto-grows correctly but with O(log n) reallocations; profile before optimising

## Deferred from: code review of 2-1-correctness-and-stability-tests (2026-06-18)

- `v5` in `test_correctness` is constructed identically to `v2`; the stated "O(N) path" distinction is not actually different in input construction — matches spec reference implementation but the spec intends them as distinct scenarios; revisit if a future test-quality pass is done
- `merge_hi` gallop bug fix (gallop_left↔gallop_right swap) is not exercised by Story 2.1 tests; Story 2.2 `test_adversarial()` includes an explicit gallop-trigger block that validates this path

## Deferred from: code review of 2-2-edge-case-type-coverage-and-adversarial-tests (2026-06-18)

- Gallop activation not instrumentally verified: test data structurally guarantees gallop (100 consecutive right wins), ASAN/UBSAN clean, but only std::is_sorted asserted; no internal probe confirms gallop code path ran
- merge_hi path (len1>len2) never explicitly targeted: all adversarial merges use equal or right-heavier runs; merge_hi gallop logic untested under adversarial pressure
- Stability not verified for equal strings or equal-key Rec structs: test_type_coverage checks sort order only, not original relative order preserved
- min_gallop persistence across multiple merges not verified: no test constructs multi-merge input and asserts gallop threshold does not reset between merges
- N=2 equal-element case missing from test_edge_cases: (7,7) pair not tested
- N=3 case not tested: smallest range where run extension inserts one element into a 2-element natural run
- Gallop with mixed-win streaks (entry + exit + re-entry cycle) not tested: adaptive gallop threshold decay/growth cycle is unexercised
- merge_collapse n-1 branch not exercised: "merge smaller pair first" tiebreaker (when stack[n-1].len < stack[n+1].len) unreachable with current adversarial data

## Deferred from: code review of 3-1-extend-benchmark-numbers-with-timsort-column (2026-06-18)

- Single-run benchmark, no warm-up or multi-iteration averaging: each data shape is timed once; no statistical rigor; pre-existing pattern matching `Test()`; address in a future benchmark-quality epic if needed
- rng() signed/unsigned mismatch and modulo bias: `mt19937::operator()` returns `uint_fast32_t` used with `int` modulus; also `rng() % NELEM_TIM` has modulo bias since `mt19937::max()+1` is not divisible by 1,000,000; both match existing Generator_* pattern and spec reference impl
- uint32_t loop index over V.size() (size_t): benign with 4 elements; pre-existing idiom from `Test()`

## Deferred from: code review of 3-2-extend-benchmark-strings-with-timsort-column (2026-06-18)

- Fixed algorithm ordering in TestTimsort/TestTimsortStr: timsort always runs first in the timing loop, giving it a cold-cache disadvantage relative to later algorithms; pre-existing benchmark design pattern from existing `Test()`; also applies to benchmark_numbers.cpp (Story 3.1); address in a future benchmark-quality epic alongside single-run averaging

## Deferred from: code review of 1-3-merge-engine-gallop-and-merge-functions (2026-06-18)

- Signed overflow in gallop `ofs = (ofs<<1)|1` (timsort.hpp): theoretical UB on 32-bit ptrdiff_t with >2^30 element runs; impossible in practice; matches CPython reference; revisit if 32-bit ports are required
- `buffer.reserve` responsibility (timsort.hpp:merge_lo/merge_hi): merge functions rely on Story 1.4 to call `reserve(n/2)` upfront; no defensive reserve in the merge functions themselves
- `do_merge` missing bounds assertion: `n+1 < stack.size()` is always satisfied by callers but has no BOOST_ASSERT; add in a future hardening pass
- `merge_collapse` merge-order selector: matches Python reference timsort exactly but is a known variant of the Stijn de Gouw issue; architect should evaluate the Java-style fix in a future epic
