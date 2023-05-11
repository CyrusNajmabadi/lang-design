# Natural type

In the absence of a constructible collection target type, a collection literal can have a natural type.

The natural type of a collection literal is some `List<T>` where the `T` is determined based in the following fashion:

1. For a non-empty literal `[e_1, ..s_n]` the element type `T` is determined from the best common type of:
    * The type of each element e_n, and
    * The iteration type of each element ..s_n.
2. For an empty literal of the form `var v = [];` the natural type 