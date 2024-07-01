# Nullability approaches with the field keyword

This document will contrast several approaches under consideration for nullability warnings in properties that use the `field` keyword. This follows up from <https://github.com/dotnet/csharplang/blob/main/meetings/working-groups/field-keyword/FK-2024-05-26.md>.

## Approaches

### Approach A: backing field nullability matches property nullability

#### A1: Continue disregarding nullability attributes applied using `[field: ...]`

This approach requires users to reflexively use `= null!;` property initializers in order to tell the compiler that no constructor initialization is needed. This introduces a safety concern; once `= null!;` is on the property and the constructor warnings go away, `field` is actually still starting as null and yet is being analyzed as "not null here" at the start of each accessor:

```cs
// No warnings when adding `.Trim()` because `= null!` had
// previously been necessary!
public string Prop { get => field.Trim(); set; } = null!;
```

However, it should be rare to want to do anything interesting in the getter besides defaulting and lazy initialization. Everything else is better handled in the setter:

```cs
public string Prop { get; set => field = value.Trim(); }
```

#### A2: Respect nullability attributes applied using `[field: ...]`

This is a change:
`[field: AllowNull] string Prop { get; } = null;`

### Approach B: Backing field is always nullable

#### B1: No additional analysis

TODO - contrast along with others

#### B2: Cross-accessor analysis

TODO - contrast along with others

### Approach C: Backing field is nullable if the getter is null-resilient

A null-resilient getter is a getter which never returns a maybe-null expression in any exit path when `field` is assumed to be maybe-null when the getter is called.

This is an important concept because it shows that the property upholds its not-null contract to callers without requiring `field` to have been initialized to a non-null value. This leaves `field` free to start as null and return to null in any accessor without breaking the property's nullability contract.

As the term 'null-resilient getter' implies, the null resilience being considered is an aspect of the getter. It's not an aspect of the code inside the getter when the code is divorced from the context of the getter.

These getters are null-resilient:

```cs
get => field ?? "";
get => field ??= new();
get => LazyInitializer.EnsureInitialized(ref field, ...);
get => field ?? throw new InvalidOperationException(...);
get => throw new NotImplementedException();
get => field.ToString();
get => field!;
```

These getters are not null-resilient:

```cs
get;
get => field;
get => field?.ToString();
get => field ?? SomethingNullable;
```

## Comparisons

### Lazy initialization

```cs
class C
{
    public List<int> Prop1 => field ??= new();

    public Foo Prop2 => LazyInitializer.EnsureInitialized(ref field, CalculateFoo);
}
```

With approaches B and C, this compiles without warnings.

With approach A, there is a warning in each constructor (here, in the implicit default constructor) that each property must be assigned a value in the constructor.

With A1, the way to tell the compiler that this is truly valid code would be to use a property initializer:

```cs
public List<int> Prop1 { get => field ??= new(); } = null!;
```

With A2, something more semantically appropriate becomes available also. The constructors stop requiring `Prop1` to be assigned in the constructor, and `field` is checked as maybe-null in the body which results in safer analysis if the getter is modified in the future:

```cs
[field: MaybeNull]
public List<int> Prop1 => field ??= new();
```

### Default if reset to default

```cs
class C
{
    public string Prop
    {
        get => field ?? parent.Prop;
        set => field = value == parent.Prop ? null : value;
    }
}
```

With approaches B and C, this compiles without warnings.

With approach A, there are the same warnings per constructor requiring the property to be initialized as in the last example. There is a further warning that `null` should not be assigned to `field`, even though in this example this is both useful and safe.

With A1, the workaround would be to use `null!` instead of `null` and add the `= null!;` property initializer.

With A2, the workaround would be to add `[field: MaybeNull, AllowNull]`. This is semantically equivalent to defining the field as `string?` rather than `string`. The `MaybeNull` resolves the constructor initialization warnings, and the `AllowNull` resolves the warning when attempting to set `field` to `null`.

## [...]
