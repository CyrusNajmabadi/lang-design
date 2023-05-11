# Natural type

In the absence of a constructible collection target type, a collection literal can have a natural type.

The natural type of a collection literal is some `List<T>` where the `T` is determined based on the elements in the literal (if present), or the usage of the collection if not.

1. For a non-empty literal `[e_1, ..s_n]` the element type `T` is determined from the best common type of:

    * The type of each element e_n, and

    * The iteration type of each element ..s_n.

1. For an empty literal of the form `var v = [];` the the natural type of `v` is `List<>`.  This is an [`inferred generic type`](InferredGenericType.md) whose type argument is determined by how `v` is used in the rest of the user's code.

1. `[]` in any other context is illegal


While `List<T>` is generally considered to be a heavier type when compared to alternatives like `T[]` and `Span<T>`. The [list performance](ListPerformance.md) specification covers the optimizations the language and compiler coordinate on to allow the natural type to be `List<T>` while also achieving low overhead and high performance.