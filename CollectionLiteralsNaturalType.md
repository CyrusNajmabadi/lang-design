# Collection literals natural type


### Goals:

1. Support literal usage like `var v = [x, y, ..z];` (elements provided within the literal).
1. Support literal usage like `var v = []; /*...*/ v.Add(x);` (empty literal with elements provided afterwards).
1. Provide broad leeway for compiler to produce heavily optimized code (i.e. 'do not leave perf on the table').
1. Support broad number of user cases in a natural/intuitive fashion.

### Non-goals:

1. Introduce a new *real* type that users would explicitly use themselves.  While there may be new helper types introduced in the BCL (ideally in System.Compiler.RuntimeServices), they may have advanced/confusing semantics and might only be intended for compilers to use. 
1. Have no 'cracks'.  Complex scenarios may reveal non-ideal semantics.  (e.g. how `T?` in unconstrained generics doesn't cleanly work with reference and value types).

## Detailed design:

For the purposes of *discussion/design* only, a new language-only type is introduced called ``anonymous_list`<T>``.  Similar to [anonymous types](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#117157-anonymous-object-creation-expressions), this is not a type that can be directly referenced by the user, but has semantics the language defines, and which a compiler is free to implement or emit however it wants as long as those semantics are preserved.

In code samples ``anonymous_list`<T>`` will be used to indicate that this it the type of a variable in place of the `var` that a user would have to write in real code.  This will clarify that the `var` is not some other type (like `Span<T>`, `T[]`, `List<T>`, etc.), while also helping see what the element type `T` is in a particular context.  For example: ``anonymous_type`<int> v = [1, 2, 3];``

The ``anonymous_list`<T>`` type has roughly the same semantics as the following type:

```c#
ref struct anonymous_list1`<T>
{
    // Original span list was constructed with.  Or wraps _arrayFromPool if that is present.
    private Span<T> _span;
    // Heap location if provided at creation, or if original span was not large enough
    private T[]? _arrayFromPool;
    private int _pos;

    // Construction
    public anonymous_list1`(Span<T> span);
    public anonymous_list1`(T[] arrayFromPool);

    // Required members:
    public int Count { get; }
    public ref T this[int index] { get; set; }

    // Might be good for caller to know this for perf reasons
    public int Capacity { get; set; }

    public ReadOnlySpan<T> AsSpan();

    // Returns array to pool
    public void Dispose();

    // Mutation:

    // Possibly redundant if we have params-span
    public void Add(T item);
    public void Insert(int index, T item);

    public void Add(params Span<T> span);
    public void Insert(int index, params Span<T> span);

    public bool Remove(T item);
    public void Remove(int start, int count);

    public void Clear();
}
```

There are a whole host of other methods that could possibly be added (found in `IList<T>/List<T>/ImmutableArray<T>/extensions`). Such as:

1. RemoveAll
1. Reverse
1. Sort
1. Contains
1. IndexOf/LastIndexOf
1. BinarySearch
1. FindIndex/FindLast/FindLastIndex
1. ForEach/TrueForAll
1. CopyTo

Important aspects of this type:

1. While this is a ref-type, it can be used in contexts where ref-types are normally not allowed (like an async method).
    1. The compiler can still instantiate the `anonymous_list` using a `Span<T>` if safe to do so (for example, if not used across an `await` call).  
    2. When not safe to use a `Span<T>` the anonmous_list is created with the constructor that takes an array.  This value then only points at heap data, and can be used safely.
