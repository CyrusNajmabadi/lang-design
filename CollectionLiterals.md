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

Extensions do not allow adding fields or destructors to a type.

Given an existing static class with extensions, a straightforward translation to modern extensions is done in the following fashion.

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

    // Existing extensions drop 
    int ExtensionMethod(...) for string x { }
    T GenericExtensionMethod<T, U>(...) for U u { }
}
```

In other words, all existing extension methods drop `static` from their signature, and move their first parameter to a `for clause` placed within the method header (currently strawmanned as after the parameter list).  This location cleanly supports clauses, being already where the type parameter constraint clauses go.  The extension itself (E) will get emitted exactly as a static class would be that contains extension methods (allowing usage from older compilers and other languages without any updates post this mechanical translation).

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
var v1 = ((Extension)receiver).ExtensionProperty;
var v2 = ((Extension)receiver)[indexerArg];
var v3 = (Extension)receiver1 + receiver2;
```

Constructors and static methods would not need any special syntax as the extension can cleanly be referenced as a type where needed.

```c#
var v3 = new Extension(...); // Makes instance of the actual extended type.
var v4 = Extension.StaticExtensionMethod(...);
```

Note: while the cast syntax traditionally casts or converts a value, that would not be the case for its use here.  It would only be used as a lookup mechanism to indicate which extension gets priority.  Importantly, even with this syntax, extensions themselves are not types.  For example:

```c#
Extension e1;                   // Not legal.  Extension is not a type.
Extension[] e2;                 // Not legal.  Extension is not a type.
List<Extension> e3;             // Not legal.  Extension is not a type.
var v1 = (Extension)receiver;   // Not legal.  Can't can't have a value of extension type.
```

This is exactly the same as the restrictions on static-types *except* with the carve out that you can use the extension in a cast-syntax or new-expression *only* for lookup purposes and nothing else.

## Future expansion



## Detailed design
[design]: #detailed-design
