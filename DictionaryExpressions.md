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

