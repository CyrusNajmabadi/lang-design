# Construct methods

Pattern 1:

Fixed length collection literals.  Collections with contiguous backing storage:

```c#
static class CollectionsMarshal
{
    public static Span<T> Create<T>(int capacity, out List<T> list); 
    public static Span<T> Create<T>(int capacity, out ImmutableArray<T> list);
}
```