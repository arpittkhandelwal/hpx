# PR draft — Bug A (data loss in parallel set_union / set_symmetric_difference)

**Title:**
Fix parallel set_union/set_symmetric_difference dropping elements of the second range

---

Fixes #ISSUE_NUMBER

## Summary

The parallel `set_union` and `set_symmetric_difference` (iterator and
`hpx::ranges::` versions alike) silently dropped elements of the second
input range: whenever a whole chunk of the first range consisted of a
single repeated value and the second range contained smaller elements,
those elements appeared in no chunk's output. The result was a
well-formed but too short sequence.

Smallest reproducer (3+ processing units): `set_union` of
`a = {1, 1, 2}` and `b = {0, 2, 2}` returned `{1, 1, 2, 2}` instead of
`{0, 1, 1, 2, 2}` — the `0` was lost.

## Root cause

All four parallel set operations share `detail::set_operation()` in
`libs/core/algorithms/include/hpx/parallel/algorithms/detail/set_operation.hpp`.
Its per-chunk functor `f1` skips a non-last chunk entirely when the
chunk's slice of the first range is a single repeated value, assuming the
next chunk will handle it. That holds for the first range (the next chunk
extends its start backwards over the duplicates), but not for the second
range: the next chunk starts its slice of the second range at
`lower_bound(range2, start_value)`. The first chunk is the only one whose
slice of the second range starts at position 0, so skipping it orphans
every second-range element smaller than the repeated value. (Non-first
skipped chunks lose nothing: the neighbouring chunks' second-range
slices meet exactly at `lower_bound(range2, value)`.)

## Fix

In `f1`'s skip branch, the first partition no longer returns early. It
falls through with an empty slice of the first range (`end1 = start1`),
so the sequential set operation for that chunk runs on
`[first2, lower_bound(range2, end_value))` and emits exactly the orphaned
elements for `set_union`/`set_symmetric_difference` — and nothing for
`set_intersection`/`set_difference`, preserving their current (correct)
behaviour. The buffer placement stays consistent: the chunk reports
`start = combiner(0, 0) = 0` and the following chunk's buffer region
begins exactly after this chunk's output. The returned `in1`/`in2`
positions are unaffected (the following chunk reports the real consumed
positions and `f2` takes the maximum).

Verified by re-running an exhaustive fuzz of the extracted chunking
logic (sorted inputs over a 1–4-symbol alphabet, lengths < 25, 1–8
chunks; 1,182,148 configurations across all four operations): mismatches
went from 7,276 (all in `set_union`/`set_symmetric_difference`) to 0,
and the returned result positions still match the sequential semantics.

## Tests

Added regression tests to the ranges test files of the two affected
operations:

- `test_set_union_values_duplicate_run` in `set_union_range.cpp`
- `test_set_symmetric_difference_values_duplicate_run` in
  `set_symmetric_difference_range.cpp`

Each runs the parallel algorithm on the failing shapes (a first range
that is one long duplicate run with a second range of smaller values,
the small `{1, 1, 2}` / `{0, 2, 2}` case, straddling values, and a
no-smaller-values control) and checks the full output contents and the
returned result size against `std::set_union` /
`std::set_symmetric_difference`. The existing tests missed the bug
because `test::random_fill` data never produces a duplicate run covering
a whole chunk.
