# Collection literals natural type (List<T>)

This document continues the original design for the [natural type](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md#natural-type) of a collection literal.  Specifically, the design is extended in two important ways:

1. The inference system is expanded to support certain cases where a natural element type `T` for `List<T>` can be determined for the empty literal `[]` (previously illegal).
1. Continuing the requirement that the `List<T>` be "well behaved" (i.e. it will not make observable changes outside of its own elements), the language permits a compliant compiler to replace fresh instances of `List<T>` within a method body with more efficient values (specifically, with different types), as long as such replacement is otherwise not observable.

### Goals

1. Support literal usage like:
    1. `var v = [x, y, ..z];` (elements provided within the literal).
    1. `var v = []; /*...*/ v.Add(x);` (empty literal with elements provided afterwards).
    1. `var v = [] /*..*/ FillList(v); ProcessList(v);` (empty literal passed to concrete List processing code).
1. Use a well known type, deeply familiar to the .Net/C# ecosystems, with well understood semantics.
1. Design should be naturally extendible to dictionaries.
1. Support a broad number of use cases in a natural/intuitive fashion.
1. Provide broad leeway for compiler to produce heavily optimized code (i.e. 'do not leave perf on the table').  Including for code that uses ``new List<T>()` directly, not just collection literals.
1. These optimizations must not violate the stated contract of [List<T>](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=net-8.0).  However, some undocumented differences may be observable in practice.
    1. For example `GetHashCode()` may produce different results, as may operations like [Capacity](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1.capacity?view=net-8.0).
    1. The contract includes 'reflection' operations.  e.g. `GetType()` must still return the System.Type instance for `List<T>`.
1. Provide hidden diagnostics and/or analyzers to let users disallow certain scenarios (for example, only allowing collection-literal natural types if they would stack-alloc and not heap alloc).

### Non-goals

1. Support all use cases *directly*, with the same perf, that are provided with `target typed` collection literals.  For example, elements may be stored on the stack, and then copied/moved to the final location, rather than directly being passed to the constructed collection.
2. Allow taking a target-typed collection, and moving it to a natural-typed location without a change in semantics.  This is similar to how the language works already for things like lambdas, formattable strings, interpolated string handlers, numeric values, etc. For example:

    ```c#
    // Directly creates and adds to HashSet in-line.  Legal.
    TakesHashSet([1, 2, 3]);

    // versus

    // Illegal. 'v' is a List<int>.  User must convert to the final HashSet.
    var v = [1, 2, 3];
    TakesHashSet(v);
    ```
3. Support optimized code-gen in all BCL versions.  For example, optimal codegen may depend on new types/methods/attributes being available not present on downlevel versions.

## Detailed design

### `List<T>` natively supported

1. Similar to `System.ValueTuple<>`, `System.Collections.Generic.List<>` is presumed to be well behaved (i.e. no observable side effects outside of its own elements).  

1. `List<T>` is presumed to only have the semantics specified [here](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=net-8.0).

1. A `List<T>` is implicitly convertible to a `Span<T>`.  This will require new APIs present in the BCL.  For example:

    ```c#
    // This works without any overhead.
    var v = [1, 2, 3];
    TakesSpan(v);
    ```

1. If an existing conversion does not already exist, a `List<T>` is explicitly convertible to any constructible collection type.  For example:

    ```C#
    var v = [1, 2, 3];
    TasksImmutableArray((ImmutableArray<int>)v);
    ```

    The goal of this (as opposed to utilizing some existing method to do the conversion) is that the compiler is free to implement this as efficiently as possible in the cases where 'v' is no longer used post conversion.  In the above, this would allow a single array allocation for 'v' and  direct ownership transfer to the ImmutableArray.

    Note: this level of support may not be necessary/desirable.  We could consider not doing this and having users have to explicitly create these other types themselves (e.g. `v.ToImmutableArray()`).


### Natural type element inference

The natural type of a list literal is always some `List<T>`.  Determining the `T` is done in the following fashion.

1. Given a literal of the form `[x, y, ..z]`, `T` is determined using the same  algorithm in the [original spec](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md#natural-type).  Effectively, using `best-common-type` on the element-expressions, and the `iterator type` of spread-elements.

1. Given a literal of the form `var v = [];` type inference for `T` is performed based on the subsequent usage of `v` in the method body, to find lower and upper bounds. If no suitable bounds can be found, the literal is not legal. Specifically:
    1. When 'v' is assigned to types (note: is there a word for that?) of concrete `List<X>` instantiations, then `T` gets an upper and lower bound of `X`.
    1. When 'v' is assigned to types of concrete *invariant* `I<X>` interfaces implemented by `List<X>` (e.g. `IList<X>`), then `T` gets an upper and lower bound of `X`.
    1. When 'v' is assigned to types of concrete *out variant* `I<out X>` interfaces implemented by `List<X>` (e.g. `IEnumerable<X>`), then `T` gets an upper bound of `X`. (TODO: should we also support `I<in X>` interfaces?  Currently there are none on `List<T>`, but we could consider spec'ing it if that ever happens).
    1. Method calls on `v` are examined. If those methods have arguments that use `T`, `I<out T>`, `I<in T>`, or `T[]` then those arguments add appropriate bounds to inference.  Note: `I<>` refers to both interfaces and delegate types. Also, these interfaces/delegates can have multiple type parameters.  These rules only apply to the usage of the `T` type parameter in the in that case.   For example:
        1. `v.Add("")` adds a lower bound of `string`.  `T` case.
        1. `v.AddRange(ienumerableOfStrings)` adds a lower bound of `string`. `I<out T>` case.
        1. `v.Sort(icomparerOfObject)` adds an upper bound of `object`. `I<in T>` case.
        1. `v.CopyTo(stringArray)`.  Adds a lower and upper bound of `string`. `T[]` case.

1. Because the natural type can only be determined for an empty literal based on how the variable it is assigned is used, there is no natural type for an empty literal in any other location.

### Allowable performance optimizations

The compiler is free to perform any or all of the optimizations below for fresh `List<T>` instances within a method body.  These optimizations apply regardless if a collection literal is used, or `new List<T>()` is used (without a capacity).

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

2. Using a `Span<T>` when the list elements are provided up front and the list size is not mutated.  For example:

    ```c#
    // can be stack alloc'ed
    foreach (var x in [a, b, c])
    ```

    ```c#
    // can be stack alloc'ed
    var v = [a, b, c];
    foreach (var x in v)
    ```

3. Using a `T[]` when the list elements are provided up front and the list size is not mutated.  For example:

    ```c#
    // Needs to be in heap to survive across await call.
    var v = [a, b, c];
    await task;
    foreach (var x in v)
    ```

4. Using lighter weight implementations of `List<T>` when it would not be observable.  For example, a ref-struct/value-type version similar to [ValueListBuilder](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/Collections/Generic/ValueListBuilder.cs).  These types would likely need to be provided in `System.Compiler.RuntimeServices` and be kept in sync with `List<T>` by the runtime team.  For example:

    ```c#
    var v = [];
    // complex logic that mutates 'v', for example:
    v.Add(a);

    foreach (var x in v)
    ```

    The above could use a helper type that provided growable span/array semantics, without the additional heap allocation that `List<T>` incurs.

5. Capturing these fresh list values in lambda methods and local functions should ideally still allow varying levels of optimization depending on if a delegate instance is created (always true for lambdas, sometimes true for local functions).  For example:

    ```c#
    // Should be able to stack-alloc
    var v = [1, 2, 3]
    f();

    void f()
    {
        foreach (var x in v)
            // ...
    }
    ```

    ```c#
    // Should allocate an int[] to allow values to survive on the heap for the lambda.
    var v = [1, 2, 3]
    Action a = () =>
    {
        foreach (var x in v)
            // ...   
    }
    ```

## Examples

```c#
// The following is illegal.  No element types are provided, and 'v' is not mutated later.
var v = [];
```

```c#
// Illegal.  `[]` only has a natural type in the `var v = [];` case.
foreach (var v in [])
```

```c#
// Illegal.  `[]` only has a natural type in the `var v = [];` case.
object o = [];
```

```c#
// v has type List<int> because of best-common-type on the literal elements.
var v = [1];
```

```c#
// v has type List<int> because of best-common-type on the literal elements.
object v = [1];
```

### Inference examples

```c#
// Has type List<int> due to 'Add'
var v = [];
v.Add(1);
```

```c#
// Has type List<double> due to 'Add'
var v = [];
v.Add(1);
v.Add(0.0);
```

```c#
// Illegal.  No comptible bounds found.
var v = [];
v.Add(1);
v.Add("");
```

```c#
// Has type List<int> due to 'AddElements'
var v = [];
AddElements(v);

void AddElements(List<int> list);
```

```c#
// Has type List<object> due to upper bound of 'AddElements'
var v = [];
AddElements(v);

void AddElements(IEnumerable<object> list);
```

```c#
// Has type List<string> due to upper bound of 'AddElements' and lower bound of 'Add'. 
var v = [];
AddElements(v);
v.Add("");

void AddElements(IEnumerable<object> list);
```

### Optimization examples

```c#
// Compiler should represent this as a stackalloc'ed Span<T> as it is not
// mutated
var v = [a, b, c];
foreach (var x in v) ...
```

```c#
// Compiler should represent this as a stackalloc'ed Span<T>.  It doesn't cross the `await` so it can stay on the stack
var v = [a, b, c];
foreach (var x in v) ...

await T;
```

```c#
// Compiler should represent this as an array as it is not mutated, but does cross the `await`.
var v = [a, b, c];
await T;
foreach (var x in v) ...
```

```c#
// 'v' is a List<int> as its type could be examined.
var v = [0, 1, 2];
var t = v.GetType();
foreach (var x in v)
```

```c#
// 'v' is a List<int> as its type could be examined.
var v = [0, 1, 2];
TakesObject(v);
foreach (var x in v)
```

```c#
// 'v' is a List<int> as its type could be examined.
var v = [0, 1, 2];
TakesList(v);
foreach (var x in v)
```

```c#
// 'v' is a List<int> as its type could be examined.
var v = [0, 1, 2];
TakesIList(v);
foreach (var x in v)
```

```c#
// 'v' is a List<int> as its type could be examined (unless compiler wants to track through other variables as well)
var v = [1, 2, 3];
var x = v;
foreach (var x in v)
```

```c#
// 'v' is a List<int> as its type could be examined in the extension method on IEnumerable
var v = [1, 2, 3];
var x = v.Select(w => w.ToString());
foreach (var x in v)
```

## Important questions

1. `T` inference can happen with the entire metho and interface surface area of `List<T>`.  This is quite a large surface area.  We could consider limiting to just core methods like `Add/AddRange/this[]`.

2. `T` inference for a `var v = [x, y, ..z]` literal only uses the elements in the literal itself.  We could consider also feeding in the usage information of `v` (like in the `var v = []` case) to inform the inference.  This would change the following:

    ```c#
    var v = [1, 2, 3];
    v.Add(0.0);
    ```

    If only the literal is examined, this would be illegal (as `v` would have the `List<int>` type).  However, if the usage was examined, this would be legal as inference would infer `double` given the bounds provided.