# Collection Literals C# 12 Plan

This document summarizes the proposed parts of [Collection Literals](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md) we would like to ship in C# 12.  It separates out the proposal into parts we strongly feel should be in the initial release, and parts which could potentially come later.

## Desired Pieces
[desired-pieces]: #desired-pieces 

1. Target-typing collection literals for the following sequence-like types:
    1. `Span<T>`
    
    1. `T[]`

    1. Types supporting collection initializers (like `List<T>`, `HashSet<T>` etc.)

    1. Interfaces implemented by `List<T>` (like `IEnumerable<string>`, etc.) 

    1. Types supporting a *new* `Construct` creation path (used for types like `ImmutableArray<T>`). Note: this may be at risk as it will require work with runtime to define pattern/attributes to support this.  See [Optional Pieces](#p[#optional-pieces]) for more details on this.

1. Target-typing collection literals for map-like types.
    1. Map-like types supporting ~~collection initializers~~
        1. `Add` semantics, or `overwrite` semantics?
    1. Interfaces implemented by `Dictionary<TKey, TValue>`
    1. `ImmutableDictionary<K,V>`

    
1. Natural-typing
    1. non-empty, non-dictionary collection literals to `List<T>` (using best-common-type algorithm on element types).

    1. non-empty, dictionary collection literals to `Dictionary<TKey,TValue>`.

    1. Empty list/dictionary listerals handled in [Optional Pieces](#p[#optional-pieces]).


## Optional Pieces
[optional-pieces]: #optional-pieces 

1. New `Construct` path for creating special types.  Needs design with Runtime.  With the methods either on the type, or found through some special attribute.  Compiler calls into method to initialize space, then gets the Span it copies the values into.  This would allow types like `ImmutableArray<T>` (or proposed future `ValueList<T>` types) to act like a `collection initializer` when the capacity could be predetermined. This may be a family of methods.  For example:

    1. `public static List<T> Create<T>(int count, int capacity, out Span<T> span);`

    1. `public static ImmutableArray<T> Create<T>(int count, out Span<T> span);`

    1. `public ValueListBuilder<T>(Span<T> buffer, int count);`

    1. It would be desirable to prefer these paths over the collection-initializer path.  `List<T>`, for example, would be much faster to initialize in this form, versus repeated `.Add` calls.

1. Optimizations around fresh `List<T>` instances.  It would be nice if collection literals could be used within a method-body without unnecessary overhead when not desirable. For example:

    ```c#
    var v = [GetA(), GetB(), GetC()]; // or even:
    List<Something> v = new List<Something>() { GetA(), GetB(), GetC() };

    foreach (var string in v)
    ```

    This should ideally be stack-alloced without the multiple heap allocations that `List<T>` incurs.  [`List<T>` Performance](ListPerformance.md) covers a proposal for how this would work, transparently speeding up existing applications, while also making it more palatable for performance conscious users to use collection literals.

1. Supporting natural types for empty collection literals.  e.g. `var v = [];`.  This is challenging due to the lack of any information in the initializer of the `var` to provide type information.
    1. This may be something we never support (due to being too complex).  However,

    1. [Inferred Generic Type](InferredGenericType.md) proposes a supported way to handle this situation.  Namely through the introduction of 'partially inferred generics' like `List<_>`, where `var v = [];` would be equivalent to `List<_> v = new List<_>();`.  In this system, the outer type is concretely known, while the instantiation is determined based on the usage of the variable later in the method.  For example, calling `v.Add(0)` would contribute to inferring the variable type to be `List<int>`.

    1. We would likely need a special syntax for empty dictionary literals in this case.  For example: `var v = [:];`.