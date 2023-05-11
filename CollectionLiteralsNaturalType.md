# Natural type

In the absence of a constructible collection target type, a collection literal can have a natural type.

The natural type of a collection literal is some `List<T>` where `T` is determined using the best common type algorithm.

## Non-empty literal

A natural element type T is first determined from the best common type of:

* The type of each element e_n, and
* The iteration type of each element ..s_n.