```c#
// can now write:
foreach (var string in someSpan.Where(v => v < 21).Select(v => v.ToString())
{

}

// New IEnumerable2 interface (which is generic on its TEnumerator type).  Calling it IEnumerable2 to make it clear
// which IEnumerable i'm talking about.  It could still be called IEnumerable since it has two type parameter.
interface IEnumerable2<TElement, TEnumerator> where TEnumerator : IEnumerator<TElement>
{
}

// Extensions on IEnumerable2's.  Importantly, when called on a struct (which most of these helper return), they need 
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

        public WhereSelectEnumerator Select<TResult>(Func<TElement, TResult> selector)
        {
            // Can return dedicated special type without indirection.  Open question if whether that is necessary.
            // If you have a standard SelectEnumerator<TResult> that wraps a WhereEnumerator, perhaps the runtime is
            // just going to be fast enough, as there will be no virtual calls.  In other words: It could write-through 
            // .Current on the SelectEnumerator all the way through to this WhereEnumerator, and perhaps it could inline 
            // SelectEnumerator.MoveNext as well.  If that's the case, we wouldn't need specialized types.
            return new(this, test, selector);
        }
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

    public SelectEnumerable<TResult> Select(Func<TElement, TResult> selector)
        => new(ref this, selector);

    public ref struct SelectEnumerable<TResult>(ref TEnumerable enumerable, Func<TElement, TResult> selector) : IEnumerable2<TResult, TSelectEnumerator<TResult>> {

        public SelectEnumerator GetEnumerator()
            => new(ref _enumerable.GetEnumerator(), selector);
    }

    public ref struct SelectEnumerator<TResult>(ref TEnumerator enumerator, Func<TElement, TResult> selector) : IEnumerator<REsult>
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