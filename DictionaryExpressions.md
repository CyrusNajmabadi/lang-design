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

