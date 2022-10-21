# Collection literals

* [x] Proposed
* [ ] Prototype: Not Started
* [ ] Implementation: Not Started
* [ ] Specification: Not Started

Many thanks to those who helped with this proposal.  Esp. @jnm2!

## Summary
[summary]: #summary

Collection literals introduce a new terse syntax, `[e1, e2, e3, etc]`, to create common collection values.  Inlining other collections into these values is possible using a spread operator `..` like so: `[e1, ..c2, e2, ..c2]`.  A `[k1: v1, ..d1]` form is also supported for creating dictionaries.

Several collection-like types can be created without requiring external BCL support.  These types are:
* [Array types](https://github.com/dotnet/csharplang/blob/main/spec/types.md#array-types), such as `int[]`.
* [`Span<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.span-1) and [`ReadOnlySpan<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.readonlyspan-1).
* Types that support [collection initializers](https://github.com/dotnet/csharplang/blob/main/spec/expressions.md#collection-initializers), such as [`List<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1) and [`Dictionary<TKey, TValue>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2).

Further support is present for collection-like types not covered under the above, such as [`ImmutableArray<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.immutable.immutablearray-1), through a new API pattern that can be adopted directly on the type itself or through extension methods.

## Motivation
[motivation]: #motivation

* Collection-like values are hugely present in programming, algorithms, and especially in the C#/.NET ecosystem.  Nearly all programs will utilize these values to store data and send or receive data from other components. Currently, almost all C# programs must use many different and unfortunately verbose approaches to create instances of such values. Some approaches also have performance drawbacks. Here are some common examples:

    - Arrays, which require either `new Type[]` or `new[]` before the `{ ... }` values.

    - Spans, which may use `stackalloc` and other cumbersome constructs.

    - Collection initializers, which require syntax like `new List<T>` (lacking inference of a possibly verbose `T`) prior to their values, and which can cause multiple reallocations of memory because they use N `.Add` invocations without supplying an initial capacity.

    - Immutable collections, which require syntax like `ImmutableArray.Create(...)` to initialize the values, and which can cause intermediary allocations and data copying. More efficient construction forms (like `ImmutableArray.CreateBuilder`) are unweildy and still produce unavoidable garbage.

* Looking at the surrounding ecosystem, we also find examples everywhere of list creation being more convenient and pleasant to use.  TypeScript, Dart, Swift, Elm, Python, and more opt for a succinct syntax for this purpose, with widespread usage, and to great effect. Cursory investigations have revealed no substantive problems arising in those ecosystems with having these literals built in.

* C# has also added [list patterns](https://github.com/dotnet/csharplang/blob/main/proposals/list-patterns.md) in C# 10.  This pattern allows matching and deconstruction of list-like values using a clean and intuitive syntax.  However, unlike almost all other pattern constructs, this matching/deconstruction syntax lacks the corresponding construction syntax.

* Getting the best performance for constructing each collection type can be tricky. Simple solutions often waste both CPU and memory.  Having a literal form allows for maximum flexibility from the compiler implementation to optimize the literal to produce at least as good a result as a user could provide, but with simple code.  Very often the compiler will be able to do better, and the specification aims to allow the implementation large amounts of leeway in terms of implementation strategy to ensure this.

An inclusive solution is needed for C#. It should meet the vast majority of casse for customers in terms of the collection-like types and values they already have. It should also feel natural in the language and mirror the work done in pattern matching.

This leads to a natural conclusion that the syntax should be like `[e1, e2, e3, e-etc]` or `[e1, ..c2, e2]`, which correspond to the pattern equivalents of `[p1, p2, p3, p-etc]` and `[p1, ..p2, p3]`.

A form for dictionary-like collections is also supported where the elements of the literal are written as `k: v` like `[k1: v1, ..d1]`.  A future pattern form that has a corresponding syntax (like `x is [k1: var v1]`) would be desirable.

## Detailed design
[design]: #detailed-design

The following [grammar](https://github.com/dotnet/csharplang/blob/main/spec/expressions.md#primary-expressions) productions are added:

```diff
primary_no_array_creation_expression
  ...
+ | collection_literal_expression
  ;

+ collection_literal_expression
  : '[' ']'
  | '[' collection_literal_element ( ',' collection_literal_element )* ']'
  ;

+ collection_literal_element
  : expression_element
  | dictionary_element
  | spread_element
  ;

+ expression_element
  : expression
  ;

+ dictionary_element
  : null_coalescing_expression ':' expression
  ;

+ spread_element
  : '..' expression
  ;
```

Collection literals are [target-typed](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.1/target-typed-default.md#motivation) but also have a [*natural type*](#natural-type) in the absence of a *target type*.

### Spec clarifications
[spec-clarifications]: #spec-clarifications

* For brevity, `collection_literal_expression` will be referred to as "literal" in the following sections.
* `expression_element` instances will commonly be referred to as `e1`, `e_n`, etc.
* `dictionary_element` instances will commonly be referred to as `k1: v1`, `k_n: v_n`, etc.
* `spread_element` instances will commonly be referred to as `..s1`, `..s_n`, etc.
* *span type* means either `Span<T>` or `ReadOnlySpan<T>`.
* Literals will commonly be shown as `[e1, ..s1, e2, ..s2, etc]` to convey any number of elements in any order.  Importantly, this form will be used to represent all cases such as:
    - Empty literals `[]`
    - Literals with no `expression_element` in them.
    - Literals with no `spread_element` in them.
    - Literals with arbitrary ordering of any element type.

* In the following sections, examples of literals without a `k: v` element should assumed to not have any `dictionary_element` in them. Any usages of `..s` should be assumed to be a spread of a non-dictionary value.  Sections that refer to dictionary behavior will call that out.

* The *iteration type* of `..s_n` is the type of the *iteration variable* determined as if `s_n` were used as the expression being iterated over in a [`foreach_statement`](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement).

* Variables starting with `__name` are used to represent the results of the evaluation of `name`, stored in a location so that it is only evaluated once.  For example `__e1` is the evaluation of `e1`.

* `List<T>`, `Dictionary<TKey, TValue>` and `KeyValuePair<TKey, TValue>` refer to the respective types in the `System.Collections.Generic` namespace.

* The specification defines a [translation](#collection-literal-translation) of the literal to existing C# constructs.  Similar to the [`query expression translation`](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11173-query-expression-translation), the literal is itself only legal if the translation would result in legal code.  The purpose of this rule is to avoid having to repeat other rules of the language that are implied (for example, about convertibility of expressions when assigned to storage locations).

* An implementation is not required to translate literals exactly as specified below.  Any translation is legal as long as the same result is produced and there are no observable differences (outside of timing) in the production of the result.

    * For example, an implementation could translate literals like `[1, 2, 3]` directly to a `new int[] { 1, 2, 3 }` expression that itself bakes the raw data into the assembly, eliding the need for `__index` or a sequence of instructions to assign each value. Importantly, this does mean if any step of the translation might cause an exception at runtime that the program state is still left in the state indicated by the translation.

    * Similarly, while a collection literal has a *natural type* of `List<T>`, it is permissable to avoid such an allocation if the result would not be observable.  For example, `foreach (var toggle in [true, false])`.  Because the elements are all that the user's code can refer to, the above could be optimized away into a direct stack allocation.

* Collections are assumed to be well-behaved.  For example:

    * It is assumed that the value of `Count` on a collection will produce that same value as the number of elements when enumerated.

    * The types used in this spec defined in the `System.Collections.Generic` namespace are presumed to be side-effect free.  As such, the compiler can optimize scenarios where such types might be used as intermediary values, but otherwise not be exposed.

    * The behavior of collection literals with collections that are not well-behaved is undefined.

## Constructible collection types
[constructible-collection-types]: #constructible-collection-types

The following types can be constructed using a collection literal.  Actual translation of the literal to the corresponding is defined [below](#collection-literal-translation).

* Single dimensional arrays (e.g. `T[]`).
* Spans (e.g. `Span<T>`)
* Types with a suitable [`Construct` method](#construct-methods).
* Instantiations of any interface type `I<T>` implemented by `List<T>`.  For example, `IEnumerable<T>`, `IList<T>`, `IReadOnlyList<T>`.
* Instantiations of any interface type `I<TKey, TValue>` implemented by `Dictionary<TKey, TValue>`.  For example, `IDictionary<TKey, TValue>`, `IReadOnlyDictionary<TKey, TValue>`.
* Types that support [`collection initializers`](https://github.com/dotnet/csharpstandard/blob/draft-v7/standard/expressions.md#117154-collection-initializers).

`List<T>` and `Dictionary<TKey, TValue>` are both constructible by virtue of them both supporting `collection initializers`.


## `Construct` methods
[construct-methods]: #construct-methods

While certain types (like arrays and spans) can always be constructed with a collection literal, an arbitrary type `T` can support being be constructed from a collection literal through the use of a `void Construct(CollectionType)` method when:

* the `Construct` method is found on an instance of `T` (including through [extension methods](https://github.com/dotnet/csharpstandard/blob/draft-v7/standard/expressions.md#11783-extension-method-invocations)), and

* `CollectionType` is some other type known to be a [`constructible`](constructible-collection-types) type.

If found, the collection can be constructed by creating a fresh instance of its type using `new T()`, producing the corresponding argument to pass to `Construct`, and then calling that method on the fresh instance.  `new T()` supports all structs, including those without a `parameterless struct constructor`.

The allowance for extension methods means that collection literal support can be added to a existing API which does not already directly support this.

Through the use of the [`init`](#init-methods) modifier, existing APIs can directly support collection literals in a manner that allows for no-overhead production of the data the final collection will store.

### `init Construct` methods
[init-methods]: #init-methods

* Like [`init accessors`](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-9.0/init.md#init-only-setters), an `init` method would be an instance method invocable at the point of object creation but become unavailable once object creation has completed. This facility thus prevents general use of such a marked method outside of known safe compiler scopes where the instance value being constructed cannot be observed until complete.

* In the context of collection literals, using the `init` modifier on the [`Construct` method](#construct-methods) would allow types to trust that the collection instances passed into them cannot be mutated outside of them, and that they are being passed ownership of the collection instance.  This would negate any need to copy data that would normally be assumed to be in an untrusted location.

* For example, if an `init void Construct(T[] values)` instance method were added to [`ImmutableArray<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.immutable.immutablearray-1), then it would be possible for the compiler to emit the following:

    ```c#
    T[] __storage = /* initialized using predefined rules */
    ImmutableArray<T> __result = new ImmutableArray<T>();
    __result.Construct(__storage);
    ```

    `ImmutableArray<T>` would then take that array directly and use it as its own backing storage.  This would be safe because the compiler (following the requirements around `init`) would ensure that no other location in the code would have access to this temporary array, and thus it would not be possible to mutate it behind the back of the `ImmutableArray<T>` instance.

    The above also demonstrates that this approach can work with struct types which do not have a [`parameterless struct constructor`](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-10.0/parameterless-struct-constructors.md).  In the above, the call to `new ImmutableArray<T>()` is equivalent to `default(ImmutableArray<T>)`, (producing an `ImmutableArray<T>` whose [`IsDefault`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.immutable.immutablearray-1.isdefault) property is initially true.  However, the `Construct` method can then safely update this to the final non-default state without that intermediate state being visible.

* This formalization is quite beneficial because the only existing mechanism to (safely) create an ImmutableArray with values without copying is both excessively verbose and produces unavoidable garbage:

    ```c#
    var __builder = ImmutableArray.CreateBuilder<int>(initialCapacity: __len);

    __builder.Add(e1);
    foreach (var __t in s1)
        __builder.Add(__t);

    // Add remainder of values.

    // Create final result. __builder is now garbage.
    ImmutableArray<int> __result = __builder.MoveToImmutable();
    ```

## Empty collection literal

* In the absence of a *target type*, the empty literal `[]` has no type.  However, similar to the [`null-literal`](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/lexical-structure.md#6457-the-null-literal), this literal can be converted to any [`constructible`](#constructible-collection-types) collection type.

    For example, the following is not legal as there is no *target type* and there are no other conversions involved:

    ```c#
    var v = []; // illegal
    ```

    However, the following is allowed because of the use of conversions between the branches in the conditional expression:

    ```c#
    bool b = ...
    var v = b ? [1, 2, 3] : [];
    ```

    In this case the type of the empty literal will be `List<int>` due to the [*natural type*](#natural-type) of `[1, 2, 3]`.

* Spreading an empty literal is permitted to be elided.  For example:

    ```c#
    bool b = ...
    var v = [x, y, .. b ? [1, 2, 3] : []];
    ```

    Here, if `b` is false, it is not required that any value actually be constructed for the empty literal since it would immediately be spread into zero values in the final literal.


## Natural type
[natural-type]: #natural-type

In the absence of a *target type*, a non-empty literal can have a *natural type*.

The *natural type* is determined using the [*best common type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#116315-finding-the-best-common-type-of-a-set-of-expressions) algorithm.

A [*natural element type*](#natural-element-type) `T` is first determined.  If that cannot be determined, the literal has no *natural type*.  If `T` can be determined and it is some `KeyValuePair<TKey, TValue>`, then the *natural type* of the collection is `Dictionary<TKey, TValue>`; otherwise, the *natural type* of the collection is `List<T>`.

This means there is no way for a literal to have a *natural type* of some `List<KeyValuePair<TKey, TValue>>` (though it certainly can be *target-typed* to that type).

### Natural Element Type
[natural-element-type]: #natural-element-type

Computing the *natural element type* starts with three sets of types and expressions called `dictionary key set`, `dictionary value set`, and `remainder set`.

The `dictionary key/value set` sets will either both be empty or both be non-empty.

Each element of the literal is examined in the following fashion:

* An element `e_n` has its *type* determined.  If that type is some `KeyValuePair<TKey, TValue>`, then `TKey` is added to `dictionary key set` and `TValue` is added to `dictionary value set`.  Otherwise, the `e_n` *expression* is added to `remainder set`.

* An element `..s_n` has its *iteration type* determined.  If that type is some `KeyValuePair<TKey, TValue>`, then `TKey` is added to `dictionary key set` and `TValue` is added to `dictionary value set`. Otherwise, the *iteration type* is added to `remainder set`.

* An element `k_n: v_n` adds the `k_n` and `v_n` *expressions* to `dictionary key set` and `dictionary value set` repectively.

* If the `dictionary key/value set` sets are empty, then there were definitely no `k_n: v_n` elements. In that case, the *fallback case* runs below.

* If `dictionary key/value set` sets are non-empty, then a first round of the *best common type* algorithm in performed on those sets to determine `BCT_Key` and `BCT_Value` respectively.

    * If the first round fails for either set, the *fallback case* runs below.

    * If the first rounds succeeds for both sets, there is a `KeyValuePair<BCT_Key, BCT_Value>` type produced.  This type is added to `remainder set`.  A second round of the *best common type* algorithm is performed on this set to determine `BCT_Final`.

        * If the second round fails, the *fallback* case runs below.
        * If the second round succeds, `BCT_Final` is the *natural element type*.
        * The algorithm ends.

* The *fallback case*:

    * If there are no `k_n: v_n` elements:

        * All `e_n` *expressions* are added to `remainder set`
        * All `..s_n` *iteration types* are added to `remainder set`
        * The *natural element type* is the *best common type* of the `remainder set`.
        * The algorithm ends.

    * Otherwise, there is no *natural element type*.

---

* For example, given:

    ```c#
    string i = ...;
    object[] objects = ...;
    var x = [i, ..objects];
    ```

    The *natural type* of `x` is `List<T>` where `T` is the *best common type* of `i` and the *iteration type* of `objects`.  Respectively, that would be the *best common type* between `string` and `object`, which would be `object`.  As such, the type of `x` would be `List<object>`.

* Given:

    ```c#
    Dictionary<string, object> d1 = ...;
    Dictionary<object, string> d2 = ...;
    var d3 = [..d1, ..d2];
    ```

    The *natural type* of `d3` is `Dictionary<object, object>`.  This is because the `..d1` will have a *iteration type* of `KeyValuePair<string, object>` and `..d2` will have a *iteration type* of `KeyValuePair<object, string>`. These will contribute `{string, object}` to the determination of the `TKey` type and `{object, string}` to the determination of the `TValue` type.  In both cases, the *best common type* of each of these sets is `object`.

* Given:

    ```c#
    var d = [null: null, "a": "b"];
    ```

    The *natural type* of `d` is `Dictionary<string, string>`.  This is because the `k_n: v_n` elements will construct the set `{null, "a"}` for the determination of the `TKey` type and `{null, "b"}` to the determination of the `TValue` type.  In both cases, the *best common type* of each of these sets is `string`.

* Given:

    ```c#
    var values = x ? [1, 2, 3] : [];
    ```

    The *best common type* between `[1, 2, 3]` and `[]` causes `[]` to take on the type `[1, 2, 3]`, which is `List<int>` as per the existing *natural type* rules. As this is a constructible collection literal type, `[]` is treated as being target-typed to that collection type.

* Given:

    ```c#
    string s1, s2;
    object o1, o2;
    var d = [s1: o1, o2: s2];
    ```

    The *natural type* of `d3` is `Dictionary<object, object>`.  This is because the `k_n: v_n` elements will construct the set `{s1, o1}` for the determination of the `TKey` type and `{o2, s2}` to the determination of the `TValue` type.  In both cases, the *best common type* of each of these sets is `object`.

* Given:

    ```c#
    string s1, s2;
    object o1, o2;
    var d = [KeyValuePair.Create(s1, o1), KeyValuePair.Create(o2, s2)];
    ```

    The *natural type* of `d3` is `Dictionary<object, object>`.  This is because the `e_n` elements are `KeyValuePair<string, object>` and `KeyValuePair<object, string>` respectively.  These will construct the set `{string, object}`for the determination of the `TKey` type and `{object, string}` to the determination of the `TValue` type.  In both cases, the *best common type* of each of these sets is `object`.

## Span types
[span-types]: #span-types


## Collection literal translation
[collection-literal-translation]: #collection-literal-translation

* The types of each `spread_element` expression are examined to see if they contain an accessible instance `int Length { get; }` or `int Count { get; }` property in the same fashion as [list patterns](https://github.com/dotnet/csharplang/blob/main/proposals/list-patterns.md).  
If they all have such a property, the literal is considered to have a *known length*.

    * In examples below, references to `.Count` refer to this computed length, however it was obtained.

    * A literal without any `spread_element` expressions has *known length*.

    * If at least one `spread_element` can not have its count of elements determined, then the literal is considered to have an *unknown length*.

    * Each `spread_element` can have a different type and a different `Length` or `Count` property than the other elements.

    * Having a *known length* does not affect what collections can be created.  It only affects how efficiently the construction can happen. For example, a *known length* literal is statically guaranteed to efficiently create an array or span at runtime.  Specifically, allocating the precise storage needed, and placing all values in the right location once.

* A literal without a *known length* does not have a guarantee around efficient construction.  However, such a literal may still be efficient at runtime.  For example, the compiler is free to use helpers like [`TryGetNonEnumeratedCount(IEnumerable<T>, out int count)`](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.trygetnonenumeratedcount) to determine *at runtime* the capacity needed for the constructed collection.  As above, in examples below, references to `.Count` refer to this computed length, however it was obtained.

* All elements expressions are evaluated left to right (similar to [array_creation_expression](https://github.com/dotnet/csharplang/blob/main/spec/expressions.md#array-creation-expressions)).  These expressions are only evaluated once and any further references to them will refer to the result of that evaluation.

* Evaluation of the element expressions happens entirely first.  Only after all those evaluations happen are calls to `Count` (or `Length` or `TryGetNonEnumeratedCount`) and all enumerations made.

* Certain translations below attempt to find a suitable `Add` method by which to add either `expression_element` or `spread_element` members to the collection.  If such an `Add` method cannot be found *and* the value being added is some `KeyValuePair<,>` `__kvp`, then the translation will instead try to emit `__result[__kvp.Key] = __kvp.Value;`.


<!--
* When evaluating a `spread_element`, the evaluation should happen with a target-type equivalent to the type of the collection being produced.  If such a evaluation is not allowed, then the evaluation should happen using the natural type of the `spread_element`.  This difference can be demonstrated with:

    ```c#
    Span<int> span = [a, ..b ? [c, d] : [], f];
    ```
-->

### Interface translation
[interface-translation]: #interface-translation

Given a target type `T` for a literal:

* If `T` is some interface `I<TKey, TValue>` where that interface is implemented by `Dictionary<TKey, TValue>`, then the literal is translated as:

    ```c#
    Dictionary<TKey, TValue> __temp = [...]; /* standard translation */
    I<TKey, TValue> __result = __temp;
    ```

* If `T` is some interface `I<T1>` where that interface is implemented by `List<T1>`, then the literal is translated as:

    ```c#
    List<T1> __temp = [...]; /* standard translation */
    I<T1> __result = __temp;
    ```

In other words, the translation works by using the specified rules with the concrete `List<T>` or `Dictionary<TKey, TValue>` types as the target type.  That translated value is then implicitly converted to the resultant interface type.

### Known length translation
[known-length-translation]: #known-length-translation

Having a *known length* allows for efficient construction of a result with the potential for no copying of data and no unnecessary slack space in a result.

Not having a *known length* does not prevent any result from being created. However, it may result in extra CPU and memory costs producing the data, then moving to the final destination.

* For a *known length* literal `[e1, k1: v1, ..s1, e2, k2: v2, ..s2, etc]`, the translation first starts with the following:

    ```c#
    int __len = count_of_expression_elements +
                count_of_dictionary_elements +
                __s1.Count;
                ...
                __s_n.Count;
    ```

* Given a target type `T` for that literal:

    * If `T` is some `T1[]`, then the literal is translated as:

        ```c#
        T1[] __result = new T1[__len];
        int __index = 0;

        __result[__index++] = __e1;
        __result[__index++] = new T1(__k1, __v1);
        foreach (T1 __t in __s1)
            __result[__index++] = __t;

        // further assignments of the remaining elements
        ```

        In this translation, `dictionary_element` is only supported if `T1` is some `KeyValuePair<,>`.

    *  If `T` is some `Span<T1>`, then the literal is translated as the same as above, except that the `__result` initialization is translated as:

        ```c#
        Span<T1> __result = stackalloc T1[__len];

        // same assignments as the array translation
        ```

    * If `T` is some `ReadOnlySpan<T1>`, then the literal is translated the same as for the `Span<T1>` case except that the final result will be that `Span<T1>` [implicitly converted](https://learn.microsoft.com/en-us/dotnet/api/system.span-1.op_implicit#system-span-1-op-implicit(system-span((-0)))-system-readonlyspan((-0))) to a `ReadOnlySpan<T1>`.

    The above forms (for arrays and spans) are the base representations of the literal value and are used for the following translation rules.

    * If `T` supports [object creation](https://github.com/dotnet/csharplang/blob/main/spec/expressions.md#object-creation-expressions), then [member lookup](https://github.com/dotnet/csharplang/blob/main/spec/expressions.md#member-lookup) on `T` is performed to find an accessible `void Construct(T1 values)` method. If found, and if `T1` is a [*constructible*](#constructible-collection-types) collection type, then the literal is translated as:

        ```c#
        // Generate __storage using existing rules.
        T1 __storage = [...];

        T __result = new T();
        __result.Construct(__storage);
        ```

    * If `T` supports [collection initializers](https://github.com/dotnet/csharplang/blob/main/spec/expressions.md#collection-initializers), then:

        * if the type `T` contains an accessible constructor with a single parameter `int capacity`, then the literal is translated as:

            ```c#
            T __result = new T(capacity: __len);

            __result.Add(__e1);
            __result[__k1] = __v1;
            foreach (var __t in __s1)
                __result.Add(__t);

            // further additions of the remaining elements
            ```

            Note: the name of the parameter is required to be `capacity`.

            This form allows for a literal to inform the newly constructed type of the count of elements to allow for efficient allocation of internal storage.  This avoids wasteful reallocations as the elements are added.

        * otherwise, the literal is translated as:

            ```c#
            T __result = new T();

            __result.Add(__e1);
            __result[__k1] = __v1;
            foreach (var __t in __s1)
                __result.Add(__t);

            // further additions of the remaining elements
            ```

            This allows creating the target type, albeit with no capacity optimization to prevent internal reallocation of storage.

### Unknown length translation
[unknown-length-translation]: #unknown-length-translation

* Given a target type `T` for an *unknown length* literal:

    * If `T` supports [collection initializers](https://github.com/dotnet/csharplang/blob/main/spec/expressions.md#collection-initializers), then the literal is translated as:

        ```c#
        T __result = new T();

        __result.Add(__e1);
        __result[__k1] = __v1;
        foreach (var __t in __s1)
            __result.Add(__t);

            // further additions of the remaining elements
        ```

        This allows spreading of any iterable type, albeit with the least amount of optimization possible.

    * If `T` is some `T1[]`, then the literal has the same semantics as:

        ```c#
        List<T1> __list = [...]; /* initialized using predefined rules */
        T1[] __result = __list.ToArray();
        ```

        The above is inefficient though; it creates the intermediary list, and then creates a copy of the final array from it.  Implementations are free to optimize this away, for example producing code like so:

        ```c#
        T1[] __result = <private_details>.CreateArray<T1>(
            count_of_expression_elements + count_of_dictionary_elements);
        __result = 0;

        <private_details>.Add(ref __result, __index++, __e1);
        <private_details>.Add(ref __result, __index++, new T1(__k1, __v1));
        foreach (var __t in __s1)
            <private_details>.Add(ref __result, __index++, __t);

            // further additions of the remaining elements

        <private_details>.Resize(ref __result, __index);
        ```

        This allows for minimal waste and copying, without additional overhead that library collections might incur.

        The counts passed to `CreateArray` are used to provide a starting size hint to prevent wasteful resizes.

## Unsupported scenarios
[unsupported-scenarios]: #unsupported-scenarios

While collection literals can be used for many scenarios, there are a few that they are not capable of replacing.  These include:

* Multi-dimensional arrays (e.g. `new int[5, 10] { ... }`). There is no facility to include the dimensions, and all collection literals are either linear or map structures only.

* Collections which pass special values to their constructors.  For example `new Dictionary<string, object>(CaseInsensitiveComparer.Instance)`.  There is no facility to access the constructor being used in either target or natural-typing scenarios.

* Nested collection initializers, e.g. `new Widget { Children = { w1, w2, w3 } }`.  This form needs to stay since it has very different semantics from `Children = [w1, w2, w3]`.  The former calls `.Add` repeatedly on `.Children` while the latter would assign a new collection over `.Children`.  We could consider having the latter form fall back to adding to an existing collection if `.Children` can't be assigned, but that seems like it could be extremely confusing.

## Syntax ambiguities
[syntax-ambiguities]: #syntax-ambiguities

* There are two "true" syntactic ambiguities where there are multiple legal syntactic interpretations of code that uses a `collection_literal_expression`.

    * The `spread_element` is ambiguous with a [`range_expression`](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-8.0/ranges.md#systemrange).  One could technically have:

        ```c#
        Range[] ranges = [range1, ..e, range2];
        ```

        To resolve this, we can either:

        * Require users to parenthesize `(..e)` or include a start index `0..e` if they want a range.
        * Choose a different syntax (like `...`) for spread.  This would be unfortunate for the lack of consistency with slice patterns.

    * `dictionary_element` can be ambiguous with a [`conditional_expression`](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#1115-conditional-operator).  For example:

        ```c#
        var v = [ a ? [b] : c ];
        ```

        This could be interpreted as `expression_element` where the `expression` is a `conditional_expression` (e.g. `[ (a ? [b] : c) ]`).  Or it could be interpreted as a `dictionary_element` `"k: v"` where `a?[b]` is `k`, and `c` is `v`.

* There are two cases where there isn't a true ambiguity but where the syntax greatly increases parsing complexity.  While not a problem given engineering time, this does still increase cognitive overhead for users when looking at code.

    * Ambiguity between `collection_literal_expression` and `attributes` on statements or local functions.  Consider:

        ```c#
        [X(), Y, Z()]
        ```

        This could be one of:

        ```c#
        // A list literal inside some expression statement
        [X(), Y, Z()].ForEach(() => ...);

        // The attributes for a statement or local function
        [X(), Y, Z()] void LocalFunc() { }
        ```

        Without complex lookahead, it would be impossible to tell without consuming the entirety of the literal.

        Options to address this include:

        * Allow this, doing the parsing work to determine which of these cases this is.

        * Disallow this, and require the user wrap the literal in parentheses like `([X(), Y, Z()]).ForEach(...)`.

    * Ambiguity between a `collection_literal_expression` in a `conditional_expression` and a `null_conditional_operations`.  Consider:

        ```c#
        M(x ? [a, b, c]
        ```

        This could be one of:

        ```c#
        // A ternary conditional picking between two collections
        M(x ? [a, b, c] : [d, e, f]);

        // A null conditional safely indexing into 'x':
        M(x ? [a, b, c]);
        ```

        Without complex lookahead, it would be impossible to tell without consuming the entirety of the literal.

        Note: this is a problem even without a *natural type* because target typing applies through `conditional_expressions`.

        As with the others, we could require parentheses to disambiguate.  In other words, presume the `null_conditional_operation` interpretation unless written like so: `x ? ([1, 2, 3]) :`.  However, that seems rather unfortunate. This sort of code does not seem unreasonable to write and will likely trip people up.

## Drawbacks
[drawbacks]: #drawbacks

* This introduces [yet another form](https://xkcd.com/927/) for collection expressions on top of the myriad ways we already have. This is extra complexity for the language.  That said, this also makes it possible to unify on one ~~ring~~ syntax to rule them all, which means existing codebases can be simplified and moved to a uniform look everywhere.

* Using `[`...`]` instead of `{`...`}` moves away from the syntax we've generally used for arrays and collection initializers already.  Specifically that it uses `[`...`]` instead of `{`...`}`.  However, this was already settled on by the language team when we did list patterns.  We attempted to make `{`...`}` work with list patterns and ran into insurmountable issues.  Because of this, we moved to `[`...`]` which, while new for C#, feels natural in many programming languages and allowed us to start fresh with no ambiguity.  Using `[`...`]` as the corresponding literal form is complimentary with our latest decisions, and gives us a clean place to work without problem.

This does introduce warts into the language.  For example, the following are both legal and (fortunately) mean the exact same thing:

```c#
int[] x = { 1, 2, 3 };
int[] x = [ 1, 2, 3 ];
```

However, given the breadth and consistency brought by the new literal syntax, we should consider recommending that people move to the new form.  IDE suggestions and fixes could help in that regard.

## Alternatives
[alternatives]: #alternatives

 * What other designs have been considered? What is the impact of not doing this?

## Resolved questions
[resolved]: #resolved-questions

* In what order should we evaluate literal elements compared with Length/Count property evaluation?  Should we evaluate all elements first, then all lengths?  Or should we evaluate an element, then its length, then the next element, and so on?

    Resolution: We evaluate all elements first, then everything else follows that.

* Can an *unknown length* literal create a collection type that needs a *known length*, like an array, span, or Construct(array/span) collection?  This would be harder to do efficiently, but it might be possible through clever use of pooled arrays and/or builders.

    Resolution: Yes, we allow creating a fixes-length collection from an *unknown length* literal.  The compiler is permitted to implement this in as efficient a manner as possible.

    The following text exists to record the original discussion of this topic.

    <details>

    Users could always make an *unknown length* literal into a *known length* one with code like this:

    ```c#
    ImmutableArray<int> x = [a, ..unknownLength.ToArray(), b];
    ```

    However, this is unfortunate due to the need to force allocations of temporary storage.  We could potentially be more efficient if we controlled how this was emitted.

    </details>

* Should a `collection_literal_expression` have a *natural type*?  In other words, should it be legal to write the following:

    ```c#
    var x = [1, 2, 3];
    ```

    Resolution: Yes, the *natural type* will be an appropriate instantiation of `List<T>`. The following text exists to record the original discussion of this topic.

    <details>

    It is virtually certain that users will want to do this.  However, there is much less certainty both on what users would want this mean and if there is even any sort of broad majority on some default.  There are numerous types we could pick, all of which have varying pros and cons.  Specifically, our options are *at least* any of the following:

    * Array types
    * Span types
    * `ImmutableArray<T>`
    * `List<T>`
    * [`ValueArray<T, N>`](https://github.com/dotnet/roslyn/pull/57286)

    Each of those options has varying benefits with respect to the following questions:

    * Will the literal cause a heap allocation (and, if so, how many), or can it live on the stack?
    * Are the values of the literal mutable after creation or are they fixed?
    * Is the resultant value itself mutable (e.g. can it be cleared, or can new elements be added to it)?
    * Can the value be used in all contexts (for example, async/non-async)?
    * Can be used for *all* literal forms (for example, a `spread_element` of an *unknown length*)?

    Note: for whatever type we pick as a *natural type*, the user can always target-type to the type they want with a simple cast, though that won't be pleasant.

    With all of that, we have a matrix like so:

    | type | heap allocs | mutable elements | mutable collection | async | all literal forms |
    |-|-|-|-|-|-|
    | `T[]` | 1 | Yes | No | Yes | No* |
    | `Span<T>` | 0 | Yes | No | No | No* |
    | `ReadOnlySpan<T>` | 0 | No | No | No | No* |
    | `List<T>` | 2 | Yes | Yes | Yes | Yes |
    | `ImmutableArray<T>` | 1 | No | No | Yes | No* |
    | `ValueArray<T, N>` | ? | ? | ? | ? | ? |

    \* `T[]`, `Span<T>` and `ImmutableArray<T>` might potentially work for 'all literal forms' if we extend this spec greatly with some sort of builder mechanism that allows us to tell it about all the pieces, with a final `T[]` or `Span<T>` obtained from the builder which can also then be passed to the `Construct` method used by *known length* translation in order to support `ImmutableArray<T>` and any other collection.

    Only `List<T>` gives us a `Yes` for all columns. However, getting `Yes` for everything is not necessarily what we desire.  For example, if we believe the future is one where immutable is the most desirable, the types like `T[]`, `Span<T>`, or `List<T>` may not compliment that well.  Similarly if we believe that people will want to use these without paying for allocations, then `Span<T>` and `ReadOnlySpan<T>` seem the most viable.

    However, the likely crux of this is the following:

    * Mutation is part and parcel of .NET
    * `List<T>` is already heavily the lingua franca of lists.
    * `List<T>` is a viable final form for any potential list literal (including those with spreads of *unknown length*)
    * Spans types and ValueArray are too esoteric, and the inability to use ref structs within async-contexts is likely a deal breaker for broad acceptance.

    As such, while it unfortunate that it has two allocations, `List<T>` seems be the most broadly applicable. This is likely what we would want from the *natural type*.

    I believe the only other reasonable alternative would be `ImmutableArray<T>`, but either with the caveat that that it cannot support `spread_elements` of *unknown length*, or that we will have to add a fair amount of complexity to this specification to allow for some API pattern to allow it to participate.  That said, we should strongly consider adding that complexity if we believe this will be the recommended collection type that we and the BCL will be encouraging people to use.

    Finally, we could consider having different *natural types* in different contexts (like in an async context, pick a type that isn't a ref struct), but that seems rather confusing and distasteful.

    </details>


* Can a `collection_literal_expression` be target-typed to an `IEnumerable<T>` or other collection interfaces?

    For example:

    ```c#
    void DoWork(IEnumerable<long> values) { ... }
    // Needs to produce `longs` not `ints` for this to work.
    DoWork([1, 2, 3]);
    ```

    Resolution: Yes, a literal can be target-typed to any interface type `I<T>` that `List<T>` implements.  For example, `IEnumerable<long>`. This is the same as target-typing to `List<long>` and then assigning that result to the specified interface type. The following text exists to record the original discussion of this topic.

    <details>

    If `collection_literal_expression` is not target-typed to an `IEnumerable<T>`, then its *natural type* of `List<T>` allows it to be assigned to a compatible `IEnumerable<T>`. This would disallow `IEnumerable<long> x = [1, 2, 3];` since `List<int>` is not assignable to `IEnumerable<long>`. This feels like it will come up. For example:

    ```c#
    void DoWork(IEnumerable<long> values) { ... }
    // ...
    DoWork([1, 2, 3]);
    ```

    Considering the case of the element types matching (both being `int`):

    ```c#
    void DoWork(IEnumerable<int> values) { ... }
    // ...
    DoWork([1, 2, 3]);
    ```

    The open question here is determining what underlying type to actually create.  One option is to look at the proposal for [`params IEnumerable<T>`](https://github.com/dotnet/csharplang/issues/179).  There, we would generate an array to pass the values along, similar to what happens with `params T[]`.

    A downside to using an array would be if a *natural type* is added for collection literals and that *natural type* is not `T[]`. There would be a potentially surprising difference when refactoring between `var x = [1, 2, 3];` and `IEnumerable<int> x = [1, 2, 3];`.

    </details>


* How would we proceed on this in the future to get dictionary literals?

    Resolution: The form `[k: v]` is supported for dictionary literals.  Dictionary literals also support spreading (e.g. `[k: v, ..d]`) The following text exists to record the original discussion of this topic.

    <details>

    This is a complex space as we have multiple forms for dictionaries today.  For example:

    ```c#
    var x = new Dictionary<int, string>
    {
        { 1, "x" },
        { 2, "y" },
    };
    ```

    And

    ```c#
    var x = new Dictionary<int, string>
    {
        [1] = "x",
        [2] = "y",
    }
    ```

    Immutable dictionaries in particular motivate doing something in this space, with the syntax that makes generic type inference possible looking like this:

    ```cs
    M(ImmutableDictionary.CreateRange(new[]
    {
        KeyValuePair.Create(1, "x"),
        KeyValuePair.Create(2, "y"),
    }));
    ```

    Would we want a syntax that draws parallels with object or collection initializers?  Or would we want something similar to these collection literals?  Options include (but are not limited to):

    - `Dictionary<int, string> x = { { 1, "x" }, { 2, "y" } };`
    - `Dictionary<int, string> x = { [1] = "x", [2] = "y" };`
    - `Dictionary<int, string> x = [ { 1, "x" }, { 2, "y" } ];`
    - `Dictionary<int, string> x = [1:"x", 2:"y"];`
    - etc.

    </details>


## Unresolved questions
[unresolved]: #unresolved-questions

Hopefully small questions:

* Should it be legal to create and immediately index into a collection literal?  Note: this requires an answer to the unresolved question below of whether collection literals have a *natural type*.

* Stack allocations for huge collections might blow the stack.  Should the compiler have a heuristic for placing this data on the heap?  Should the language be unspecified to allow for this flexibility?  We should follow what the spec/impl does for [`params Span<T>`](https://github.com/dotnet/csharplang/issues/1757).

* Should we expand on collection initializers to look for the very common `AddRange` method? It could be used by the underlying constructed type to perform adding of spread elements potentially more efficiently.  We might also want to look for things like `.CopyTo` as well.  There may be drawbacks here as those methods might end up causing excess allocations/dispatches versus directly enumerating in the translated code.

* Do we need to target-type `spread_element`?  Consider, for example:

    ```c#
    Span<int> span = [a, ..b ? [c] : [d, e], f];
    ```

    Note: this may commonly come up in the following form to allow conditional inclusion of some set of elements, or nothing if the condition is false:

    ```c#
    Span<int> span = [a, ..b ? [c, d, e] : [], f];
    ```

    In order to evaluate this full literal, we need to evaluate the element expressions within.  That means being able to evaluate `b ? [c] : [d, e]`.  However, absent a target type to evaluate this expression in the context of, and absent any sort of *natural type*, this would we would be unable to determine what to do with either `[c]` or `[d, e]` here.

    To resolve this, we could say that when evaluating a literal's `spread_element` expression, there was an implicit target type equivalent to the target type of the literal itself.  So, in the above, that would rewritten as:

    ```c#
    int __e1 = a;
    Span<int> __s1 = b ? [c] : [d, e];
    int __e2 = f;

    Span<int> __result = stackallock int[2 + __s1.Length];
    int __index = 0;

    __result[__index++] = a;
    foreach (int __t in __s1)
        __result[index++] = __t;
    __result[__index++] = f;

    Span<int> span = __result;
    ```

## Design meetings
[design-meetings]: #design-meetings

https://github.com/dotnet/csharplang/blob/main/meetings/2021/LDM-2021-11-01.md#collection-literals
https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-03-09.md#ambiguity-of--in-collection-expressions
https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-09-28.md#collection-literals

## Working group meetings
[working-group-meetings]: #working-group-meetings

https://github.com/dotnet/csharplang/blob/main/meetings/working-groups/collection-literals/CL-2022-10-06.md
https://github.com/dotnet/csharplang/blob/main/meetings/working-groups/collection-literals/CL-2022-10-14.md

## Upcoming agenda items

* Stack allocations for huge collections might blow the stack.  Should the compiler have a heuristic for placing this data on the heap?  Should the language be unspecified to allow for this flexibility?  We should follow what the spec/impl does for [`params Span<T>`](https://github.com/dotnet/csharplang/issues/1757). Options are:

    * Always stackalloc.  Teach people to be careful with Span.  This allows things like `Span<T> span = [1, 2, ..s]` to work, and be fine as long as `s` is small.  If this could blow the stack, users could always create an array instead, and then get a span around this.  This seems like the most in line with what people might want, but with extreme danger.

    * Only stackalloc when the literal has a *fixed* number of elements (i.e. no spread elements).  This then likely makes things always safe, with fixed stack usage, and the compiler (hopefully) able to reuse that fixed buffer.  However, it means things like `[1, 2, ..s]` would never be possible, even if the user knows it is completely safe at runtime.

* Creating an `ImmutableDictionary<,>` is not efficient with any of the translations exposed today.  The only translation that can create them would be:

    ```c#
    KeyValuePair<,>[] __temp = ...; // initialized with all values
    ImmutableDictionary<,> __result = new ImmutableDictionary<,>();
    __result.Construct(__temp);
    ```

    This has the benefit of giving the dictionary all values at once.  But it has the downside of allocating a buffer to just be thrown away.  Is it more desirable to define something like an `init void Add(T value)` method so that the above could be:

    ```c#
    KeyValuePair<,>[] __temp = ...; // initialized with all values
    ImmutableDictionary<,> __result = new ImmutableDictionary<,>();
    __result.Add(__e1);
    foreach (var __t in __s1)
        __result.Add(__t);

    // and so on.
    ```

* How does overload resolution work?  If an API has:

    ```C#
    public void M(T[] values);
    public void M(List<T> values);
    ```

    What happens with `M([1, 2, 3])`?  We likely need to define 'betterness' for these conversions.


* Should we expand on collection initializers to look for the very common `AddRange` method? It could be used by the underlying constructed type to perform adding of spread elements potentially more efficiently.  We might also want to look for things like `.CopyTo` as well.  There may be drawbacks here as those methods might end up causing excess allocations/dispatches versus directly enumerating in the translated code.
