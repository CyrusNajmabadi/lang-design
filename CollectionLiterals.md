# Modern Extensions

* [x] Proposed
* [ ] Prototype: Not Started
* [ ] Implementation: Not Started
* [ ] Specification: Not Started

Many thanks to those who helped with this proposal.  Esp. @jnm2!

## Summary
[summary]: #summary

Modern Extensions introduce a new syntax to produce extension members, greatly expanding on the set of supported members (including properties, static methods and constructors) in a clean and cohesive fashion. This new form subsumes C# 3's "classic extension methods", allowing migration to the new modern form in semantically *identical* fashion (both at a source and ABI level).

A rough strawman of the syntax is as follows.  In all cases, the extended type is shown to be generic, to indicate handling that complex case:

Note: the syntax is intentionally a strawman to aid in discussion.  It is not intended to be final form.

```c#
extension E
{
    // Method form, replaces `public static T1 M<X>(this T2 p, ...) { } 
    public int M<X>(...) for SomeType<X> { }

    // Property form:
    public int Count<X> for SomeType<X> { get { ... } }

    // Indexer form:
    public int this<T>[int index] for SomeType<X> { get { ... } }

    // Operator form:
    public static SomeTypeX<X> operator+ <X>(SomeType<X> s1, SomeType<X> s2) for SomeType<X> { ... }

    // Constructor form:
    public static SomeType<X> new<X>() for SomeType<X> { ... }

    // *Static* extension method (not possible today).  Called as `T.Assert("", "")`
    public static bool Assert(string s1, string s2) for T { }

}
```

Modern extensions continue to not allow adding fields or destructors to a type.

Given an existing static class with extensions, a straightforward *semantically identical* translation to modern extensions is done in the following fashion.

```c#
// Existing style
static class E
{
    static TField field;
    static void NonExtensionHelperMethod() { }
    static int Property => ...

    static int ExtensionMethod(this string x, ...) { }
    static T GenericExtensionMethod<T, U>(this U u, ...) { }
}

// New style
extension E
{
    // Non extensions stay exactly the same:
    static TField field;
    static void NonExtensionHelperMethod() { }
    static int Property => ...

    int ExtensionMethod(...) for string x { }
    T GenericExtensionMethod<T, U>(...) for U u { }
}
```

In other words, all existing extension methods drop `static` from their signature, and move their first parameter to a `for-clause` placed within the method header (currently strawmanned as after the parameter list).  Note: the syntax of a `for-clause` is `for parameter`, allowing things like a parameter name to be specified.  `parameter` is critical in this design to ensure the classic extension method `this` parameter can always cleanly move.

This location cleanly supports clauses, being already where the type parameter constraint clauses go.  The extension itself (E) will get emitted exactly as a static class would be that contains extension methods (allowing usage from older compilers and other languages without any updates post this mechanical translation).

New extension members (beyond instance members) would have to have their metadata form decided on.  Consumption from older compilers and different languages of these new members will be specified at a later point in time.

A full example of this translation with a real world complex signature would be:

```c#
static class Enumerable
{
    public static TResult Sum<TSource, TResult, TAccumulator>(this IEnumerable<TSource> source, Func<TSource, TResult> selector)
        where TResult : struct, INumber<TResult>
        where TAccumulator : struct, INumber<TAccumulator>
    {
        // Original body
    }
}

extension Enumerable
{
    public TResult Sum<TSource, TResult, TAccumulator>(Func<TSource, TResult> selector)
        for IEnumerable<TSource> source
        where TResult : struct, INumber<TResult>
        where TAccumulator : struct, INumber<TAccumulator>
    {
        // Exactly the same code as original body.
    }
}
```

## Disambiguation

Classic extension methods today can be disambiguated by falling back to static-invocation syntax.  For example, if `x.Count()` is ambiguous, it is possible to switch to some form of `StaticClass.Count(x)` to call the desired method.  A similar facility is needed for modern extension methods.  While the existing method-invocation-translation approach works fine for methods (where the receiver can be remapped to the first argument of the static extension method call), it is ungainly for these other extension forms.

As an initial strawman this proposal suggests reusing `cast expression` syntax for disambiguation purposes.  For example:

```c#
var v1 = ((Extension)receiver).ExtensionMethod(); // instead of Extension.ExtensionMethod(receiver)
var v2 = ((Extension)receiver).ExtensionProperty;
var v3 = ((Extension)receiver)[indexerArg];
var v4 = (Extension)receiver1 + receiver2;
```

Constructors and static methods would not need any special syntax as the extension can cleanly be referenced as a type where needed.

```c#
var v3 = new Extension(...); // Makes instance of the actual extended type.
var v4 = Extension.StaticExtensionMethod(...);
```

Note 1: while the cast syntax traditionally casts or converts a value, that would not be the case for its use here.  It would only be used as a lookup mechanism to indicate which extension gets priority.  Importantly, even with this syntax, extensions themselves are not types.  For example:

```c#
Extension e1;                   // Not legal.  Extension is not a type.
Extension[] e2;                 // Not legal.  Extension is not a type.
List<Extension> e3;             // Not legal.  Extension is not a type.
var v1 = (Extension)receiver;   // Not legal.  Can't can't have a value of extension type.
```

This is exactly the same as the restrictions on static-types *except* with the carve out that you can use the extension in a cast-syntax or new-expression *only* for lookup purposes and nothing else.

Note 2. If cast syntax is not desirable here (especially if confuses the idea if extensions are types), we can come up with a new syntactic form.  We are not beholden to the above syntax.

# Future expansion

The above initial strawman solves several major goals for we want for the extensions space:

1. Supporting a much broader set of extension member types.
2. Having a clean syntax for extension members that matches the regular syntax form (in other words, an extension proeprty still looks like a property).
3. Ensuring teams can move safely to modern extensions *especially* in environments where source binary compatibility is non-negotiable.

However, there are parts of its core design that are not ideal which we would like to ensure we can expand on.  These expansions could be released with extensions if time and other resources permit.  Or they could came later and cleanly sit on top of the feature to improve the experience.

These areas are:

## Expansion 1: Syntactic clumsiness and repetition

The initial extension form values source and binary compatibility as core requirements that must be present to ensure easy migration, allowing codebases to avoid both:
1. bifurcation; where some codebases adopt modern extensions and some do not.
2. internal inconsistency; where some codebases must keep around old extensions and new extensions, with confusion about the semantics of how each interacts with the other.

Because classic extension methods have very few restrictions, modern extension methods need to be flexible enough to support all the scenarios supported there.

However, many codebases do not need all the flexibility that classic extension methods afforded.  For example, classic extension methods allow disparate extension methods in a single static class to target multiple different types.  For use cases where that isn't required, we forsee a natural extension (pun intended) where one can modern translate extensions like so:

```c#
extension E
{
    // All extension members extend the same thing:

    public void M() for SomeType { ... }
    public int P for SomeType { get { ... } }
    public static operator+(...) for SomeType { ... }
    // etc
}

// Can be translated to:

extension E for SomeType
{
    public void M() { ... }
    public int P { get { ... } }
    public static operator+(...) { ... }
}
```

TODO: Do an ecosystem check on what percentage of existing extensions could use this simpler form.

TODO: It's possible someone might have an extension where almost all extensions extend a single type, and a small handful do something slightly different (perhaps extending by `this ref`).  Would it be beneficial here to *still* allow the extension members to provide a `for-clause` to override that default for that specific member.  For example:

```c#
extension StringExtensions for string
{
    // Lots of normal extension methods on string ...

    // Override here to extend `string?`
    public bool MyIsNullOrEmpty() for [NotNullWhen(false)] string? str
    {
    }
}
```

It seems like this would be nice to support with little drawback.

## Expansion 2: Optional syntactic components

As above, we want modern extensions to completely subsume classic extension methods.  As such, a modern extension  method must be able to support everything a classic extension method supported.  For example:

```c#
static class Extensions
{
    // Yes, this is legal
    public static void MakeNonNull([Attr] this ref int? value)
    {
        if (value is null)
            value = 0;
    }
}
```

For this reason, the strawman syntax for this `for clause` is `for parameter`, where `parameter` is the familiar:

```g4
parameter
    | attributes? modifiers? type identifier
    ;
```

(fortunately, extension methods today don't support a default value for the `this` parameter, so wel don't have to support that).

However, for many extensions no name is really required.  All extension members (except for extension-constructors and extension-static-methods) are conceptually a way to extend `this` with new functionality.  This is so much so the case that we even designed classic extension methods to use the `this` keyword as their designator.  As such, we forsee potentially making the name optional, allowing one to write an extension like so:

```c#
extension Enumerable
{
    public TResult Sum<TSource, TResult, TAccumulator>(Func<TSource, TResult> selector)
        for IEnumerable<TSource> // no name
        where TResult : struct, INumber<TResult>
        where TAccumulator : struct, INumber<TAccumulator>
    {
        // Use 'this' in here to represent the value being extended
    }
}
```

This would have to come with some default name chosen by the language for the parameter in metadata.  But that never be needed by anyone calling it from a modern compiler.

## Expansion 3: Generic extensions.

The initial design allows for extending generic types through the use of generic extension members.  For example:

```c#
extension IListExtensions
{
    public void ForEach<T>(Action<T> act) for IList<T> list
    {
        foreach (var value in list)
            act(list);
    }

    public long LongCount<T> for IList<T> list
    {
        get
        {
            long count = 0;
            foreach (var value in list)
                count++;

            return count;
        }
    }
}
```

Ideally with the optional first expansion we could 'lift' `List<T>` up to `extension IListExtensions`.  However, this doesn't work as we need to define the type parameter it references.  This naturally leads to the following idea:

```c#
extension IListExtensions<T> for IList<T> list
{
    public void ForEach(Action<T> act) { ... }
    public long LongCount { ... }
}
```

This has a few new, but solvable, design challenges.  For example, say one has the code:

```c#
List<int> ints = ...;
var v = ints.ForEach(i => Console.WriteLine(i));
```

This naturally raises the question of how does this extension get picked for this particular receiver, and how does its type parameter get instantiated to the `int` type.

Conceptually (and trying to keep somewhat in line with classic extension methods), we really want to think of the 'receiver' as an 'argument' to some method where normal type inference occurs.  Morally, we could think of there being a `IListExtension<T> Infer<T>(IList<T> list)` function whose shape is determined by the extension and its type-parameters and the extended receiver parameter.

Then, when trying to determine if an extension applies to a receiver, it would be akin to calling that function with the receiver and seeing if inference works.  In the above example that would mean performing type inference on `Infer(ints)` seeing that `T` then bound to `int`, which then gives you back `IListExtensions<int>`.  At that point, lookup would then find and perform overload resolution on `ForEach(Action<int>)` with the lambda parameter.

This approach does fundamentally take expand on the initial extension-members approach as now calling extensions is done in two-phases.  An initial phase to determine and infer extension type parameters based on the receiver, and  a second phase to determine and perform overload resolution on the member.

We believe this is very powerful and beneficial.  But there are deep design questions here which may cause this to be scheduled after the core extension members work happens.

## Detailed design
[design]: #detailed-design
