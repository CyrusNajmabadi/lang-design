# `List<T>` optimizations

Continuing the requirement from [collection literals](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md) specification that the `List<T>` be "well behaved" (i.e. it will not make observable changes outside of its own elements), the language permits a compliant compiler to replace fresh instances of `List<T>` within a method body with more efficient values (specifically, with different types), as long as such replacement is otherwise not observable.

### `List<T>` natively supported

1. Similar to `System.ValueTuple<>`, `System.Collections.Generic.List<>` is presumed to be well behaved (i.e. no observable side effects outside of its own elements).  

1. `List<T>` is presumed to only have the semantics specified [here](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1).

### Allowable performance optimizations

The compiler is free to perform any or all of the optimizations below for fresh `List<T>` instances within a method body.  These optimizations apply regardless if a collection literal is used, or `new List<T>()` is used (without a capacity).  They can be applied when `new List<T>` is used with a collection initializer (e.g. `new List<T> { a, b, c }`)

These optimizations depend on the list not being used in any fashion that could observe its type (e.g. assigning to another variable, passing to any methods as an argument, etc.).  The optimizations also depend on knowing which methods of `List<T>` can affect the size of the collection, as opposed to just operating on the elements within (for example `Add` vs `Sort`).  It may be appropriate for attributes to be provided on these methods to provide the compiler this information.

Specific optimizations:

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

    ```c#
    // can be stack alloc'ed.  Does not cross await/yield operation.
    var v = [a, b, c];
    foreach (var x in v)
    {

    }
    await task;
    ```

3. Using a `T[]` when the list elements are provided up front and the list size is not mutated, but using the stack is not safe.  For example:

    ```c#
    // Needs to be in heap to survive across await call.
    var v = [a, b, c];
    await task;
    foreach (var x in v)
    ```

4. Using lighter-weight implementations of `List<T>` when it would not be observable.  For example, a ref-struct/value-type version similar to [ValueListBuilder](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/Collections/Generic/ValueListBuilder.cs).  These types would likely need to be provided in `System.Compiler.RuntimeServices` and be kept in sync with `List<T>` by the runtime team.  For example:

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
    // Should allocate an int[] to allow values to survive on the heap for the delegate.
    var v = [1, 2, 3]
    Action x = f;

    void f()
    {
        foreach (var x in v)
            // ...
    }
    ```

    ```c#
    // Should allocate an int[] to allow values to survive on the heap for the delegate.
    var v = [1, 2, 3]
    Action a = () =>
    {
        foreach (var x in v)
            // ...   
    }
    ```

### Optimization examples

```c#
// Compiler should represent this as a stackalloc'ed Span<T> as it is not mutated
List<int> v = new List<int> { a, b, c };
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
// Compiler should represent this as lightweight list-like Span/Array wrapping helper 
// (like ValueListBuilder) as it is mutated, but can stay on the stack.
var v = [a, b, c];
v.Add(d);

foreach (var x in v) ...
```

```c#
// Compiler should represent this as lightweight list-like Array wrapping helper 
// as needs the data on the heap.
var v = [a, b, c];
v.Add(d);
await task;
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