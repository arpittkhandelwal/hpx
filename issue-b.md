# Issue draft — Bug B (hpx::ranges::search_n ill-formed)

**Title:**
hpx::ranges::search_n is ill-formed in all overloads — any call fails to compile

---

## Summary

Any call to `hpx::ranges::search_n` fails to compile, in all four
overloads (iterator/range × with/without execution policy). The ranges
CPO still uses the legacy HPX needle-range signature —
`search_n(first, count, s_first, s_last, op, proj1, proj2)` — instead of
the standard `std::ranges::search_n` signature
`search_n(first, last, count, value, pred, proj)`, and it forwards seven
arguments after the execution policy to `hpx::parallel::detail::search_n`,
whose implementation takes six:

```cpp
// libs/core/algorithms/include/hpx/parallel/algorithms/detail/search.hpp
static FwdIter sequential(ExPolicy, FwdIter first, Sent last, Size count,
    T const& value, Pred&& pred, Proj&& proj);
```

## Steps to Reproduce the Problem

```cpp
#include <hpx/algorithm.hpp>

#include <vector>

void f()
{
    std::vector<int> v{1, 2, 2, 2, 3};

    // the standard ranges call shape — does not compile
    auto it = hpx::ranges::search_n(v, 2, 2);
}
```

Reducing the CPO's forwarding call and the detail implementation to
their exact signatures, GCC 13 (`-std=c++20`) reports:

```
error: no matching function for call to
  'detail_search_n<...>::call(const seq_policy&, iterator&, std::size_t&,
   iterator&, iterator&, equal_to&, identity&, identity&)'
note: candidate expects 7 arguments, 8 provided
```

(the candidate count includes the policy). The call has one argument too
many, in the wrong order and with the wrong types: the `count` lands in
the implementation's `Sent last` slot, the needle iterators land in the
`count`/`value` slots, and there are two projections where the
implementation takes one. The iterator overloads additionally instantiate
`detail::search_n<FwdIter, FwdIter>` — the needle iterator type used as
the sentinel type. The identical call rewritten in the shape the
implementation expects (`policy, first, last, count, value, pred, proj`)
compiles cleanly, so the implementation is satisfiable and the CPO
forwarding is the broken part.

## Cause

`search_n_t::invoke_default` in
`libs/core/algorithms/include/hpx/parallel/container_algorithms/search.hpp`
was never ported from the old needle-range shape (where the searched-for
sequence was given as a second iterator range) to the standard
count/value shape that `detail::search_n` — and the non-ranges
`hpx::search_n` — use.

Nothing in-tree instantiates the ranges version, which is why this
shipped: `search_n_range.cpp` only calls the non-ranges `hpx::search_n`.
Related: #6536 fixed the analogous problem for the non-ranges
`hpx::search_n` CPO but missed the ranges CPO.

## Expected Behavior

`hpx::ranges::search_n` should accept the standard signatures —

```cpp
search_n(first, last, count, value[, pred[, proj]])
search_n(policy, first, last, count, value[, pred[, proj]])
search_n(rng, count, value[, pred[, proj]])
search_n(policy, rng, count, value[, pred[, proj]])
```

— and return an iterator to the beginning of the first subsequence of
`count` consecutive elements equal to `value` (or `last` if there is
none), like `std::ranges::search_n` (modulo returning an iterator rather
than a subrange, as the neighbouring `hpx::ranges::search` does).

## Specifications

- HPX Version: master (2152675)
- Platform (compiler, OS): Linux, GCC 13
