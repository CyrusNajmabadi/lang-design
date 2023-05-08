# Collection literals natural type (List<T>)

This document continues the original design for the [natural type](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md#natural-type) of a collection literal, based on feedback it received.  Specifically, the design is extended in two important ways:

1. The inference system is expanded to support certain cases where a natural element type `T` for `List<T>` can be determined for the empty literal `[]` (previously illegal).
1. Continuing the requirement that the `List<T>` be "well behaved" (i.e. it will not make observable changes outside of its own elements), the language permits a compliant compiler to replace fresh instances of `List<T>` within a method body with more efficient values, as long as such replacement is otherwise not observable.

### Goals

1. Support literal usage like `var v = [x, y, ..z];` (elements provided within the literal).
1. Support literal usage like `var v = []; /*...*/ v.Add(x);` (empty literal with elements provided afterwards).
1. Support literal usage like `var v = [] /*..*/ FillList(v);` (empty literal passed to concrete List processing code).
1. Support a broad number of use cases in a natural/intuitive fashion.
1. Use a well known type, deeply familiar to the .Net/C# ecosystems, with well understood semantics.
1. Provide broad leeway for compiler to produce heavily optimized code (i.e. 'do not leave perf on the table').  Including for code that uses `List<T>` directly, not just collection literals.
1. These optimizations must not violate the stated contract of [List<T>](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=net-8.0).  However, some difference may be observable.
    1. For example `GetHashCode()` may produce different results, as may operations like [Capacity](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1.capacity?view=net-8.0).
    1. The contract includes 'reflection' operations.  e.g. `GetType()` must still return the System.Type instance for `List<T>`.

### Non-goals

1. Support all use cases *directly*, with the same perf, that are provided with `target typed` collection literals.
2. Allow taking a target-typed collection, and moving it to a natural-typed location without a change in semantics.

