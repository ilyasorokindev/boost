# Deferred Work Ledger

## Deferred from: code review of 1-4-public-api-overloads-and-library-integration (2026-06-18)

- Exception safety contract (timsort.hpp public API): `timsort()` allocates `run_stack` and `state.buffer`; if either throws mid-sort the range is left in a valid-but-unspecified state with no documented guarantee; add a contract note when exception-safety audit is done across the library
- Iterator concept check (timsort.hpp:443): `static_assert` uses `is_base_of<random_access_iterator_tag, category>` which rejects non-RA iterators but cannot reject mis-tagged custom iterators that claim RA without satisfying the concept; acceptable C++11 idiom but revisit with C++20 concepts
- Scan-loop exception ordering (timsort.hpp): `cur` and `remaining` advance after `merge_collapse`; a throwing comparator or allocator during merge leaves the run-stack and loop variables in an inconsistent state; same concern as all prior stories — track in a future exception-safety epic
- `run_stack` not pre-reserved (timsort.hpp:461): Python reference timsort uses a fixed 85-entry stack to avoid heap churn; `std::vector` auto-grows correctly but with O(log n) reallocations; profile before optimising

## Deferred from: code review of 2-1-correctness-and-stability-tests (2026-06-18)

- `v5` in `test_correctness` is constructed identically to `v2`; the stated "O(N) path" distinction is not actually different in input construction — matches spec reference implementation but the spec intends them as distinct scenarios; revisit if a future test-quality pass is done
- `merge_hi` gallop bug fix (gallop_left↔gallop_right swap) is not exercised by Story 2.1 tests; Story 2.2 `test_adversarial()` includes an explicit gallop-trigger block that validates this path

## Deferred from: code review of 1-3-merge-engine-gallop-and-merge-functions (2026-06-18)

- Signed overflow in gallop `ofs = (ofs<<1)|1` (timsort.hpp): theoretical UB on 32-bit ptrdiff_t with >2^30 element runs; impossible in practice; matches CPython reference; revisit if 32-bit ports are required
- `buffer.reserve` responsibility (timsort.hpp:merge_lo/merge_hi): merge functions rely on Story 1.4 to call `reserve(n/2)` upfront; no defensive reserve in the merge functions themselves
- `do_merge` missing bounds assertion: `n+1 < stack.size()` is always satisfied by callers but has no BOOST_ASSERT; add in a future hardening pass
- `merge_collapse` merge-order selector: matches Python reference timsort exactly but is a known variant of the Stijn de Gouw issue; architect should evaluate the Java-style fix in a future epic
