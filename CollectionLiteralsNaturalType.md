# Collection literals natural type


### Goals:

1. Support literal usage like `var v = [x, y, ..z];` (elements provided within the literal).
1. Support literal usage like `var v = []; /*...*/ v.Add(x);` (empty literal with elements provided afterwards).
1. Provide broad leeway for compiler to produce heavily optimized code (i.e. 'do not leave perf on the table').
1. Support a broad number of use cases in a natural/intuitive fashion.

### Non-goals:

1. Introduce a new *real* type that users would be expected to use themselves.  While there may be new helper types introduced in the BCL (ideally in System.Compiler.RuntimeServices), they may have advanced/subtle semantics and might only be intended for compilers to use.
    1. A corollary of this is that any explicit (non-capture) passing across method/function boundaries would need to be explicit about what real collection type was being used to pass the data. 
1. Have no 'cracks'.  Complex scenarios may reveal non-ideal semantics.  (e.g. how `T?` in unconstrained generics doesn't cleanly work with reference and value types).
1. Have stable reflection semantics.  Operations that attempt to work on the System.Type of these values, or which otherwise use reflection APIs, will not have any guarantee of what they will get, and may observe different results across different use cases, or different versions of the compiler.

## Detailed design:

For the purposes of *discussion/design* only, a new language-only type is introduced called `unspeakable_list<T>`.  Similar to [anonymous types](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#117157-anonymous-object-creation-expressions), this is not a type that can be directly referenced by the user, but has semantics the language defines, and which a compiler is free to implement or emit however it wants as long as those semantics are preserved.

In code samples, `unspeakable_list<T>` will be used to indicate that this is the type of a variable in place of the `var` that a user would have to write in real code.  This will clarify that the `var` is not some other type (like `Span<T>`, `T[]`, `List<T>`, etc.), while also helping see what the element type `T` is in a particular context.  For example: `unspeakable_list<int> v = [1, 2, 3];`

The `unspeakable_list<T>` type has *roughly* the same semantics as the following type, with several important distinctions outlined below:

```c#
ref struct unspeakable_list1<T>
{
    // Original span list was constructed with.  Or wraps _arrayFromPool if that was originally provided (or created in reaction to a mutation operation).
    private Span<T> _span;
    // Heap location if provided at creation, or if original span was not large enough
    private T[]? _arrayFromPool;
    private int _pos;

    // Construction
    public unspeakable_list1(Span<T> span);
    public unspeakable_list1(T[] arrayFromPool);

    // Required members:
    public int Count { get; }
    public ref T this[int index] { get; set; }

    // Might be good for caller to know this for perf reasons
    public int Capacity { get; set; }

    // Used to get at the data, to actually pass to 
    // Span methods, or construct collection instances.
    public ReadOnlySpan<T> AsSpan();

    // Returns array to pool if present
    public void Dispose();

    public void Add(T item);
    public void Insert(int index, T item);

    public void AddRange(params Span<T> span);
    public void InsertRange(int index, params Span<T> span);

    public bool Remove(T item);
    public void Remove(int start, int count);

    public void Clear();

    // This list is not exhaustive, and is subject to a lot of concerns.  See 'Important Questions' below.
}
```

Important aspects of this type:

1. While this is a ref-type, it can be used in contexts where ref-types are normally not allowed (like an async method).
    1. The compiler can still instantiate the `unspeakable_list` using a `Span<T>` if safe to do so (for example, if not used across an `await` call).  
    1. When not safe to use a `Span<T>` the unspeakable_list is created with the constructor that takes an array.  This value then only points at heap data, and can be used safely.
1. The type has a mix of non-mutating and mutating methods.  If the user code does not mutate the variable, the compiler is free to emit the `unspeakable_list directly as a `Span<T>` or `T[]` (depending on if Spans are allowed and the compiler determines if placing on the stack is appropriate).
1. Converting this to a `constructible collection type` has copy semantics.  In other words, the new collection will have the values of the  `unspeakable_list when the new collection is created.  Further mutations to either will not be seen by the other.  If the variable is not referenced after conversion, the compiler is free though to intelligently pass ownership of the data in the `unspeakable_list to the newly constructed collection.

### Inference

1. As the user cannot specify the `unspeakable_list<T>` instantiation directly, it must be inferred.  If the literal contains elements:

    ```c#
    var v = [x, y, ..e];
    ```

    then the element type `T` is determined using the best-common-type algorithm specified at https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md#natural-type.

1. However, if the literal contains no elements, then the element type `T` is determined by how that particular variable is mutated (specifically by the Add/AddRange/Insert/InsertRange/Remove methods).  The variable types passed into these feed into a best-common-type algorithm (spec out) that determines the `T` type. For example:   

    ```c#
    unspeakable_list<int> v = [];
    v.Add(1);
    ```

1. It is illegal to have a natural type for the empty literal that is itself not mutated (which would also be useless in practice).

## Examples:

```c#
// The following is illegal.  No element types are provided, and 'v' is not mutated later.
var v = [];
```

```c#
// v has type unspeakable_list<int> because of best-common-type on the literal elements.
var v = [1];
```  

```c#
// Compiler should represent this as a stackalloc'ed Span<T> as it is not
// mutated and is in a context that supports ref-structs.
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
// Compiler should represent this as an array as it is not mutated, butdoes cross the `await`.
var v = [a, b, c];
await T;
foreach (var x in v) ...
```

```c#
var v = [1, 2, 3];

// v is not referenced used after this line.  Compiler is free to represent 'v' as an array whose ownership is directly passed to the ImmutableArray.
TakesImmutableArray(v);
```

```c#
var v = [1, 2, 3];

// v is referenced after this line.  Compiler will construct fresh ImmutableArray, but keep the original data available.
TakesImmutableArray(v);

// 'v' is now locally [1, 2, 3, 4].  ImmutableArray is unaffected.
v.Add(4);
```

```c#
var v = [1, 2, 3];

// v is not referenced used after this line.  Compiler is free to represent 'v' as an array whose ownership is directly passed to the List.
TakesList(v);
```

```c#
var v = [1, 2, 3];

// v is referenced after this line.  Compiler will construct fresh List, but keep the original data available.
TakesList(v);

// 'v' is now locally [1, 2, 3, 4].  List passed to TakesList is still the values "1, 2, 3".  No sharing of data
v.Add(4);
```

## Important questions

1. There are a whole host of other methods that could possibly be added (found in `IList<T>/List<T>/ImmutableArray<T>/extensions`). Such as:

    1. RemoveAll
    1. Reverse
    1. Sort
    1. Contains
    1. IndexOf/LastIndexOf
    1. BinarySearch
    1. FindIndex/FindLast/FindLastIndex
    1. ForEach/TrueForAll
    1. CopyTo

Should we attempt to provide a maximal, minimal, or somewhere-in-between surface area?

2. Should it be possible to capture these values into interfaces which do *not* copy?  For example:

    ```c#
    unspeakable_list<int> v = [1, 2, 3];
    TakesIList(v); // copy?  or wrap?

    void TakesIList(IList<int> values)
        => values.Add(4);
    ```

    If we do copy in these cases, should there be explicit helper methods that allow non-copying wrapping?  For example:
    
    ```c#
    unspeakable_list<int> v = [1, 2, 3];
    TakesIList(v); // copies.
    TakesIList(v.AsIList()); // wraps.

    void TakesIList(IList<int> values)
        => values.Add(4);
    ```

    Note: wrapping would mean that `unspeakable_list<T>` would be a ref-struct that also supported interfaces (another deviation from normal ref-structs).  This would only be safe as long as the compiler ensured if such wrapping happened that the data not be stored in a `Span<T>` (i.e. the `T[]` construction pattern woudl be used instead).

3. Should it be possible to capture these values across local-function/lambda bodies?  For example:

    ```c#
    unspeakable_list<int> v = [1, 2, 3];
    Func<bool> f = () => v.Contains(0);

    // or
    bool f()
    {
        return v.Contains(0);
    }
    ```

    This is conceivably allowable as long as the compiler heap allocates the data for 'v' in the cases where the capturing code is converted to a delegate (the lambda case, or the local function case when 'f' is converted and not just called directly).  The compiler could still stack allocate in the case of a local function not converted to a delegate. 

4. If a literal is provided initial values, should we also see how it is mutated, and feed both the initial literal values and the mutation values into best-common-type?  For example:

    ```c#
    unspeakable_list<double> v = [1, 2, 3]; // inferred as double because of 'Add' call below
    v.Add(0.0);
    ```

    Should this be illegal (because 'T' was inferred to be `int` from th literal)? Or legal (because both the literal and mutation methods contribute)?

5. Should we allow boxing these types into `object`/`dynamic`.  If we did, we would certainly disallow the span-usage form.  However, even if wrapping an array, it feels like it would be extremely difficult to use such a value effectively.  Perhaps best to disallow.