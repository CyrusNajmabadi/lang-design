# KeyValuePair correspondence

In .Net dictionary-types and the System.Collection.KeyValuePair<TKey, TValue> (aka KVP or KVP<>) types are intertwined.  A dictionary is commonly defined as a collection of elements of that KVP type, not just a mapping from some TKey to some TValue.  Indeed, this duality allows one to treat the two spaces mostly uniformly.  For example:

```c#
var dictionary = new Dictionary<string, int>();
var collection = (ICollection<KeyValuePair<string, int>>)dictionary;
collection.Add(new KeyValuePair<string, int>("mads", 21));
```

The only special thing here over standard element-based collection expressions is that the the dictionary-types have a general view that any particular key will only be contained once, and can be used to then efficiently map to its associated value. 

Because of this correspondance we believe that dictionary expressions should not be considered very special.  Rather, what the dictionary-expression feature is is a way to simply allow KeyValuePairs to be expressed within collection expressions, along with a sensible and uniform set of rules to allow KeyValuePairs to naturally initialize collection types.

For example:

1. There is a new special syntax for declaring a KeyValuePair within a collection expression:

    ```c#
    X x = [k: v];
    ```

1. It can be used with dictionary-types like so:

    ```c#
    Dictionary<string, int> nameToAge = ["mads": 21];
    ```

1. This syntax is not unique to 'dictionary types'.  It can be used for standard existing collection types like so:

    ```c#
    List<KeyValuePair<string, int>> pairs = ["mads": 21];
    ```

1. While the syntax allows for easy specification of the particular key and value, it is optional and the feature works with normal KeyValuePair instances:

    ```c#
    KeyValuePair<string, int> kvp = new("mads", 21);
    Dictionary<string, int> nameToAge = [kvp];
    ```

1. The above allows for *uniformity* of processing KeyValuePair values, which we consider desirable so that users can expect them to work for all collection expressions elements like so:

    ```c#
    Dictionary<string, int> nameToAge = [.. defaultValues, otherMap.Single(predicate)];
    ```

    Here, being able to 'spread' in another collection (which would normally be some `IEnumerable<KeyValuePair<>>`) is desirable.  Similarly, being able to add invidual pairs found through some means, without having to decompose into `k: v` syntax is equally preferable.

# KeyValuePair transparency

We expect the exact type of the KeyValuePair<> values to be generally transparent.  Rather than being strictly that type, the language will generally see *through* it to be a pair of some TKey and TValue types.  This transparency is in line with how tuples behave and serves as a strong intuition for how we want users to intuit KeyValuePairs in the context of collection expressions.

How does this transparency manifest?  Consider the following scenario:

```c#
Dictionary<object, int?> map = ["mads": 21];
```

The above expression would certainly be expected to work.  While `"mads"` is a string, and `21` an `int`, the target-typed nature of collection expressions would push the `object` and `int?` types through the constituent key and value expressions to type them properly.  This would also be expected to work in the following:

```c#
Dictionary<object?, int?> map = [null: null];
```

