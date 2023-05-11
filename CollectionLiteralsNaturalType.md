# Natural type

In the absence of a constructible collection target type, a collection literal can have a natural type.

The natural type of a collection literal is some `List<T>` where the `T` is determined based in the following fashion:

1. For a non-empty literal `[e_1, ..s_n]` the element type `T` is determined from the best common type of:

    * The type of each element e_n, and
    * The iteration type of each element ..s_n.

1. For an empty literal of the form `var v = [];` the the natural type of `v` is `List<>`.  This is an `inferred generic` type whose type argument is determined by how it is used in the rest of the user's code.

1. `[]` in any other context is illegal

## Inferred generic types

An `inferred generic type` is a type that can be used on a local variable, indicating that the actual instantiation should be determined from code that follows.  For example:

```c#
X<> v;

// X<>'s type argument determined by how 'v' is used later.
```