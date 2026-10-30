# Issue draft — Bug A (data loss in parallel set_union / set_symmetric_difference)

**Title:**
Parallel set_union and set_symmetric_difference silently drop elements of the second range

---

## Expected Behavior

`hpx::set_union` / `hpx::ranges::set_union` (and the
`set_symmetric_difference` counterparts) with a parallel execution policy
produce the same output values as the sequential algorithms: every element
of the second range that belongs into the result appears in the output,
and the returned `out` iterator reflects the full result size.

## Actual Behavior

With a parallel execution policy, elements of the second input range are
silently missing from the output — the result is a well-formed but too
short sequence; no error is reported. This happens whenever a whole chunk
of the first range consists of a single repeated value and the second
range contains elements smaller than that value. Only `set_union` and
`set_symmetric_difference` are affected (they are the operations that emit
elements coming only from the second range); `set_intersection` and
`set_difference` produce correct values.

Note this is distinct from #7932, which was about the returned `in1`/`in2`
iterators — there the output *values* were correct; here the values
themselves are wrong.

## Steps to Reproduce the Problem

```cpp
#include <hpx/algorithm.hpp>
#include <hpx/execution.hpp>
#include <hpx/init.hpp>

#include <cstdio>
#include <vector>

int hpx_main()
{
    // smallest case (needs a machine with at least 3 processing units,
    // so that the parallel chunk size of the first range degenerates
    // to a single element)
    std::vector<int> a{1, 1, 2};
    std::vector<int> b{0, 2, 2};
    std::vector<int> out(6);

    auto r = hpx::ranges::set_union(
        hpx::execution::par, a, b, out.begin());
    out.resize(r.out - out.begin());

    // expected: 0 1 1 2 2
    // actual:   1 1 2 2      (the 0 from the second range is lost)
    for (int v : out)
        std::printf("%d ", v);
    std::printf("\n");

    return hpx::local::finalize();
}

int main(int argc, char* argv[])
{
    return hpx::local::init(hpx_main, argc, argv);
}
```

A larger, thread-count-independent example: `a` = 1000 copies of `5`,
`b = {1, 2, 3}`. The expected union is `{1, 2, 3}` followed by 1000 copies
of `5`; the parallel algorithm returns only the 1000 copies of `5`.

## Cause

All four parallel set operations share `detail::set_operation()` in
`libs/core/algorithms/include/hpx/parallel/algorithms/detail/set_operation.hpp`.
The first range is split into `cores` contiguous chunks. For each
non-last chunk, the per-chunk functor `f1` first checks whether the whole
chunk is a single repeated value (the chunk's first value is not less
than the first value of the next chunk); if so, it returns without doing
any work, on the theory that "this chunk will be handled by the next
one".

That hand-off works for the first range: the next chunk walks its start
backwards over all elements equal to its first value, re-processing the
skipped chunk's elements. It does not work for the second range. The next
chunk computes its slice of the second range as starting at
`lower_bound(range2, start_value)`. For the *first* chunk — the only chunk
whose slice of the second range starts at the very beginning of that
range — skipping therefore orphans every element of the second range that
is smaller than the repeated value: no chunk ever processes them.
(Skipped non-first chunks are harmless: the previous chunk's slice of the
second range ends at `lower_bound(range2, value)` and the next chunk's
slice starts at the same position, because a skipped chunk implies both
neighbours share its value at the boundary.)

The returned output length shrinks accordingly (the `f2` compaction sums
the per-chunk lengths), so the loss is silent.

The existing tests miss this because they fill the inputs with
`test::random_fill(10007)` — values spread over a wide range, where a
duplicate run never covers a whole chunk. Low-cardinality data (sorted
timestamps, status codes, categorical keys) or small inputs on
many-threaded machines trigger it readily.

## Specifications

- HPX Version: master (2152675)
- Platform (compiler, OS): Linux, GCC 13

## Additional Details

Verified against the sequential result by exhaustive fuzzing of the
chunking logic (sorted inputs over a 1–4-symbol alphabet, lengths < 25,
1–8 chunks): 7,276 of 1,182,148 configurations produced wrong values,
all of them in `set_union` or `set_symmetric_difference`, none in
`set_intersection` or `set_difference`.
