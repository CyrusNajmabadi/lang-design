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
Dictionary<object, int?> map1 = ["mads": 21];
```

The above expression would certainly be expected to work.  While `"mads"` is a string, and `21` an `int`, the target-typed nature of collection expressions would push the `object` and `int?` types through the constituent key and value expressions to type them properly.  This would also be expected to work in the following:

```c#
Dictionary<object?, int?> map2 = [null: null];
```

KeyValuePair transparency means though that just as we expect the code for map1 to be legal, we should consider the following legal as well:

```c#
KeyValuePair<string, int> kvp = new("mads", 21);
Dictionary<object, int?> map1 = [kvp];
```

After all, why would that be illegal, while the following then became legal:

```c#
KeyValuePair<string, int> kvp = new("mads", 21);
Dictionary<object, int?> map1 = [kvp.Key: kvp.Value];
```

Requiring explicit deconstruction of the constituent key and value portions of a KVP just to satisfy the compiler so it could then target type them just adds extra, painful steps.  It would become doubly worse once all collection element expressions are considered, like so:

```c#
Dictionary<object, int?> map = [.. nameToAge.Select(kvp => new KeyValuePair<object, int?>(kvp.Key, kvp.Value)];
```

# Tuple analogy 

While this transparency seems like it might be a large stretch beyond how C# generally works.  It turns out that this sort of behavior is *exactly* what already exists in the language today for tuples.  Consider the following:

```c#
List<(object? key, int? value)> map = [("mads", 21)];
```

This already works today.  The language transparently sees through into the tuple expression to ensure that the above it legal.  This is also not a conversion applied to some `(string, int)` tuple type.  That can be seen here which is also legal:

```c#
List<(object? key, int? value)> map = [(null, null)];
```

Here, the types of the destination flow all the way through (including recursively through nested tuple types) into the tuple expression in the initializer.  This transparency is not limited to *tuple expressions* either.  All of hte following are legal as well, despite non-matching ValueTuple<> types:

```c#
(string x, int y) kvp = ("mads", 21);
List<(object? key, int? value)> map = [kvp];
```

And

```c#
(string? x, int? y) kvp = (null, null);
List<(object? key, int? value)> map = [kvp];
```

The language permissively always views tuples as a loose aggregation of constituent elements, each with their own type.  Conversions and compatibility are all performed on those constituent element types, not on the top level `ValueTuple<>` type (which would normally not be compatible based on .Net type system rules).

# KeyValuePair Inference

The above tuple-analogy then serves as an analogous sytem we can look to to see how we would like KeyValuePair<> to behave in collection expressions.  For example:

```c#
void M<TKey, TValue>(List<(TKey key, TValue value)> list1, List<(TKey key, TValue value)> list);

// Note: neither kvp1 nor kvp2 are assignable/implcitly-convertible to each other.
(string x, int? y) kvp1 = ("mads", 21);
(object x, int y) kvp2 = ("cyrus", 22);

M([kvp1], [kvp2]);
```

This works today and correctly infers `M<object, int?>`.  Given the above, we would then expect the following to work:


```c#
void M<TKey, TValue>(Dictionary<TKey, TValue> d1, Dictionary<TKey, TValue> s2);

// Note: neither kvp1 nor kvp2 are assignable/implcitly-convertible to each other.
(string x, int? y) kvp1 = ("mads", 21);
(object x, int y) kvp2 = ("cyrus", 22);

M([kvp1], [kvp2]); // Should infer `M<object, int?>` as well.
```

# Tuple analogy (cont.)

The analogous tuple behavior serves as a good *bedrock* for our intuitions on how we want KeyValuePairs.  However, how far we want to take this analogy is up to us, and we can consider several levels of increasing transparency support.  Those levels are:

1. No transparency support.  Do not treat KVPs like tuples.  Force users to explicitly convert between KVP types to satisfy type safety at the KVP level itself.  For example:

    ```c#
    KeyValuePair<string, int> kvp = new("mads", 21);
    Dictionary<object, int?> map1 = [kvp]; // illegal.  user must write:

    Dictionary<object, int?> map1 = [kvp.Key: kvp.Value];
    ```

1. Transparent only when targeting some 'dictionary type', but not non-dictionary types:

    ```c#
    KeyValuePair<string, int> kvp = new("mads", 21);
    Dictionary<object, int?> map1 = [kvp]; // legal.

    List<(object, int?)> map1 = [kvp]; // not legal.  User must write:
    List<(object, int?)> map1 = [kvp.Key: kvp.Value]; // or
    List<(object, int?)> map1 = [new KeyValuePair<object, int?>(kvp.Key, kvp.Value)];
    ```

1. Transparent in any collection expression, but no further:

    ```c#
    KeyValuePair<string, int> kvp = new("mads", 21);
    Dictionary<object, int?> map1 = [kvp]; // legal.
    List<(object, int?)> map1 = [kvp]; // legal.
    
    KeyValuePair<object, int?> kvp2 = kvp1; // not legal.  User must write:
    KeyValuePair<object, int?> kvp2 = new KeyValuePair<object, int?>(kvp1.Key, kvp.Value);
    ```

1. Transparent everywhere:


    ```c#
    KeyValuePair<string, int> kvp = new("mads", 21);
    Dictionary<object, int?> map1 = [kvp]; // legal.
    List<(object, int?)> map1 = [kvp]; // legal.
    KeyValuePair<object, int?> kvp2 = kvp1; // legal.
    ```

These four options form a spectrum.  Where the starting point is doing nothing special, then only handling dictionaries, then handling any collection, all the way to the final support which effectively puts KeyValuePair handling at the same level as tuples for the language.

Open question 1: How far would we like to take this transparency?  All the way to full analogy with tuples?  No transparency at all?  Somewhere in the middle?

# Deconstruction

All of the above so far has been about KeyValuePair, how the language would enable working more conveniently with it.  And, there are good arguments to be made that KeyValuePair needs both to allow these important scenarios to light up, and due to how integral it is to the dictionary-type space to begin with.  However, fundemantally, all of the above could be reformulated, enabling the same scenarios without specializing KeyValuePair at all.  Specifically, all of the above works by stating that KeyValuePair can be seen transparently as a pair of two typed values (the `TKey Key` and the `TValue Value`).  Fundamentally, as that's all that is truly required, a relaxation could be performed that restates all of the above as:

    Any type that is *constructible* and *deconstructible* into two elements would be transparently supported in the context of collection expressions and the `k: v` element.

That relaxation would consume all the KeyValuePair support.  But would also then enable tuples to be used in all those cases *as well as* any appropriate type supporting two-element construction/deconstruction.  As such, all of the below would be legal:

```c#
Dictionary<string, int> nameToAge1 = [("mads", 21)];

List<(string, int)> pairs = ...;
Dictionary<string, int> nameToAge1 = [.. pairs];

record struct Point(double X, double Y);
Dictionary<int, int> function = [point1, point2];

List<Point> points = [1.0: 1.0, 2.0: 4.0, 3.0: 8.0]
// etc.
```

Open question 2: How far would we like to take this?  Only support KeyValuePair?  Support KeyValuePair and 2-element tuples?  Support any 2-element deconstructible/constructible types?