# Collection literals natural type


### Goals:

1. Support literal usage like `var v = [x, y, ..z];` (elements provided within the literal).
1. Support literal usage like `var v = []; /*...*/ v.Add(x);` (empty literal with elements provided afterwards).
1. Provide broad leeway for compiler to produce heavily optimized code (i.e. 'do not leave perf on the table').
1. Support broad number of user cases in a natural/intuitive fashion.

### Non-goals:

1. Introduce a new *real* type that users would explicitly use themselves.  While there may be new helper types introduced in the BCL (ideally in System.Compiler.RuntimeServices), they may have advanced/confusing semantics and might only be intended for compilers to use. 
1. Have no 'cracks'.  Complex scenarios may reveal non-ideal semantics.  (e.g. how `T?` in unconstrained generics doesn't cleanly work with reference and value types).

## Detailed design:

For the purposes of *discussion/design* only, a new language-only type is introduced called ``anonymous_list`<T>``.  Similar to [anonymous types](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#117157-anonymous-object-creation-expressions), this is not a type that can be directly referenced by the user, but has semantics the language defines, and which a compiler is free to implement or emit however it wants as long as those semantics are preserved.

In code samples ``anonymous_list`<T>`` will be used to indicate that this it the type of a variable in place of the `var` that a user would have to write in real code.  This will clarify that the `var` is not some other type (like `Span<T>`, `T[]`, `List<T>`, etc.), while also helping see what the element type `T` is in a particular context.  For example: ``anonymous_type`<int> v = [1, 2, 3];``

