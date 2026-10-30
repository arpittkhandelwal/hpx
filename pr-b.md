# PR draft — Bug B (hpx::ranges::search_n ill-formed)

**Title:**
Fix hpx::ranges::search_n being ill-formed in all overloads

---

Fixes #ISSUE_NUMBER

## Summary

`hpx::ranges::search_n` could not be called at all: all four overloads
failed to compile. The CPO kept a legacy needle-range signature,
`search_n(first, count, s_first, s_last, op, proj1, proj2)`, and
forwarded seven arguments after the policy to
`hpx::parallel::detail::search_n`, which takes six
(`first, last, count, value, pred, proj`). Nothing in-tree instantiated
the ranges version (`search_n_range.cpp` only calls the non-ranges
`hpx::search_n`), so the defect went unnoticed; #6536 fixed the same
class of problem for the non-ranges CPO only.

## Changes

`libs/core/algorithms/include/hpx/parallel/container_algorithms/search.hpp`:

- Rewrote `search_n_t` to the C++ standard ranges signatures, mirroring
  the structure of the neighbouring `search_t` in the same file
  (sentinel support via `detail::search_n<FwdIter, Sent>`,
  `algorithm_result_t` for the policy overloads):
  - `search_n(first, last, count, value[, pred[, proj]])`
  - `search_n(policy, first, last, count, value[, pred[, proj]])`
  - `search_n(rng, count, value[, pred[, proj]])`
  - `search_n(policy, rng, count, value[, pred[, proj]])`
  
  Each overload forwards `(policy, first, last, count, value, pred,
  proj)` to `detail::search_n`, exactly like the (working) non-ranges
  `hpx::search_n` does. The predicate constraint follows the standard:
  `pred` must be invocable with the projected element and the projected
  value (`projected<Proj, FwdIter>` and `projected<Proj, T const*>`).
- Updated the doxygen declarations of `hpx::ranges::search_n` in the
  same file to the new signatures.

Like the neighbouring `hpx::ranges::search`, the algorithm returns an
iterator to the beginning of the found subsequence (`last` if none),
not a `std::ranges::subrange`.

## Verification

The new CPO text was extracted verbatim into a standalone translation
unit against faithful stubs of its HPX dependencies and a model of
`detail::search_n` carrying the exact signatures from
`algorithms/detail/search.hpp`. The previously failing call shapes now
compile (GCC 13, `-std=c++20`, exit 0) and produce the expected results
at run time — covering the iterator and range overloads, sequenced and
parallel policies, an explicit predicate, a projection, the not-found
and zero-count cases, and a range with a distinct sentinel type
(`std::counted_iterator` + `std::default_sentinel`). (This validates the
signatures and forwarding; it is not a substitute for the unit tests
below in a full build.)

## Tests

Extended `search_n_range.cpp` — which previously tested only the
non-ranges `hpx::search_n` — with tests that actually call
`hpx::ranges::search_n`:

- `ranges_search_n_test_basic`: iterator and range overloads, no
  policy / `seq` / `par`, checking the returned iterator
- `ranges_search_n_test_predicate`: custom predicate, iterator and
  range overloads
- `ranges_search_n_test_projection`: projection over a user-defined
  type, iterator and range overloads
- `ranges_search_n_test_sentinel`: a range whose sentinel type differs
  from its iterator type (counted iterator + default sentinel)
- `ranges_search_n_test_not_found` / `ranges_search_n_test_zero_count`
