```c#
// Goal: one can write, and have that be non-allocating (for the enumerables/enumerators).
// Making the delegates non-allocating happens if they do not capture.  An extension to avoid
// allocs even if they capture is shown at the end.
foreach (var string in someSpan.Where(v => v < 21).Select(v => v.ToString())
{

}

// How?

// 1. Introduce IEnumerable2 interface (which is generic on its TEnumerator type).  Calling it IEnumerable2
// in this doc to make it clear which IEnumerable is being talked about.  It could still be called IEnumerable
// if it ships since it has two type parameters.
interface IEnumerable2<TElement, TEnumerator> where TEnumerator : IEnumerator<TElement>
{
}

// 1. Add extensions on IEnumerable2's.  Importantly, when called on a struct (which all of these helper methods return), they need 
// to take 'by reference' (i.e. ref-this) so that mutation methods call-through to the original.  In other words, if 
// we have a SelectEnumerator wrapping a WhereEnumerator.  Then MoveNext on the SelectEnumerator needs to call through
// to the WhereEnumerator.MoveNext and have it actually mutate it.
extension Enumerable2Extensions<TEnumerable, TElement, TEnumerator> where TEnumerable : IEnumerable2<TElement, TEnumerator>
{
    // Until we got iterator/yield support we would write the extensions as:

    // Similar to Enumerable.Where.  But gives a stack-based enumerable which gives a stack based enumerator.
    public WhereEnumerable Where(Func<TElement, bool> test)
        => new(ref this, test);

    public ref struct WhereEnumerable(ref TEnumerable enumerable, Func<TElement, bool> test) : IEnumerable2<TElement, TWhereEnumerator>
    {
        // Ref struct, so we don't get a copy in here, this can just point up the stack as necessary.
        private ref TEnumerable _enumerable = enumerable;

        public WhereEnumerator GetEnumerator()
            => new(ref _enumerable.GetEnumerator(), test);

        // Possible, but hopefully not necessary.
        public WhereSelectEnumerator Select<TResult>(Func<TElement, TResult> selector)
        {
            // Can return dedicated special type without indirection.  Open question if whether that is necessary.
            // If you have a standard SelectEnumerator<TResult> that wraps a WhereEnumerator, perhaps the runtime is
            // just going to be fast enough as there will be no virtual calls.  In other words: It could read/write through 
            // .Current on the SelectEnumerator all the way through to this WhereEnumerator, and perhaps it could inline 
            // SelectEnumerator.MoveNext as well.  If that's the case, we wouldn't need specialized types.
            return new WhereSelectEnumerator(ref this, test, selector);
        }

        public ref struct WhereEnumerator(ref TEnumerator enumerator, Func<TElement, bool> test) : IEnumerator<TElement>
        {
            // Ref struct, so we don't get a copy in here, this can just point up the stack as necessary.
            private ref TEnumerator _enumerator = ref enumerator;

            public TElement? Current { get; private set }

            public bool MoveNext()
            {
                while (_enumerator.MoveNext())
                {
                    var current = _enumerator.Current;
                    if (test(_enumerator.Current))
                    {
                        this.Current = current;
                        return true;
                    }
                }

                this.Current = default;
                return false;
            }
        }
    }

    // Similar to Enumerable.Select.  But gives a stack-based enumerable which gives a stack based enumerator.
    public SelectEnumerable<TResult> Select<TResult>(Func<TElement, TResult> selector)
        => new(ref this, selector);

    public ref struct SelectEnumerable<TResult>(ref TEnumerable enumerable, Func<TElement, TResult> selector) : IEnumerable2<TResult, TSelectEnumerator<TResult>> {

        public SelectEnumerator GetEnumerator()
            => new(ref _enumerable.GetEnumerator(), selector);

        public ref struct SelectEnumerator(ref TEnumerator enumerator, Func<TElement, TResult> selector) : IEnumerator<TResult>
        {
            // Ref struct, so we don't get a copy in here, this can just point up the stack as necessary.
            private ref TEnumerator _enumerator = ref enumerator;

            public TElement? Current { get; private set }

            public bool MoveNext()
            {
                if (_enumerator.MoveNext())
                {
                    this.Current = selector(_enumerator.Current);
                    return true;
                }

                this.Current = default;
                return false;
            }
        }
    }

    // Ideally, one could write above with iterators/yield.  A major decision would have to be made if you had a guaranteed
    // nominal type as part of your abi or not.  If a nominal type was required, the following strawman would be possible:

    // `iterator(element_type, synthesized_type_name)`. 
    //
    // A `ref struct WhereEnumerable : IEnumerable<TElement, WhereEnumerable.Enumerator> { ref struct Enumerator { ... } ... }`
    // would be synthesized
    public iterator(TElement, WhereEnumerable) Where(Func<TElement, bool> test)
    {
        foreach (var value in this)
            if (test(value))
                yield return value;
    }

    // A `ref struct SelectEnumerable<TResult> : IEnumerable<TResult, SelectEnumerable<TResult>.Enumerator> { ref struct Enumerator { ... } ... }`
    // would be synthesized
    public iterator(TResult, SelectEnumerable<TResult>) Select<TResult>(Func<TElement, bool> test)
    {
        foreach (var value in this)
            yield return test(value);
    }

    // Other strawmen might be: `struct iterator(element_type, synthesized_type_name)` or `ref struct iterator(element_type, synthesized_type_name)`
    // (including 'struct' or 'ref struct' to be clear on that).
    //
    // Whatever we create here needs to indicate at least the outer type name (the inner type can always be called OuterTypeName.Enumerator),
    // and needs to have enough to reconstruct `ref struct NewEnumerable : IEnumerable2<NewElement, NewEnumerable.Enumerator>`.
    // so it needs bit in the syntax for NewEnumerable/NewElement, and potentially bits for the ref-ness of the struct.  Or we always give
    // you that, and if you don't want that, you use a normal iterator.
    //
    // Other strawmen:
    // public iterator<TElement> as WhereEnumerable Where(...)
    // public iterator<TElement> Where(...) named WhereEnumerable

    // If nominal abi guarantees are not necessary then we could simplify to `iterator(element_type, *optional*_typename)`:
    //
    // This would give you an unnamed struct you could use in places like var/foreach.  But which you could not name or put in your own abi.
    // In particular, the name of this could change, which could lead to binary breaks.

    public trait iterator(TElement) Where(Func<TElement, bool> test)
    {
        foreach (var value in this)
            if (test(value))
                yield return value;
    }

    public trait iterator(TResult) Select<TResult>(Func<TElement, bool> test) 
    {
        foreach (var value in this)
            yield return test(value);
    }
}

## No allocs at all?

// What if we want no alloc at all (even delegates).  Can the runtime do that automatically with lambdas?  It knows the 
// .Where/.Select calls are only on stack objects that can't capture the delegate.  So perhaps the new-capture and new-delegate
// can become stack-based.  Otherwise, we'd need something like:

ref struct delegate TResult RefFunc<T, TResult>(T value);

extension VeryVeryFastEnumerable<TEnumerable, TElement, TEnumerator> where TEnumerable : IEnumerable2<TElement, TEnumerator>
{
       public WhereEnumerable Where<TRefFunc>(ref TRefFunc test) where TRefFunc : RefFunc<TElement, bool>
        => new(ref this, ref test);

    public ref struct WhereEnumerable<TRefFunc>(
        ref TEnumerable enumerable,
        ref TRefFunc test) :
        IEnumerable2<TElement, TWhereEnumerator>
        where TRefFunc : RefFunc<TElement, bool>
    {
        // Ref struct, so we don't get a copy in here, this can just point up the stack as necessary.
        private ref TEnumerable _enumerable = ref enumerable;
        private ref TRefFunc _test = ref test;

        public WhereEnumerator GetEnumerator()
            => new(ref _enumerable.GetEnumerator(), ref test);

        public ref struct Enumerator(
            ref TEnumerator enumerator,
            ref TRefFunc test) :
            IEnumerator<TElement>
        {
            // Ref struct, so we don't get a copy in here, this can just point up the stack as necessary.
            private ref TEnumerator _enumerator = ref enumerator;
            private ref TRefFunc _test = ref test;

            public TElement? Current { get; private set }

            public bool MoveNext()
            {
                while (_enumerator.MoveNext())
                {
                    var current = _enumerator.Current;
                    if (test(_enumerator.Current))
                    {
                        this.Current = current;
                        return true;
                    }
                }

                this.Current = default;
                return false;
            }
        }
    }
}





//

    public iterator(WhereEnumerable, TElement) Where<TRefStructFunc>(ref TRefStructFunc test) where TRefStructFunc : IFunc<TElement, bool>
    {
        foreach (var value in this)
            if (test(value))
                yield return value;
    }