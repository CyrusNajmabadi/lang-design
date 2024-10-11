```c#
interface IEnumerable2<TElement, TEnumerator> where TEnumerator : IEnumerator<TElement>
{
}

extension Enumerable2Extensions<TEnumerable, TElement, TEnumerator> where TEnumerable : IEnumerable2<TElement, TEnumerator>
{
    // if we don't have yield-support could write as:

    public WhereEnumerable Where(Func<TElement, bool> test)
        => new(this, test);

    public ref struct WhereEnumerable(ref TEnumerable enumerable, Func<TElement, bool> test) : IEnumerable2<TElement, TWhereEnumerator>
    {
        // Ref struct, so we don't get a copy in here, this can just point up the stack as necessary.
        private ref TEnumerable _enumerable = enumerable;

        public WhereEnumerator GetEnumerator()
            => new(ref _enumerable.GetEnumerator(), test);

        public WhereSelectEnumerator Select<TResult>(Func<TElement, TResult> selector)
        {
            // Can return dedicated special type without indirection.
        }
    }

    public ref struct WhereEnumerator(ref TEnumerator enumerator, Func<TElement, bool> test) : IEnumerator<TElement>
    {
        // Ref struct, so we don't get a copy in here, this can just point up the stack as necessary.
        private ref TEnumerator _enumerator = enumerator;

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
        => new(this, selector);

    public ref struct SelectEnumerable<TResult>(ref TEnumerable enumerable, Func<TElement, TResult> selector) : IEnumerable2<TResult, TSelectEnumerator<TResult>> {

        public SelectEnumerator GetEnumerator()
            => new(ref _enumerable.GetEnumerator(), selector);
    }

    public ref struct SelectEnumerator<TResult>(ref TEnumerator enumerator, Func<TElement, TResult> selector) : IEnumerator<REsult>
    {
        // Ref struct, so we don't get a copy in here, this can just point up the stack as necessary.
        private ref TEnumerator _enumerator = enumerator;

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