# Collection Literals C# 12 Plan

This document summarizes the proposed parts of [Collection Literals](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md) we would like to ship in C# 12.  It separates out the proposal into parts we strongly feel should be in the initial release, and parts which could potentially come later.

## Desired Pieces
[desired-pieces]: #desired-pieces 

1. Target-typing collection literals for the following sequence-like types:
    1. `Span<T>`
    1. `T[]`
    1. Types supporting collection initializers (like `List<T>`, `HashSet<T>` etc.)
    1. Types supporting a *new* `Construct` creation path (used for types like `ImmutableArray<T>`). Note: this may be at risk as it will require work with runtime to define pattern/attributes to support this.
1. Natural-typing non-empty, non-dictionary collection literals to `List<T>` (using best-common-type algorithm on element types).  See [Optional Pieces](#p[#optional-pieces])
1. Natural-typing non-empty, dictionary collection literals to `Dictionary<TKey,TValue>`.

