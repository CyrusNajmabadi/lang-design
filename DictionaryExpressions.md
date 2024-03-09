# Dictionary Expressions

## Summary

Dictionary expressions are a continuation of the C# 12 collection expressions feature.  They extend that system with a new terse syntax, `["mads": 21, "dustin": 22]`, for creating common dictionary values.  Like with collection expressions, merging other dictionaries into these values is possible using the existing spread operator `..` like so: `[.. currentStudents, "mads": 21, "dustin": 22]`

Several dictionary-like types can created without external BCL support.  These types are:

1. Dictionary-like types, containing an indexer `TValue this[TKey] { get; set; }`, like `Dictionary<TKey, TValue>` and `ConcurrentDictionary<TKey, TValue>`.
1. The well-known BCL dictionary interface types: `IDictionary<TKey, TValue>` and `IReadOnlyDictionary<TKey, TValue>`.

Further support is present for dictionary-like types not covered above through the `CollectionBuilderAttribute` and a similar API pattern to the corresponding `Create` pattern introduced for collection-expressions.

## Motivation

While dictionary-values are similar to standard sequential collection-values in that they can be interpreted as a sequence of key/value pairs, they differ in that they are often used for their more fundamental capability of efficient looking up of values based on a provided key. 

In an analysis of the BCL and the nuget package ecosystem, sequential-collection types and values make up the lion's share of collections used.  However, dictionary types were still used a significant amount, with appearances in APIs occurring at between 5 and 10% the amount of sequential collections, and with values appearing universally.  Currently, all C# programs must use many different and unfortunately verbose approaches to create instances of such values. Some approaches also have performance drawbacks. Here are some common examples:

1. Collection-initializer, which require syntax like `new Dictionary<X, Y>` (lacking inference of possibly verbose TKey and TValue) prior to their values, and which can cause multiple reallocations of memory because they use `N` `.Add` invocations without supplying an initial capacity.
1. Immutable collections, which require syntax like `ImmutableDictionary.CreateRange(...)`, but which is also unpleasant due to the need to provide values as an `IEnumerable<KeyValuePair>`s.  Builders are even more unwieldy.
1. Read-only dictionaries, which require both making a normal dictionary first, then wrapping them.
1. Concurrent-dictionaries, which lack an `.Add` method, and thus cannot be used with normal collection initializers.

Looking at the surrounding ecosystem, we also find examples everywhere of dictionary creation being more convenient and pleasant to use. TypeScript, Dart, Ruby, Python, and more opt for a succinct syntax for this purpose, with widespread usage, and to great effect. Cursory investigations have revealed no substantive problems arising in those ecosystems with having these literals built in.

Unlike collection-expressions, C# does not have an existing dictionary-pattern the corresponding deconstruction form.  Designs here should be made with a consideration for being complimentary with deconstruction work. 

An inclusive solution is needed for C#. It should meet the vast majority of case for customers in terms of the dictionary-like types and values they already have. It should also feel natural in the language,  complement the work done with collection expressions, and naturally extend to pattern matching in the future.

## Detailed Design

The following grammar productions are added:

```diff
collection_element
  : expression_element
  | spread_element
+ | dictionary_element  
  ;

+dictionary_element
  : expression ':' expression
```

Alternative syntaxes are available for consideration, but should be considered later due to the bike-shedding cost involved.  Picking the above syntax allows for the compiler team to move quickly at implementing the semantic side of the feature, allowing earlier previews to be made available.  These syntaxes include, but are not limited to:

1. Using braces instead of brackets.  `{ k1: v1, k2: v2 }`.
2. Using brackets for keys: `[k1] = v1, [k2] = v2`
3. Using arrows for elements: `k1 => v1, k2 => v2`.

Choices here would have implications regarding potential syntactic ambiguities, collisions with potential future language features, and concerns around corresponding pattern forms.  However, all of those should not generally affect the semantics of the feature and can be considered at a later point dedicated to determining the most desirable syntax.

Intuitively, dictionary-expressions work similarly to collection-expressions, except treating `k:v` as a shorthand for creating a `System.Collections.Generic.KeyValuePair<TKey, TValue>`.  As such, the following would be legal:

```c#
Dictionary<string, int> nameToAge = ["mads": 21, existingKvp]; // as would
Dictionary<string, int> nameToAge = ["mads": 21, .. existingDict]; // as would
Dictionary<string, int> nameToAge = ["mads": 21, .. existingListOfKVPS];
```

The support for spreads only being concerned with the element types matches the equivalent cases in the collection-expr case, such as:

```c#
List<KeyValuePair<string, int>> nameToAge = [.. someDict]; // supported today in C# 12
```

Open question 1: How far do we want to accept this KeyValuePair representation of things.  For example, should the following be allowed:

```c#
List<KeyValuePair<string, int>> = ["mads": 21];
```

Or should the presence of `k:v` element mandate some dictionary type as the receiver.  Note: the above sort of API is not uncommon.  Roslyn itself contains many apis that permissively *receive* an `IEnumerable<KeyValuePair<string, int>>`.

Importantly, we do not believe it wise to *require* the presence of a `k:v` element to produce a dictionary instance.  For example, we believe it very reasonable and desirable that someone be able to write:

```c#
Dictionary<string, int> everyone = [.. students, .. teachers];
```



### Conversions

A *collection expression conversion* allows a collection expression to be converted to a type.

An *implicit collection expression conversion* exists from a collection expression to the following types:

```diff
// Existing rules ...

+ A type with a `CreateRange` with an iteration type of some `KeyValuePair<TKey, TValue>` and an argument type of some `CollectionType<KeyValuePai<TKey, TValue>`.  For example `public static ImmutableDictionary CreateRange<TKey, TValue>(IEnumerable<KeyValuePair<TKey, TValue>>)`. Note: it is an open question what collection types are supported for the argument type.

A struct or class type that implements System.Collections.IEnumerable where:
The type has an applicable constructor that can be invoked with no arguments, and the constructor is accessible at the location of the collection expression.
+ The type has an applicable indexer that can be invoked  
If the collection expression has any elements, the type has an applicable instance or extension method Add that can be invoked with a single argument of the iteration type, and the method is accessible at the location of the collection expression.