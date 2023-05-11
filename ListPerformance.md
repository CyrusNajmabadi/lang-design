# `List<T>` optimizations

Continuing the requirement from [collection literals](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md) specification that the `List<T>` be "well behaved" (i.e. it will not make observable changes outside of its own elements), the language permits a compliant compiler to replace fresh instances of `List<T>` within a method body with more efficient values (specifically, with different types), as long as such replacement is otherwise not observable.

