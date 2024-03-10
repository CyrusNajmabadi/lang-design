# Dictionary Expressions

## Summary

*Dictionary Expressions* are a continuation of the C# 12 *Collection Expressions* feature.  They extend that system with a new terse syntax, `["mads": 21, "dustin": 22]`, for creating common dictionary values.  Like with collection expressions, merging other dictionaries into these values is possible using the existing spread operator `..` like so: `[.. currentStudents, "mads": 21, "dustin": 22]`

Several dictionary-like types can created without external BCL support.  These types are:

1. Dictionary-like types, containing an indexer `TValue this[TKey] { get; set; }`, like `Dictionary<TKey, TValue>` and `ConcurrentDictionary<TKey, TValue>`.
1. The well-known BCL dictionary interface types: `IDictionary<TKey, TValue>` and `IReadOnlyDictionary<TKey, TValue>`.

Further support is present for dictionary-like types not covered above through the `CollectionBuilderAttribute` and a similar API pattern to the corresponding *create method* pattern introduced for collection expressions.

## Motivation

While dictionary-values are similar to standard sequential collection-values in that they can be interpreted as a sequence of key/value pairs, they differ in that they are often used for their more fundamental capability of efficient looking up of values based on a provided key.  In an analysis of the BCL and the nuget package ecosystem, sequential-collection types and values make up the lion's share of collections used.  However, dictionary types were still used a significant amount, with appearances in APIs occurring at between 5 and 10% the amount of sequential collections, and with values appearing universally.

Currently, all C# programs must use many different and unfortunately verbose approaches to create instances of such values. Some approaches also have performance drawbacks. Here are some common examples:

1. Collection-initializer types, which require syntax like `new Dictionary<X, Y> { ... }` (lacking inference of possibly verbose TKey and TValue) prior to their values, and which can cause multiple reallocations of memory because they use `N` `.Add` invocations without supplying an initial capacity.
1. Immutable collections, which require syntax like `ImmutableDictionary.CreateRange(...)`, but which is also unpleasant due to the need to provide values as an `IEnumerable<KeyValuePair>`s.  Builders are even more unwieldy.
1. Read-only dictionaries, which require both making a normal dictionary first, then wrapping them.
1. Concurrent-dictionaries, which lack an `.Add` method, and thus cannot be used with normal collection initializers.

Looking at the surrounding ecosystem, we also find examples everywhere of dictionary creation being more convenient and pleasant to use. Swift, TypeScript, Dart, Ruby, Python, and more opt for a succinct syntax for this purpose, with widespread usage, and to great effect. Cursory investigations have revealed no substantive problems arising in those ecosystems with having these literals built in.

Unlike *collection expressions*, C# does not have an existing pattern serving as the corresponding deconstruction form.  Designs here should be made with a consideration for being complimentary with deconstruction work. 

An inclusive solution is needed for C#. It should meet the vast majority of case for customers in terms of the dictionary-like types and values they already have. It should also feel pleasant in the language,  complement the work done with collection expressions, and naturally extend to pattern matching in the future.

## Detailed Design

The following grammar productions are added:

```diff
collection_element
  : expression_element
  | spread_element
+ | dictionary_element  
  ;

+ dictionary_element
+  : expression ':' expression
+  ;
```

Alternative syntaxes are available for consideration, but should be considered later due to the bike-shedding cost involved.  Picking the above syntax allows for the compiler team to move quickly at implementing the semantic side of the feature, allowing earlier previews to be made available.  These syntaxes include, but are not limited to:

1. Using braces instead of brackets.  `{ k1: v1, k2: v2 }`.
2. Using brackets for keys: `[k1] = v1, [k2] = v2`
3. Using arrows for elements: `k1 => v1, k2 => v2`.

Choices here would have implications regarding potential syntactic ambiguities, collisions with potential future language features, and concerns around corresponding pattern forms.  However, all of those should not generally affect the semantics of the feature and can be considered at a later point dedicated to determining the most desirable syntax.

## Design Intuition

Intuitively, *dictionary expressions* work similarly to *collection expressions*, except treating a `k:v` element as a shorthand for creating a `System.Collections.Generic.KeyValuePair<TKey, TValue>`.  Many rules for *dictionary expressions* will correspond to existing rules for *collection expressions*, just requiring aspects such as *element* and *iteration types* to be some `KeyValuePair<,>`.

With a broad interpretation of these rules all of the following would be legal:

```c#
Dictionary<string, int> nameToAge1 = ["mads": 21, existingKvp]; // as would
Dictionary<string, int> nameToAge2 = ["mads": 21, .. existingDict]; // as would
Dictionary<string, int> nameToAge3 = ["mads": 21, .. existingListOfKVPS];
```

### Open Question 1

Should we allow *expression elements* when producing dictionaries?  Or only *dictionary elements* and *spread elements*?  If we do not allow *expression elements* then the following would not be legal:

```c#
Dictionary<string, int> nameToAge = ["mads": 21, existingKvp]; // A user would have to write:

Dictionary<string, int> nameToAge = ["mads": 21, existingKvp.Key: existingKvp.Value];
```

### Open question 2

Having spreads in a *dictionary expression* only be concerned with element types (and not the collection type being spread itself), matches the equivalent case in the collection-expression case:

```c#
List<KeyValuePair<string, int>> nameToAge = [.. someDict]; // supported today in C# 12
```

But we could restrict spreads in a *dictionary expression* to only allow dictionary types themselves.  If we require dictionary types, then the following would not be legal:

```c#
Dictionary<string, int> nameToAge = ["mads": 21, .. existingListOfKVPS];
```

Note: this seems particularly restrictive given that people may commonly use things like linq-expressions to filter and transform dictionaries like so:

```c#
Dictionary<string, int> nameToAge = ["mads": 21, .. existingDict.Where(kvp => kvp.Value >= 21)];
```

### Open question 3

How far do we want to take this KeyValuePair representation of things? Do we allow *dictionary elements* when producing normal collections? For example, should the following be allowed:

```c#
List<KeyValuePair<string, int>> = ["mads": 21];
```

Or should the presence of `k:v` element mandate some dictionary type as the receiver.  Note: the above sort of API is not uncommon.  Roslyn itself contains many apis that permissively *receive* an `IEnumerable<KeyValuePair<string, int>>`.

Importantly, we do not believe it wise to *require* the presence of a `k:v` element to produce a dictionary instance.  For example, we believe it very reasonable and desirable that someone be able to write:

```c#
Dictionary<string, int> everyone = [.. students, .. teachers];
```

### Open question 4

Should we take a very restrictive view of `KeyValuePair<,>`?  Specifically, should we allow only that exact type?  Or should we allow any types with an implicit conversion to that type.  For example:

```c#
struct Pair<X, Y>
{
  public static implicit operator KeyValuePair<X, Y>(Pair<X, Y> pair) => ...;
}

Dictionary<int, string> map1 = [pair1, pair2]; // ?


List<Pair<int, string>> pairs = ...;
Dictionary<int, string> map2 = [.. pairs]; // ?
```

### Open question 5

Dictionaries provide two ways of initializing their contents.  A restrictive `.Add`-oriented form that throws when a key is already present in the dictionary, and a permissive indexer-oriented form which does not.  The restrictive form is useful for catching mistakes ("oops, i didn't intend to add the same thing twice!"), but is limiting *especially* in the spread case.  For example:

```c#
Dictionary<string, Option> optionMap = [opt1Name: opt1Default, opt2Name: opt2Default, .. userProvidedOptions];
```

Or, conversely:

```c#
Dictionary<string, Option> optionMap = [.. Defaults.CoreOptions, feature1Name: feature1Override];
```

Which approach should we go with with our dictionary expressions? Options include:

1. Purely restrictive.  All elements use `.Add` to be added to the list.  Note: types like `ConcurrentDictionary` would then not work, not without adding support with something like the `CollectionBuilderAttribute`.
2. Purely permissive.  All elements are added using the indexer.  Perhaps with compiler warnings if the exact same key is given the same constant value twice.
3. Perhaps a hybrid model.  `.Add` if only using `k:v` and switching to indexers if using spread elements.  Deep potential for confusion here.


## Conversions

> A *collection expression conversion* allows a collection expression to be converted to a type.
>
> An *implicit collection expression conversion* exists from a collection expression to the following types:
>
> - Array rules...
> - Span rules...
>
> - A type with a create method with an iteration type determined from a GetEnumerator instance method or enumerable interface, not from an extension method.
> - ```diff
>   + A type with a create method with an iteration type determined
    + from a GetEnumerator instance method or enumerable interface, not from an extension method, that is some `KeyValuePair<TKey, TValue>` and an argument type `IEnumerable<KeyValuePair<TKey, TValue>`.
> 
>   + For example `public static ImmutableDictionary CreateRange<TKey, TValue>(IEnumerable<KeyValuePair<TKey, TValue>>)`. Note: it is an open question what collection types are supported for the argument type.
>   ```
> - A struct or class type that implements System.Collections.IEnumerable where:
>   - The type has an applicable constructor that can be invoked with no arguments, and the constructor is accessible at the location of the collection expression.
>   - If the collection expression has any elements, the type has an applicable instance or extension method Add that can be invoked with a single argument of the iteration type, and the method is accessible at the location of the collection expression.
>   - ```diff
>     + If the collection expression has any elements and the type has an iteration type of some `KeyValuePair<TKey, TValue>` and the type has applicable indexer that can be invoked with a single argument of the `TKey` type, and a value of the `TValue` type, and the indexer is accessible at the location of the collection expression. 
>     ```
> - An interface type:
>   - System.Collections.Generic.IEnumerable<T>
>   - System.Collections.Generic.IReadOnlyCollection<T>
>   - System.Collections.Generic.IReadOnlyList<T>
>   - System.Collections.Generic.ICollection<T>
>   - System.Collections.Generic.IList<T>
>   - ```diff
>     + System.Collections.Generic.IDictionary<TKey, TValue>
>     + System.Collections.Generic.IReadOnlyDictionary<TKey, TValue>
>     ```

## Create methods

> A create method is indicated with a [CollectionBuilder(...)] attribute on the collection type. The attribute specifies the builder type and method name of a method to be invoked to construct an instance of the collection type.
> ```diff
> + A create method will commonly use the name `CreateRange` in the dictionary domain.
> ```
> For the create method:
>
> ```diff
> The method must have a single parameter of type System.ReadOnlySpan<E>, passed by value, and there is an identity conversion from E to the iteration type of the collection type.
>
> + Or, the method have a single parameter of `IEnumerable<KeyValuePair<TKey, TValue>>` and iteration type of the collection type is the same `KeyValuePair<,>` type. 
> ```

This would allow `ImmutableDictionary<TKey, TValue>` to be annotated with `[CollectionBuilder(typeof(ImmutableDictionary), "CreateRange")]` to light up support for creation.

### Open question 1

We could consider not adding this rule, and instead still require the create method to take a `ReadOnlySpan<E>`.  However, that would require the BCL to then add such a method to `ImmutableDictionary` as the existing `CreateRange` methods take an `IEnumerable`.

### Open question 2

It is common for dictionaries to take in a `comparer` value, to determine how keys are compared when adding, removing, and looking them up.  Use in the ecosystem is prevalent, and much discussion and feedback from the community is that being able to supply a comparer is important to them.  How could we accomplish this with the new dictionary-expression?  Options include:

1. Provide no support (or no support in C# 13), leaving such dictionaries as ones you could not use *dictionary expressions* for.

1. Provide a special syntactic form *solely* for the purpose of supplying that value.  For example: `[comparer: myComp, "mads": 21, .. ldmMembers]`.  Here `comparer` would be a contextual keyword.  Users wanting to use that as an actual key would say `@comparer: 1`

1. Provide a special syntactic form for the purpose of supplying data to constructors and create methods.  For example: `[new: (comparer: myComp, capacity: 50), "mads": 21, .. ldmMembers]`

## Construction

> The elements of a collection expression are evaluated in order, left to right. Each element is evaluated exactly once, and any further references to the elements refer to the results of this initial evaluation.
> 
> ```diff
> + A dictionary_element evaluates its interior expressions in order, left to right.  In other words, the key is evaluated before the value. 
> ```
>
> For each element in order:
>
> - If the element is an expression element, the applicable Add instance or extension method is invoked with the element expression as the argument. (Unlike classic collection initializer behavior, element evaluation and Add calls are not necessarily interleaved.).
> 
> - ```diff
>   + If the target is a dictionary type, then the element must be a `KeyValuePair<,>`.  The applicable indexer is invoked with the `.Key` and `.Value` members of that pair.
>   ```
>
> - If the element is a spread element then one of the following is used:
>   - An applicable GetEnumerator instance or extension method is invoked on the spread element expression and for each item from the enumerator the applicable Add instance or extension method is invoked on the collection instance with the item as the argument. If the enumerator implements IDisposable, then Dispose will be called after enumeration, regardless of exceptions.
> 
>    - ```diff
>      + If the target is a dictionary-type, the enumerator's element type must be some `KeyValuePair<,>`, and for each of those elements the applicable indexer is invoked on the collection instance with the `.Key` and `.Value` members of that pair.
>      ```

## Type inference

```c#
var a = AsDictionary(["mads": 21, "dustin": 22]); // AsDictionary<string, int>(Dictionary<string, int> arg)

static Dictionary<TKey, TValue> AsDictionary<TKey, TValue>(Dictionary<TKey, TValue> arg) => arg;
```

Rules TBD.  Intuition though is to be inferring both a `TKey` and `TValue` type. `k: v` elements contribute input and output inferences respectively to those types.  Normal expression elements and spread elements must have associated `KeyValuePair<K_n, V_n>` types, where the `K_n` and `V_n` then contribute as well.

For example:

```c#
KeyValuePair<object, long> kvp = ...;
var a = AsDictionary(["mads": 21, "dustin": 22, kvp]); // AsDictionary<object, long>(Dictionary<object, long> arg)

static Dictionary<TKey, TValue> AsDictionary<TKey, TValue>(Dictionary<TKey, TValue> arg) => arg;
```

## Extension methods

No changes here.  Like with collection expressions, dictionary expressions do not have a natural type, so the existing conversions from type are not applicable. As a result, a dictionary expression cannot be used directly as the first parameter for an extension method invocation. 

## Overload resolution

No changes currently.  But open question if any `better conversion from expression` rules are needed.

Tentatively we think the answer is no.  The types that would appear in signatures would likely be:

```c#
void X(IDictionary<A, B> dict);
void X(Dictionary<A, B> dict);
```

In this case, standard betterness would pick the latter method.

Similarly for:

```c#
void X(IEnumerable<KeyValuePair<A, B>> dict);
void X(Dictionary<A, B> dict);
```

Similar to *collection expressions*, there is no betterness between disparate concrete dictionary types.  For example:

```c#
void X(Dictionary<A, B> dict);
void X(ImmutableDictionary<A, B> dict);

X([a, b]); // ambiguous
```

## Interface translation

Given a target type `IReadOnlyDictionary<TKey, TValue>` or `IDictionary<TKey, TValue>` a compliant implementation is only required to produce a value that implements that interface. A compliant implementation is free to:

Use an existing type that implements that interface.
Synthesize a type that implements the interface.
In either case, the type used is allowed to implement a larger set of interfaces than those strictly required.

Synthesized types are free to employ any strategy they want to implement the required interfaces properly. 

### Non-mutable interface translation

Given a target type of `IReadOnlyDictionary<TKey, TValue>`, the value generated is allowed to implement more interfaces than required. For example, implementing the mutable interfaces as well (specifically, implementing IDictionary<TKey, TValue>`). However, in that case:

1. The value must return true when queried for `ICollection<KeyValuePair<TKey, TValue>>.IsReadOnly`. This ensures consumers can appropriately tell that the collection is non-mutable, despite implementing the mutable views.
1. The value must throw on any call to a mutation method. This ensures safety, preventing a non-mutable collection from being accidentally mutated.

### Mutable interface translation

Given a target type or `IDictionary<TKey, TValue>`:

1. The value must return false when queried for `ICollection<KeyValuePair<TKey, TValue>>.IsReadOnly`.
The value generated is allowed to implement more interfaces than required.  For example, implementing the non-generic `IDictionary` as well.

1. The value must support all mutation methods (like IDictionary.Add).

### Open question 1

There is a subtle concern around the following interface destinations:

```c#
void Xxx(IEnumerable<KeyValuePair<string, int>> pairs) ...
void Yyy(IDictionary<string, int> pairs) ...

Xxx(["mads": 21, .. ldm]);
Yyy(["mads": 21, .. ldm]);
```

When the destination is an IEnumerable, we tend to think we're producing a sequence (so "mads" could show up twice).  However, the use of the `k:v` syntax more strongly indicates production of a dictionary-value.

What should we do here when targeting `IEnumerable<...>` *and* using `k:v` elements? Produce an ordered sequence, with possibly duplicated values?  Or produce an unordered dictionary, with unique keys?