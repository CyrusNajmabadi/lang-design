# Construct methods

In all cases, types are presumed to have a `[CollectionBuilder(typeof(SomeType), Name = "Create")]` like attribute on them to direct the compiler on where to look to find the helper methods.

## Pattern 1:

1. Fixed length collection literals.
2. Collections with contiguous backing storage.
3. Ref-struct possible

```c#
static class CollectionsMarshal
{
    public static void Create<T>(int capacity, out List<T> list, out Span<T> storage); 
    public static void Create<T>(int capacity, out ImmutableArray<T> list, out Span<T> storage);
}
```

Usage:

```c#
ImmutableArray<string> values = ["a", "b", "c"];
```

Translation:

```c#
CollectionsMarshal.Create<string>(capacity: 3, out ImmutableArray<string> values, out Span<T> __storage);
__storage[0] = "a";
__storage[1] = "b";
__storage[2] = "c";
```

## Pattern 2

1. Fixed length collection literals.
2. Collections with contiguous backing storage.
3. Ref-struct not possible

```c#
static class CollectionsMarshal
{
    public static void Create<T>(int capacity, out List<T> list, out T[] storage); 
    public static void Create<T>(int capacity, out ImmutableArray<T> list, out T[] storage);
}
```

// We're a little hesitant here because of exposing array, and allowing abuse.
// Though it feels like we may have already crossed that because we're in CollectionsMarshal
// and we already allow things like making an IA that wraps an array that others point at.

// We could be ok with less efficiency here, esp since you're already async/await.

// if we don't have the array, we could have hte compiler new-up an array, put the values in it,
// then call the `out span` version right at the end, and then copy into it.



Usage:

```c#
ImmutableArray<string> values = [await GetA(), await GetB(), await GetC()];
```

Translation:

```c#
CollectionsMarshal.Create<string>(capacity: 3, out ImmutableArray<string> values, out T[] __storage);
__storage[0] = await GetA();
__storage[1] = await GetB();
__storage[2] = await GetC();
```


// What about:

```c#
public static void Create<T>(int capacity, out ImmutableArray<T> list, out Memory<T> storage);
```

Translation:

```c#
CollectionsMarshal.Create<string>(capacity: 3, out ImmutableArray<string> values, out Memory<T> __storageMemory);
__storageMemory.Span[0] = await GetA();
__storageMemory.Span[1] = await GetB();
__storageMemory.Span[2] = await GetC();
```


Alternative: Caller allocates and populates array, then passes to method to take ownership.

Do we need pattern 1 if we have pattern 2?  Pattern 1 would work for the rare case where the collection itself is a ref-struct not backed by array.  Does that case exist?

## Pattern 3:

1. Fixed length collection literals.
2. Collections with non-contiguous backing storage.
3. Ref-struct possible

```c#
static class CollectionsMarshal
{
    public static void Create<T>(ReadOnlySpan<T> storage, out ImmutableHashSet<T> set); 
    public static ImmutableHashSet<T> Create<T>(ReadOnlySpan<T> storage);
}
```

// Could live on the type itself:

```c#
[CollectionBuilder(typeof(ImmutableHashSet), Name = "Create")]
public class ImmutableHashSet<T>
{
    // ...
}

public static class ImmutableHashSet
{
    public static ImmutableHashSet<T> Create<T>(ReadOnlySpan<T> storage);
}
```

// Have to ensure that lang supports this, and if you have an array, it picks the latter.
// today it is considered ambiguous :'(

Usage:

```c#
ImmutableHashSet<string> values = ["a", "b", "c"];
```


Translation:

```c#
ReadOnlySpan<string> storage = ["a", "b", "c"];
CollectionsMarshal.Create<string>(storage out  ImmutableHashSet<string> values);
```


```c#
ImmutableArray<int> values = [1, 2, 3];
// compiler could grab a span to a data segment with 1,2,3 in it.
// Still best to call approach that gives us the Span<int> to write into,
// and compiler copies from data segment into that Span.
```

```c#
// variable length 
ImmutableArray<int> values = [1, ..b, 3];
```

translation:

```c#
// build the above on the stack/heap using whatever algorithm we come up with (NO RENTING :)).
CollectionMarshal.Create<int>(capacity: N, out ImmutableArray<int> values, out Span<int> __storage);
Copy(from: __compilerStorage, to: __storage);
```

// ImmutableArray added .Create(ROS<T>) and .Create(Span<T>).  So for consistency, likely makes
// sense to add ImmutableXXX.Create that is similar.  This sidesteps the problem with IEnumerable and Span.

```c#
[AttributeUsage(AttributeTargets.Interface | AttributeTargets.Class | AttributeTargets.Struct))
public class CollectionBuilderAttribute(Type Type, string Name) : System.Attribute
{
}
```

## Pattern 4:

1. Non-fixed length collections.

```c#
static class CollectionsMarshal
{
    public static SBuilder Create<T>(int minCapacity, out ImmutableHashSet<T> set);

    public struct SBuilder
    {
        public void Add(T value);
        public void AddRange(TCollection type);
        public void Complete();
    }
}
```

Usage:

```c#
ImmutableHashSet<string> values = ["a", ..b, "c"];
```

Translation:

```c#
SBuilder builder = CollectionsMarshal.Create<string>(minCapacity: 2, out ImmutableHashSet<string> values);
builder.Add("a");
builder.AddRange(b);
builder.Add("c");
builder.Complete();
```

Alternative: Compiler creates and manages an growable-array it fills, then passes to construction method.  Downside is managing/pooling that array, then the walk of the array to create final form.