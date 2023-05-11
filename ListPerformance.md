# `List<T>` optimizations

Continuing the requirement from [collection literals](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md) specification that the `List<T>` be "well behaved" (i.e. it will not make observable changes outside of its own elements), the language permits a compliant compiler to replace fresh instances of `List<T>` within a method body with more efficient values (specifically, with different types), as long as such replacement is otherwise not observable.

### `List<T>` natively supported

1. Similar to `System.ValueTuple<>`, `System.Collections.Generic.List<>` is presumed to be well behaved (i.e. no observable side effects outside of its own elements).  

1. If present, certain types (see [New BCL types](#new-bcl-types)) are trusted to be drop-in replacements for `List<T>` for the specific usage in a method body.

### Allowable performance optimizations

The compiler is free to perform any or all of the optimizations below for fresh `List<T>` instances within a method body.  These optimizations apply regardless if a collection literal is used, or `new List<T>()` is used (without a capacity).  They can be applied when `new List<T>` is used with a collection initializer (e.g. `new List<T> { a, b, c }`)

These optimizations depend on the list not being used in any fashion that could observe its type (e.g. assigning to another variable, passing to any methods as an argument, etc.).  The optimizations also depend on knowing which methods of `List<T>` can affect the size of the collection, as opposed to just operating on the elements within (for example `Add` vs `Sort`).  See the [New BCL Types](#new-bcl-types) section for how that is determined.

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

2. Using purely stack allocated data when the list elements are provided up front and the list size is not mutated.  For example:

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

3. Using a heap allocated array `T[]` when the list elements are provided up front and the list size is not mutated, but using the stack is not safe.  For example:

    ```c#
    // Needs to be in heap to survive across await call.
    var v = [a, b, c];
    await task;
    foreach (var x in v)
    ```

4. Using lighter-weight implementations of `List<T>` when it would not be observable.  These types would likely need to be provided in `System.Compiler.RuntimeServices` and be kept in sync with `List<T>` by the runtime team. See [New BCL Types](#new-bcl-types) for more information.

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

6. `new List<T>(capacity: ...)` is also something that should be supported.  Depending on if the capacity is constant or not, and if the variable is ref-safe, the appropriate collection will be picked.  For example:

```c#
// Should use RefList<int> here
var v = new List<int>(capacity: 3);
v.Add(1);
v.Add(2);
```

### Optimization examples

```c#
// Compiler should represent this as a stackalloc'ed Span<T> as it is not mutated
var v = new List<int> { a, b, c };
foreach (var x in v) ...
```

```c#
// Compiler should represent this as a stackalloc'ed Span<T>.  It doesn't cross the `await` so it can stay on the stack
var v = new List<int> { a, b, c };
foreach (var x in v) ...

await T;
```

```c#
// Compiler should represent this as an array as it is not mutated, but does cross the `await`.
var v = new List<int> { a, b, c };
await T;
foreach (var x in v) ...
```

```c#
// Compiler should represent this as lightweight list-like Span/Array wrapping helper 
// (like ValueListBuilder) as it is mutated, but can stay on the stack.
var v = new List<int> { a, b, c };
v.Add(d);

foreach (var x in v) ...
```

```c#
// Compiler should represent this as lightweight list-like Array wrapping helper 
// as needs the data on the heap.
var v = new List<int> { a, b, c };
v.Add(d);
await task;
foreach (var x in v) ...
```

```c#
// 'v' is a List<int> as its type could be examined.
var v = new List<int> { a, b, c };
var t = v.GetType();
foreach (var x in v)
```

```c#
// 'v' is a List<int> as its type could be examined.
var v = new List<int> { a, b, c };
TakesObject(v);
foreach (var x in v)
```

```c#
// 'v' is a List<int> as its type could be examined.
var v = new List<int> { a, b, c };
TakesList(v);
foreach (var x in v)
```

```c#
// 'v' is a List<int> as its type could be examined.
var v = new List<int> { a, b, c };
TakesIList(v);
foreach (var x in v)
```

```c#
// 'v' is a List<int> as its type could be examined (unless compiler wants to track through other variables as well)
var v = new List<int> { a, b, c };
var x = v;
foreach (var x in v)
```

```c#
// 'v' is a List<int> as its type could be examined in the extension method on IEnumerable
var v = new List<int> { a, b, c };
var x = v.Select(w => w.ToString());
foreach (var x in v)
```

## New BCL types.
[new-bcl-types]: #new-bcl-types

In order to support these optimizations, the BCL will be expanded to include the following types:

1. `System.Runtime.CompilerServices.FixedSizeRefList<T>`
1. `System.Runtime.CompilerServices.FixedSizeValueList<T>`
1. `System.Runtime.CompilerServices.RefList<T>`
1. `System.Runtime.CompilerServices.ValueList<T>`

These types are not intended for end user use.  But, at the same time, nothing blocks that.  The types will have complex and subtle (especially in ref-struct/value-struct space) semantics, which is why these exist in `CompilerServices` and not `System.Collections.Generic`.

The APIs will mirror the API of `List<T>`.  When `List<T>` receives any api changes, they should be placed in the appropriate apis above (depending on if they might change the size of the list or not).  These apis have the following characteristics:

1. `FixedSizeRefList<T>` and `RefList<T>` are ref-structs.
1. `FixedSizeValueList<T>` and `ValueList<T>` are normal structs.
1. The `FixedSizeXXX<T>` types do not have any methods on them that could affect the `Count` of items in the list.
1. The `FixedSizeRefList<T>` and `RefList<T> types can point at a span holding the data.
1. The `RefList<T>` type can also point at a rented array that holds the data (in the case that its `Span` isn't large enough)
2. The `FixedValueList<T>` and `ValueList<T>` types point at a rented array that holds the data.

Analysis of how a fresh `List<T>` variable is used will determine which of the above types to use.  Specifically:

1. If the usage only involves methods present in the `FixedSizeRefList<T>` or  `FixedSizeValueList<T>`types, then those will be used versus the non-fixed size versions.  In other words, if the user doesn't do anything to change the size of the list, then we can use fixed-size storage without any other overhead.  Otherwise, the non-fixed size types are used.
2. If the data for the list can live entirely on the stack (i.e. it doesn't cross an `await` call), then the `FixedSizeRefList<T>` or `RefList<T>` types can be used.  Otherwise the `FixedSizeValueList<T>` or `ValueList<T>` types are used.
3. Depending on which of these are used, the compiler will allocate space on the stack (as a `Span<T>`) or on the heap (as a `T[]`) which is passed into the type.
4. If the variables do not escape, and involve rented arrays, those arrays are returned to the pool when the variable goes out of scope.
5. The compiler is free to eschew using these types as well if such work is considered desirable/reasonable.  For example, emitting a `Span<T>` directly and operating on it itself.  This should only be done though if there are tangible benefits, and the cost of having the compiler understand and implement a subset of `List<T>` operations on `Span<T>` is acceptable.

### New BCL types shape

Briefly, this is what the types would look like. Note: These are more sketches for brevity to keep the specification clearer.

```c#
// Lowest overhead type.  Used whenever:
// 1. we can't completely elide a collection literal.
// 2. the number of elements is low enough the compiler is ok stack-allocating them.
// 3. the list's size is never changed. 
// 4. the list is not used in a way incompatible with storing on the stack.
public ref struct FixedSizeRefList<T>
{
    // Where the actual data is stored.
    // This ensures the size of the List is no more than the size of the span.
    private readonly Span<T> _data;

    public FixedSizeRefList<T>(Span<T> data);

    // All the same members from List<T>.  Nothing is virtual and inline-ability 
    // should be very good.

    public int Capacity { get; } // no setter
    public int Count { get; }
    public T this[int index] { get; set; } // setter is fine, it doesn't change size

    public ReadOnlyCollection<T> AsReadOnly();
    int BinarySearch(int index, int count, T item, IComparer<T>? comparer); // And the other BinarySearch overloads.
    public bool Contains(T item);
    public void CopyTo(T[] array); // And the other CopyTo overloads.
    public bool Exists(Predicate<T> match);
    public T? Find(Predicate<T> match); // And the FindAll/FindIndex/FindLast/FindLastIndex overloads
    public void ForEach(Action<T> action);
    public bool TrueForAll(Predicate<T> match);
    public int IndexOf(T item); // And the other IndexOf/LastIndexOf overloads.
    public void Reverse(); // And the other Reverse overloads.
    public void Sort(); // And the other Sort overloads.

    public T[] ToArray();

    public List<TOutput> ConvertAll<TOutput>(Converter<T, TOutput> converter);
    public List<T> GetRange(int index, int count);
    public List<T> Slice(int start, int length);

    public Enumerator GetEnumerator();
    public ref struct Enumerator { ... }
}

// Can be disposed.  Returns rented array if present.
public ref struct RefList<T>
{
    // Where the actual data is stored, may be on stack, or may point at _rentedArray
    private Span<T> _data;
    private int _pos;
    private T[]? _rentedArray;

    // Has all methods from FixedSizeRefList<T> with the following changes/additions:

    public int Capacity { get; set; } // now has setter

    public void Add(T item);
    public void AddRange(IEnumerable<T> collection);
    public void Clear();
    public int EnsureCapacity(int capacity);
    public void Insert(int index, T item);
    public void InsertRange(int index, IEnumerable<T> collection);
    public bool Remove(T item); // And the RemoveAll/RemoveAt/RemoveRange overloads.
    public void TrimExcess();

    public Enumerator GetEnumerator();
    public ref struct Enumerator { ... }
}

// Can be disposed if not captured.  Returns rented array if present.
public struct FixedSizeValueList<T>
{
    private readonly T[] _rentedArray;

    // Same api as FixedSizeRefList<T>
}

// Can be disposed if not captured.  Returns rented array if present.
public struct ValueList<T>
{
    private int _pos;
    private T[] _rentedArray;

    // Same api as RefList<T>
}