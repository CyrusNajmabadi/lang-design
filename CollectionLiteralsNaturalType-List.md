# Collection literals natural type (List<T>)

This document continues the original design for the [natural type](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md#natural-type) of a collection literal, based on feedback it received.  Specifically, the design is extended in two important ways:

1. The inference system is expanded to support certain cases where a natural element type `T` for `List<T>` can be determined for the empty literal `[]` (previously illegal).
1. Continuing the requirement that the `List<T>` be "well behaved" (i.e. it will not make observable changes outside of its own elements), the language permits a compliant compiler to replace fresh instances of `List<T>` within a method body with more efficient values, as long as such replacement is otherwise not observable.

### Goals

1. Support literal usage like `var v = [x, y, ..z];` (elements provided within the literal).
1. Support literal usage like `var v = []; /*...*/ v.Add(x);` (empty literal with elements provided afterwards).
1. Support literal usage like `var v = [] /*..*/ FillList(v); ProcessList(v);` (empty literal passed to concrete List processing code).
1. Use a well known type, deeply familiar to the .Net/C# ecosystems, with well understood semantics.
1. Support a broad number of use cases in a natural/intuitive fashion.
1. Provide broad leeway for compiler to produce heavily optimized code (i.e. 'do not leave perf on the table').  Including for code that uses `List<T>` directly, not just collection literals.
1. These optimizations must not violate the stated contract of [List<T>](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=net-8.0).  However, some difference may be observable.
    1. For example `GetHashCode()` may produce different results, as may operations like [Capacity](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1.capacity?view=net-8.0).
    1. The contract includes 'reflection' operations.  e.g. `GetType()` must still return the System.Type instance for `List<T>`.

### Non-goals

1. Support all use cases *directly*, with the same perf, that are provided with `target typed` collection literals.
2. Allow taking a target-typed collection, and moving it to a natural-typed location without a change in semantics. For example:

    ```c#
    // Directly creates and adds to hash-set in-line.  Legal
    TakesHashSet([1, 2, 3]);

    // versus

    // Illegal. 'v' is a List<int>.  User must convert to the final HashSet.
    var v = [1, 2, 3];
    TakesHashSet(v);
    ```
3. Support optimized code-gen in all BCL versions.  For example, optimal codegen may depend on new types/methods/attributes being available not present on downlevel versions.

## Detailed design

### `List<T>` natively supported

Similar to `System.ValueTuple<>`, `System.Collections.Generic.List<>` is presumed to be well behaved (i.e. no observable side effects outside of its own elements).  


### Natural type element inference

The natural type of a list literal is always some `List<T>`.  Determining the `T` is done in the following fashion.

1. Given a literal of the form `[x, y, ..z]`, `T` is determined using the same  algorithm in the [original spec](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md#natural-type).  Effectively, using `best-common-type` on the element-expressions, and the `iterator type` of spread-elements.

1. Given a literal of the form `var v = [];` type inference for `T` is performed based on the subsequent usage of `v` in the method body, finding lower and upper bounds for `T`. If no suitable bounds can be found, the literal is not legal. Specifically:
    1. When 'v' is assigned to types (note: is there a word for that?) of concrete `List<X>` instantiations, then `T` gets an upper and lower bound of `X`.
    1. When 'v' is assigned to types of concrete *invariant* `I<X>` interfaces implemented by `List<X>` (e.g. `IList<X>`), then `T` gets an upper and lower bound of `X`.
    1. When 'v' is assigned to types of concrete *out variant* `I<out X>` interfaces implemented by `List<X>` (e.g. `IEnumerable<X>`), then `T` gets an upper bound of `X`. (TODO: should we also support `I<in X>` interfaces?  Currently there are none on `List<T>`, but we could consider spec'ing it).
    1. Method calls on `v` are examined. If those methods have arguments that uses `T`, `I<out T>`, `I<in T>`, `T[]` then those arguments add appropriate bounds to inference.  Note: `I<>` refers to both interfaces and delegate types.  For example:
        1. `v.Add("")` adds a lower bound of `string`.  `T` case.
        1. `v.AddRange(ienumerableOfStrings)` adds a lower bound of `string`. `I<out T>` case.
        1. `v.Sort(icomparerOfObject)` adds an upper bound of `object`. `I<in T>` case.
        1. `v.CopyTo(stringArray)`.  Adds a lower and upper bound of `string`. `T[]` case.

1. Because the natural type can only be determined for an empty literal based on how the variable it is assigned is used, there is no natural type for an empty literal in any other location.

### Allowable performance optimizations

The compiler is free to perform any or all of the following optimizations for fresh `List<T>` instances within a method body.  These optimizations apply regardless if a collection literal is used, or `new List<T>()` is used (without a capacity).

These optimizations depend on the list not being used in any fashion that could observe its type (e.g. assigning to another variable, passing to any methods as an argument, etc.).  The optimizations also depend on knowing which methods of `List<T>` can affect the size of the collection, as opposed to just operating on the elements within (for example `Add` vs `Sort`).  It may be appropriate for attributes to be provided on these methods to provide the compiler this information.

1. Eliding the list entirely.  For example:

    ```c#
    foreach (var x in [1, 2, 3])

    // can be rewritten to:

    for (int x = 1; x <= 3; x++);
    ```

    This also applies to:

    ```c#
    foreach (var x in new List<int> { 1, 2, 3 })

    // can be rewritten to:

    for (int x = 1; x <= 3; x++);
    ```

2. Using a `Span<T>` when the list elements are provided up front, no size-mutating list methods are used, and 