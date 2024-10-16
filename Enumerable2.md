```c#
// Fully binary compat with existing enumerable type
extension Enumerable
{
    public TSource (IEnumerable<TSource> source).Aggregate<TSource>(Func<TSource, TSource, TSource> func);
    public TAccumulate (IEnumerable<TSource> source).Aggregate<TSource, TAccumulate>(TAccumulate seed, Func<TAccumulate, TSource, TAccumulate> func);
    public TResult (IEnumerable<TSource> source).Aggregate<TSource, TAccumulate, TResult>(TAccumulate seed, Func<TAccumulate, TSource, TAccumulate> func, Func<TAccumulate, TResult> resultSelector);

    public IEnumerable<KeyValuePair<TKey, TAccumulate>> (IEnumerable<TSource> source).AggregateBy<TSource, TKey, TAccumulate>(Func<TSource, TKey> keySelector, TAccumulate seed, Func<TAccumulate, TSource, TAccumulate> func, IEqualityComparer<TKey>? keyComparer = null) where TKey : notnull;
    public IEnumerable<KeyValuePair<TKey, TAccumulate>> (IEnumerable<TSource> source).AggregateBy<TSource, TKey, TAccumulate>(Func<TSource, TKey> keySelector, Func<TKey, TAccumulate> seedSelector, Func<TAccumulate, TSource, TAccumulate> func, IEqualityComparer<TKey>? keyComparer = null) where TKey : notnull;

    public bool (IEnumerable<TSource> source).All<TSource>(Func<TSource, bool> predicate);

    public bool (IEnumerable<TSource> source).Any<TSource>();
    public bool (IEnumerable<TSource> source).Any<TSource>(Func<TSource, bool> predicate);

    public IEnumerable<TSource> (IEnumerable<TSource> source).Append<TSource>(TSource element);

    public IEnumerable<TSource> (IEnumerable<TSource> source).AsEnumerable<TSource>();

    public float? (IEnumerable<TSource> source).Average<TSource>(Func<TSource, float?> selector);
    public double? (IEnumerable<TSource> source).Average<TSource>(Func<TSource, long?> selector);
    public double? (IEnumerable<TSource> source).Average<TSource>(Func<TSource, int?> selector);
    public double? (IEnumerable<TSource> source).Average<TSource>(Func<TSource, double?> selector);
    public decimal? (IEnumerable<TSource> source).Average<TSource>(Func<TSource, decimal?> selector);
    public double (IEnumerable<TSource> source).Average<TSource>(Func<TSource, long> selector);
    public double (IEnumerable<TSource> source).Average<TSource>(Func<TSource, int> selector);
    public double (IEnumerable<TSource> source).Average<TSource>(Func<TSource, double> selector);
    public decimal (IEnumerable<TSource> source).Average<TSource>(Func<TSource, decimal> selector);
    public double? (IEnumerable<double?> source).Average();
    public float? (IEnumerable<float?> source).Average();
    public double? (IEnumerable<long?> source).Average();
    public double? (IEnumerable<int?> source).Average();
    public float (IEnumerable<TSource> source).Average<TSource>(Func<TSource, float> selector);
    public decimal? (IEnumerable<decimal?> source).Average();
    public double (IEnumerable<long> source).Average();
    public double (IEnumerable<int> source).Average();
    public double (IEnumerable<double> source).Average();
    public decimal (IEnumerable<decimal> source).Average();
    public float (IEnumerable<float> source).Average();

    public IEnumerable<TResult> (IEnumerable source).Cast<TResult>();

    public IEnumerable<TSource[]> (IEnumerable<TSource> source).Chunk<TSource>(int size);

    public IEnumerable<TSource> (IEnumerable<TSource> first).Concat<TSource>(IEnumerable<TSource> second);

    public bool (IEnumerable<TSource> source).Contains<TSource>(TSource value);
    public bool (IEnumerable<TSource> source).Contains<TSource>(TSource value, IEqualityComparer<TSource>? comparer);

    public int (IEnumerable<TSource> source).Count<TSource>();
    public int (IEnumerable<TSource> source).Count<TSource>(Func<TSource, bool> predicate);

    public IEnumerable<KeyValuePair<TKey, int>> (IEnumerable<TSource> source).CountBy<TSource, TKey>(Func<TSource, TKey> keySelector, IEqualityComparer<TKey>? keyComparer = null) where TKey : notnull;

    public IEnumerable<TSource?> (IEnumerable<TSource> source).DefaultIfEmpty<TSource>();
    public IEnumerable<TSource> (IEnumerable<TSource> source).DefaultIfEmpty<TSource>(TSource defaultValue);

    public IEnumerable<TSource> (IEnumerable<TSource> source).Distinct<TSource>();
    public IEnumerable<TSource> (IEnumerable<TSource> source).Distinct<TSource>(IEqualityComparer<TSource>? comparer);

    public IEnumerable<TSource> (IEnumerable<TSource> source).DistinctBy<TSource, TKey>(Func<TSource, TKey> keySelector);
    public IEnumerable<TSource> (IEnumerable<TSource> source).DistinctBy<TSource, TKey>(Func<TSource, TKey> keySelector, IEqualityComparer<TKey>? comparer);

    public TSource (IEnumerable<TSource> source).ElementAt<TSource>(Index index);
    public TSource (IEnumerable<TSource> source).ElementAt<TSource>(int index);

    public TSource? (IEnumerable<TSource> source).ElementAtOrDefault<TSource>(Index index);
    public TSource? (IEnumerable<TSource> source).ElementAtOrDefault<TSource>(int index);

    public IEnumerable<TResult> Empty<TResult>();

    public IEnumerable<TSource> (IEnumerable<TSource> first).Except<TSource>(IEnumerable<TSource> second);
    public IEnumerable<TSource> (IEnumerable<TSource> first).Except<TSource>(IEnumerable<TSource> second, IEqualityComparer<TSource>? comparer);

    public IEnumerable<TSource> (IEnumerable<TSource> first).ExceptBy<TSource, TKey>(IEnumerable<TKey> second, Func<TSource, TKey> keySelector);
    public IEnumerable<TSource> (IEnumerable<TSource> first).ExceptBy<TSource, TKey>(IEnumerable<TKey> second, Func<TSource, TKey> keySelector, IEqualityComparer<TKey>? comparer);

    public TSource (IEnumerable<TSource> source).First<TSource>(Func<TSource, bool> predicate);
    public TSource (IEnumerable<TSource> source).First<TSource>();

    public TSource (IEnumerable<TSource> source).FirstOrDefault<TSource>(Func<TSource, bool> predicate, TSource defaultValue);
    public TSource (IEnumerable<TSource> source).FirstOrDefault<TSource>(TSource defaultValue);

    public TSource? (IEnumerable<TSource> source).FirstOrDefault<TSource>();
    public TSource? (IEnumerable<TSource> source).FirstOrDefault<TSource>(Func<TSource, bool> predicate);

    public IEnumerable<TResult> (IEnumerable<TSource> source).GroupBy<TSource, TKey, TResult>(Func<TSource, TKey> keySelector, Func<TKey, IEnumerable<TSource>, TResult> resultSelector, IEqualityComparer<TKey>? comparer);
    public IEnumerable<TResult> (IEnumerable<TSource> source).GroupBy<TSource, TKey, TElement, TResult>(Func<TSource, TKey> keySelector, Func<TSource, TElement> elementSelector, Func<TKey, IEnumerable<TElement>, TResult> resultSelector, IEqualityComparer<TKey>? comparer);
    public IEnumerable<TResult> (IEnumerable<TSource> source).GroupBy<TSource, TKey, TElement, TResult>(Func<TSource, TKey> keySelector, Func<TSource, TElement> elementSelector, Func<TKey, IEnumerable<TElement>, TResult> resultSelector);
    public IEnumerable<TResult> (IEnumerable<TSource> source).GroupBy<TSource, TKey, TResult>(Func<TSource, TKey> keySelector, Func<TKey, IEnumerable<TSource>, TResult> resultSelector);

    public IEnumerable<IGrouping<TKey, TSource>> (IEnumerable<TSource> source).GroupBy<TSource, TKey>(Func<TSource, TKey> keySelector, IEqualityComparer<TKey>? comparer);
    public IEnumerable<IGrouping<TKey, TElement>> (IEnumerable<TSource> source).GroupBy<TSource, TKey, TElement>(Func<TSource, TKey> keySelector, Func<TSource, TElement> elementSelector);
    public IEnumerable<IGrouping<TKey, TElement>> (IEnumerable<TSource> source).GroupBy<TSource, TKey, TElement>(Func<TSource, TKey> keySelector, Func<TSource, TElement> elementSelector, IEqualityComparer<TKey>? comparer);
    public IEnumerable<IGrouping<TKey, TSource>> (IEnumerable<TSource> source).GroupBy<TSource, TKey>(Func<TSource, TKey> keySelector);

    public IEnumerable<TResult> (IEnumerable<TOuter> outer).GroupJoin<TOuter, TInner, TKey, TResult>(IEnumerable<TInner> inner, Func<TOuter, TKey> outerKeySelector, Func<TInner, TKey> innerKeySelector, Func<TOuter, IEnumerable<TInner>, TResult> resultSelector);
    public IEnumerable<TResult> (IEnumerable<TOuter> outer).GroupJoin<TOuter, TInner, TKey, TResult>(IEnumerable<TInner> inner, Func<TOuter, TKey> outerKeySelector, Func<TInner, TKey> innerKeySelector, Func<TOuter, IEnumerable<TInner>, TResult> resultSelector, IEqualityComparer<TKey>? comparer);

    public IEnumerable<(int Index, TSource Item)> (IEnumerable<TSource> source).Index<TSource>();

    public IEnumerable<TSource> (IEnumerable<TSource> first).Intersect<TSource>(IEnumerable<TSource> second);
    public IEnumerable<TSource> (IEnumerable<TSource> first).Intersect<TSource>(IEnumerable<TSource> second, IEqualityComparer<TSource>? comparer);

    public IEnumerable<TSource> (IEnumerable<TSource> first).IntersectBy<TSource, TKey>(IEnumerable<TKey> second, Func<TSource, TKey> keySelector);
    public IEnumerable<TSource> (IEnumerable<TSource> first).IntersectBy<TSource, TKey>(IEnumerable<TKey> second, Func<TSource, TKey> keySelector, IEqualityComparer<TKey>? comparer);

    public IEnumerable<TResult> (IEnumerable<TOuter> outer).Join<TOuter, TInner, TKey, TResult>(IEnumerable<TInner> inner, Func<TOuter, TKey> outerKeySelector, Func<TInner, TKey> innerKeySelector, Func<TOuter, TInner, TResult> resultSelector);
    public IEnumerable<TResult> (IEnumerable<TOuter> outer).Join<TOuter, TInner, TKey, TResult>(IEnumerable<TInner> inner, Func<TOuter, TKey> outerKeySelector, Func<TInner, TKey> innerKeySelector, Func<TOuter, TInner, TResult> resultSelector, IEqualityComparer<TKey>? comparer);

    public TSource (IEnumerable<TSource> source).Last<TSource>(Func<TSource, bool> predicate);
    public TSource (IEnumerable<TSource> source).Last<TSource>();

    public TSource (IEnumerable<TSource> source).LastOrDefault<TSource>(Func<TSource, bool> predicate, TSource defaultValue);
    public TSource (IEnumerable<TSource> source).LastOrDefault<TSource>(TSource defaultValue);

    public TSource? (IEnumerable<TSource> source).LastOrDefault<TSource>();
    public TSource? (IEnumerable<TSource> source).LastOrDefault<TSource>(Func<TSource, bool> predicate);

    public long (IEnumerable<TSource> source).LongCount<TSource>();
    public long (IEnumerable<TSource> source).LongCount<TSource>(Func<TSource, bool> predicate);

    public decimal (IEnumerable<TSource> source).Max<TSource>(Func<TSource, decimal> selector);
    public double (IEnumerable<TSource> source).Max<TSource>(Func<TSource, double> selector);
    public TResult? (IEnumerable<TSource> source).Max<TSource, TResult>(Func<TSource, TResult> selector);

    public int (IEnumerable<TSource> source).Max<TSource>(Func<TSource, int> selector);
    public long (IEnumerable<TSource> source).Max<TSource>(Func<TSource, long> selector);
    public int? (IEnumerable<TSource> source).Max<TSource>(Func<TSource, int?> selector);
    public double? (IEnumerable<TSource> source).Max<TSource>(Func<TSource, double?> selector);
    public long? (IEnumerable<TSource> source).Max<TSource>(Func<TSource, long?> selector);
    public float? (IEnumerable<TSource> source).Max<TSource>(Func<TSource, float?> selector);
    public float (IEnumerable<TSource> source).Max<TSource>(Func<TSource, float> selector);
    public decimal? (IEnumerable<TSource> source).Max<TSource>(Func<TSource, decimal?> selector);
    public TSource? (IEnumerable<TSource> source).Max<TSource>(IComparer<TSource>? comparer);
    public float? (IEnumerable<float?> source).Max();
    public int (IEnumerable<int> source).Max();
    public TSource? (IEnumerable<TSource> source).Max<TSource>();
    public float (IEnumerable<float> source).Max();
    public long? (IEnumerable<long?> source).Max();
    public int? (IEnumerable<int?> source).Max();
    public double? (IEnumerable<double?> source).Max();
    public decimal? (IEnumerable<decimal?> source).Max();
    public long (IEnumerable<long> source).Max();
    public decimal (IEnumerable<decimal> source).Max();
    public double (IEnumerable<double> source).Max();

    public TSource? (IEnumerable<TSource> source).MaxBy<TSource, TKey>(Func<TSource, TKey> keySelector, IComparer<TKey>? comparer);
    public TSource? (IEnumerable<TSource> source).MaxBy<TSource, TKey>(Func<TSource, TKey> keySelector);

    public double (IEnumerable<TSource> source).Min<TSource>(Func<TSource, double> selector);
    public int (IEnumerable<TSource> source).Min<TSource>(Func<TSource, int> selector);
    public long (IEnumerable<TSource> source).Min<TSource>(Func<TSource, long> selector);
    public decimal? (IEnumerable<TSource> source).Min<TSource>(Func<TSource, decimal?> selector);
    public double? (IEnumerable<TSource> source).Min<TSource>(Func<TSource, double?> selector);
    public decimal (IEnumerable<TSource> source).Min<TSource>(Func<TSource, decimal> selector);
    public long? (IEnumerable<TSource> source).Min<TSource>(Func<TSource, long?> selector);
    public float? (IEnumerable<TSource> source).Min<TSource>(Func<TSource, float?> selector);
    public float (IEnumerable<TSource> source).Min<TSource>(Func<TSource, float> selector);
    public TResult? (IEnumerable<TSource> source).Min<TSource, TResult>(Func<TSource, TResult> selector);
    public int? (IEnumerable<TSource> source).Min<TSource>(Func<TSource, int?> selector);
    public TSource? (IEnumerable<TSource> source).Min<TSource>(IComparer<TSource>? comparer);
    public decimal (IEnumerable<decimal> source).Min();
    public long (IEnumerable<long> source).Min();
    public TSource? (IEnumerable<TSource> source).Min<TSource>();
    public float (IEnumerable<float> source).Min();
    public float? (IEnumerable<float?> source).Min();
    public long? (IEnumerable<long?> source).Min();
    public int? (IEnumerable<int?> source).Min();
    public double? (IEnumerable<double?> source).Min();
    public decimal? (IEnumerable<decimal?> source).Min();
    public double (IEnumerable<double> source).Min();
    public int (IEnumerable<int> source).Min();

    public TSource? (IEnumerable<TSource> source).MinBy<TSource, TKey>(Func<TSource, TKey> keySelector, IComparer<TKey>? comparer);
    public TSource? (IEnumerable<TSource> source).MinBy<TSource, TKey>(Func<TSource, TKey> keySelector);

    public IEnumerable<TResult> (IEnumerable source).OfType<TResult>();

    public IOrderedEnumerable<T> (IEnumerable<T> source).Order<T>(IComparer<T>? comparer);
    public IOrderedEnumerable<T> (IEnumerable<T> source).Order<T>();

    public IOrderedEnumerable<TSource> (IEnumerable<TSource> source).OrderBy<TSource, TKey>(Func<TSource, TKey> keySelector);
    public IOrderedEnumerable<TSource> (IEnumerable<TSource> source).OrderBy<TSource, TKey>(Func<TSource, TKey> keySelector, IComparer<TKey>? comparer);

    public IOrderedEnumerable<TSource> (IEnumerable<TSource> source).OrderByDescending<TSource, TKey>(Func<TSource, TKey> keySelector);
    public IOrderedEnumerable<TSource> (IEnumerable<TSource> source).OrderByDescending<TSource, TKey>(Func<TSource, TKey> keySelector, IComparer<TKey>? comparer);

    public IOrderedEnumerable<T> (IEnumerable<T> source).OrderDescending<T>(IComparer<T>? comparer);
    public IOrderedEnumerable<T> (IEnumerable<T> source).OrderDescending<T>();

    public IEnumerable<TSource> (IEnumerable<TSource> source).Prepend<TSource>(TSource element);

    public static IEnumerable<int> Range(int start, int count);

    public IEnumerable<TResult> Repeat<TResult>(TResult element, int count);

    public IEnumerable<TSource> (IEnumerable<TSource> source).Reverse<TSource>();

    public IEnumerable<TResult> (IEnumerable<TSource> source).Select<TSource, TResult>(Func<TSource, int, TResult> selector);
    public IEnumerable<TResult> (IEnumerable<TSource> source).Select<TSource, TResult>(Func<TSource, TResult> selector);

    public IEnumerable<TResult> (IEnumerable<TSource> source).SelectMany<TSource, TCollection, TResult>(Func<TSource, int, IEnumerable<TCollection>> collectionSelector, Func<TSource, TCollection, TResult> resultSelector);
    public IEnumerable<TResult> (IEnumerable<TSource> source).SelectMany<TSource, TResult>(Func<TSource, int, IEnumerable<TResult>> selector);
    public IEnumerable<TResult> (IEnumerable<TSource> source).SelectMany<TSource, TResult>(Func<TSource, IEnumerable<TResult>> selector);
    public IEnumerable<TResult> (IEnumerable<TSource> source).SelectMany<TSource, TCollection, TResult>(Func<TSource, IEnumerable<TCollection>> collectionSelector, Func<TSource, TCollection, TResult> resultSelector);

    public bool (IEnumerable<TSource> first).SequenceEqual<TSource>(IEnumerable<TSource> second);
    public bool (IEnumerable<TSource> first).SequenceEqual<TSource>(IEnumerable<TSource> second, IEqualityComparer<TSource>? comparer);

    public TSource (IEnumerable<TSource> source).Single<TSource>();
    public TSource (IEnumerable<TSource> source).Single<TSource>(Func<TSource, bool> predicate);

    public TSource? (IEnumerable<TSource> source).SingleOrDefault<TSource>();
    public TSource (IEnumerable<TSource> source).SingleOrDefault<TSource>(TSource defaultValue);
    public TSource? (IEnumerable<TSource> source).SingleOrDefault<TSource>(Func<TSource, bool> predicate);
    public TSource (IEnumerable<TSource> source).SingleOrDefault<TSource>(Func<TSource, bool> predicate, TSource defaultValue);

    public IEnumerable<TSource> (IEnumerable<TSource> source).Skip<TSource>(int count);

    public IEnumerable<TSource> (IEnumerable<TSource> source).SkipLast<TSource>(int count);

    public IEnumerable<TSource> (IEnumerable<TSource> source).SkipWhile<TSource>(Func<TSource, bool> predicate);
    public IEnumerable<TSource> (IEnumerable<TSource> source).SkipWhile<TSource>(Func<TSource, int, bool> predicate);

    public float (IEnumerable<TSource> source).Sum<TSource>(Func<TSource, float> selector);
    public int (IEnumerable<TSource> source).Sum<TSource>(Func<TSource, int> selector);
    public long (IEnumerable<TSource> source).Sum<TSource>(Func<TSource, long> selector);
    public decimal? (IEnumerable<TSource> source).Sum<TSource>(Func<TSource, decimal?> selector);
    public double (IEnumerable<TSource> source).Sum<TSource>(Func<TSource, double> selector);
    public int? (IEnumerable<TSource> source).Sum<TSource>(Func<TSource, int?> selector);
    public long? (IEnumerable<TSource> source).Sum<TSource>(Func<TSource, long?> selector);
    public float? (IEnumerable<TSource> source).Sum<TSource>(Func<TSource, float?> selector);
    public double? (IEnumerable<TSource> source).Sum<TSource>(Func<TSource, double?> selector);
    public decimal (IEnumerable<TSource> source).Sum<TSource>(Func<TSource, decimal> selector);
    public double? (IEnumerable<double?> source).Sum();
    public float? (IEnumerable<float?> source).Sum();
    public long? (IEnumerable<long?> source).Sum();
    public int? (IEnumerable<int?> source).Sum();
    public decimal? (IEnumerable<decimal?> source).Sum();
    public long (IEnumerable<long> source).Sum();
    public int (IEnumerable<int> source).Sum();
    public double (IEnumerable<double> source).Sum();
    public decimal (IEnumerable<decimal> source).Sum();
    public float (IEnumerable<float> source).Sum();

    public IEnumerable<TSource> (IEnumerable<TSource> source).Take<TSource>(Range range);
    public IEnumerable<TSource> (IEnumerable<TSource> source).Take<TSource>(int count);

    public IEnumerable<TSource> (IEnumerable<TSource> source).TakeLast<TSource>(int count);

    public IEnumerable<TSource> (IEnumerable<TSource> source).TakeWhile<TSource>(Func<TSource, int, bool> predicate);
    public IEnumerable<TSource> (IEnumerable<TSource> source).TakeWhile<TSource>(Func<TSource, bool> predicate);

    public IOrderedEnumerable<TSource> (IOrderedEnumerable<TSource> source).ThenBy<TSource, TKey>(Func<TSource, TKey> keySelector);
    public IOrderedEnumerable<TSource> (IOrderedEnumerable<TSource> source).ThenBy<TSource, TKey>(Func<TSource, TKey> keySelector, IComparer<TKey>? comparer);

    public IOrderedEnumerable<TSource> (IOrderedEnumerable<TSource> source).ThenByDescending<TSource, TKey>(Func<TSource, TKey> keySelector);
    public IOrderedEnumerable<TSource> (IOrderedEnumerable<TSource> source).ThenByDescending<TSource, TKey>(Func<TSource, TKey> keySelector, IComparer<TKey>? comparer);

    public TSource[] (IEnumerable<TSource> source).ToArray<TSource>();

    public Dictionary<TKey, TElement> (IEnumerable<TSource> source).ToDictionary<TSource, TKey, TElement>(Func<TSource, TKey> keySelector, Func<TSource, TElement> elementSelector, IEqualityComparer<TKey>? comparer) where TKey : notnull;
    public Dictionary<TKey, TElement> (IEnumerable<TSource> source).ToDictionary<TSource, TKey, TElement>(Func<TSource, TKey> keySelector, Func<TSource, TElement> elementSelector) where TKey : notnull;
    public Dictionary<TKey, TSource> (IEnumerable<TSource> source).ToDictionary<TSource, TKey>(Func<TSource, TKey> keySelector, IEqualityComparer<TKey>? comparer) where TKey : notnull;
    public Dictionary<TKey, TSource> (IEnumerable<TSource> source).ToDictionary<TSource, TKey>(Func<TSource, TKey> keySelector) where TKey : notnull;
    public Dictionary<TKey, TValue> (IEnumerable<(TKey Key, TValue Value)> source).ToDictionary<TKey, TValue>() where TKey : notnull;
    public Dictionary<TKey, TValue> (IEnumerable<(TKey Key, TValue Value)> source).ToDictionary<TKey, TValue>(IEqualityComparer<TKey>? comparer) where TKey : notnull;
    public Dictionary<TKey, TValue> (IEnumerable<(TKey Key, TValue Value)> source).ToDictionary<TKey, TValue>() where TKey : notnull;
    public Dictionary<TKey, TValue> (IEnumerable<(TKey Key, TValue Value)> source).ToDictionary<TKey, TValue>(IEqualityComparer<TKey>? comparer) where TKey : notnull;

    public HashSet<TSource> (IEnumerable<TSource> source).ToHashSet<TSource>();
    public HashSet<TSource> (IEnumerable<TSource> source).ToHashSet<TSource>(IEqualityComparer<TSource>? comparer);

    public List<TSource> (IEnumerable<TSource> source).ToList<TSource>();

    public ILookup<TKey, TElement> (IEnumerable<TSource> source).ToLookup<TSource, TKey, TElement>(Func<TSource, TKey> keySelector, Func<TSource, TElement> elementSelector);
    public ILookup<TKey, TElement> (IEnumerable<TSource> source).ToLookup<TSource, TKey, TElement>(Func<TSource, TKey> keySelector, Func<TSource, TElement> elementSelector, IEqualityComparer<TKey>? comparer);
    public ILookup<TKey, TSource> (IEnumerable<TSource> source).ToLookup<TSource, TKey>(Func<TSource, TKey> keySelector);
    public ILookup<TKey, TSource> (IEnumerable<TSource> source).ToLookup<TSource, TKey>(Func<TSource, TKey> keySelector, IEqualityComparer<TKey>? comparer);

    public bool (IEnumerable<TSource> source).TryGetNonEnumeratedCount<TSource>(out int count);

    public IEnumerable<TSource> (IEnumerable<TSource> first).Union<TSource>(IEnumerable<TSource> second);
    public IEnumerable<TSource> (IEnumerable<TSource> first).Union<TSource>(IEnumerable<TSource> second, IEqualityComparer<TSource>? comparer);

    public IEnumerable<TSource> (IEnumerable<TSource> first).UnionBy<TSource, TKey>(IEnumerable<TSource> second, Func<TSource, TKey> keySelector);
    public IEnumerable<TSource> (IEnumerable<TSource> first).UnionBy<TSource, TKey>(IEnumerable<TSource> second, Func<TSource, TKey> keySelector, IEqualityComparer<TKey>? comparer);

    public IEnumerable<TSource> (IEnumerable<TSource> source).Where<TSource>(Func<TSource, bool> predicate);
    public IEnumerable<TSource> (IEnumerable<TSource> source).Where<TSource>(Func<TSource, int, bool> predicate);
    
    public IEnumerable<(TFirst First, TSecond Second, TThird Third)> (IEnumerable<TFirst> first).Zip<TFirst, TSecond, TThird>(IEnumerable<TSecond> second, IEnumerable<TThird> third);
    public IEnumerable<(TFirst First, TSecond Second)> (IEnumerable<TFirst> first).Zip<TFirst, TSecond>(IEnumerable<TSecond> second);
    public IEnumerable<TResult> (IEnumerable<TFirst> first).Zip<TFirst, TSecond, TResult>(IEnumerable<TSecond> second, Func<TFirst, TSecond, TResult> resultSelector);
}
```