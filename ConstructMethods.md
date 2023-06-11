# Construct methods

## Pattern 1:

1. Fixed length collection literals.
2. Collections with contiguous backing storage.
3. Ref-struct possible

```c#
static class CollectionsMarshal
{
    public static Span<T> Create<T>(int capacity, out List<T> list); 
    public static Span<T> Create<T>(int capacity, out ImmutableArray<T> list);
}
```

Usage:

```c#
ImmutableArray<string> values = ["a", "b", "c"];
```

Translation:

```c#
var __storage = CollectionsMarshal.Create<string>(capacity: 3, out ImmutableArray<string> values);
__storage[0] = "a";
__storage[1] = "b";
__storage[2] = "c";
```

## Pattern 2

Fixed length collection literals.  Collections