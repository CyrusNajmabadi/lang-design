# Inferred generic types

An `inferred generic type` is a type that can be used on a local variable, indicating that the actual instantiation should be determined from code that follows.  For example:

```c#
X<> v;

// X<>'s type arguments determined by how 'v' is used later
```

Note: The forms of an inferred generic type are `X<>`, `X<,>`, `X<,,>` etc. For the purposes of explanation though all of the above will be represented with `X<>` below.

The rules for inference for the type argument `T` for the `inferred generic type` `X<>` of a local variable `v` work as follows:

1. When 'v' is target-typed to some concrete `X<A>` instantiation, then `T` gets an upper and lower bound of `A`.  For example an inferred generic type `List<>` target-typed to `List<int>` infers `int` for `T`

1. When 'v' is target-typed to some concrete *invariant* `I<A>` interface or class implemented/inherited by `X<A>`, then `T` gets an upper and lower bound of `A`. For example an inferred generic type `List<>` target-typed to `IList<int>` infers `int` for `T`

1. When 'v' is target-typed to some concrete *out variant* `I<out A>` interface implemented by `X<A>`, then `T` gets an upper bound of `X`. For example an inferred generic type `List<>` target-typed to `IEnumerable<object>` infers an upper bound of `object` for `T`.

1. When 'v' is target-typed to some concrete *in variant* `I<in A>` interface implemented by `X<A>`, then `T` gets an lower bound of `X`. For example an inferred generic type `Comp<T>` target-typed to `IComparer<string>` infers an lower bound of `string` for `T`.

1. Method calls (including property accessor methods) on `v` are examined. If those methods have arguments that use `T`, `I<out T>`, `I<in T>`, or `T[]` then those arguments add appropriate bounds to inference.  Note: `I<>` refers to both interfaces and delegate types. Also, these interfaces/delegates can have multiple type parameters.  These rules only apply to the usage of the `T` type parameter in the in that case.   For example:
    1. `v.Add("")` adds a lower bound of `string`.  `T` case.

    1. `v.AddRange(ienumerableOfStrings)` adds a lower bound of `string`. `I<out T>` case.
    1. `v.Sort(icomparerOfObject)` adds an upper bound of `object`. `I<in T>` case.
    1. `v.CopyTo(stringArray)`.  Adds a lower bound of `string`. `T[]` case.

1. Extension method calls count as `target-typing` cases as `v` is passed as an argument to the extension method's `this` parameter. 