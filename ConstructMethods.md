# Construct methods

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
}
```

Usage:

```c#
ImmutableHashSet<string> values = ["a", "b", "c"];
```

Translation:

```c#
ReadOnlySpan<string> storage = ["a", "b", "c"];
CollectionsMarshal.Create<string>(storage out  ImmutableHashSet<string> values);
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