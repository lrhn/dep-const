# Optional `const` prefix in const context

## Contact information

Name: Lasse R.H. Nielsen  
E-mail: lrn@google.com  
[DEP Proposal Location][]  
Further stakeholders:  
- Bob Nystrom - rnystrom@google.com
- Lars Bak - bak@google.com

## Summary

Allow omitting `const` inside a known const expression where it's redundant.

## Motivation
Currently "const" in an expression is a prefix used for some const expressions (literal list or map) and for calling const constructors (`const Foo(42)`).

This makes `const []` a compile-time constant, and `[]` without the leading "const" is not a constant.
Since compound compile time constants' sub-expressions (list elements, map keys and values, constructor parameters) must also be compile-time constant expressions, this means that a list of lists must be written `const [const [42, 37], const [87, 23]]`.
The inner "const" are required by the language specification, but they don't actually provide any extra information because a non-const list in the same position is not allowed by the language syntax. The extra occurrences of `const` are redundant.

This proposal makes these redundant inner `const` optional, so you can write: `const [[42, 37], [87, 23]]` without extra unnecessary `const` prefixes. This solves [Issue 4046][].

## Const Contexts
The proposal is to make the "const" prefix optional on list/map/constructor expressions where it's currently mandatory. It introduces the concept of an expression being in a "const context".
A const context is introduced by a "const" prefixed list/map/constructor expression, and extends to all sub-expressions (elements, keys, values and parameters). It is also introduced by other syntax productions that require a sub-expression to be a compile-time constant.

## Const Insertion
Inside a const context, some expressions are interpreted differently than they would be outside of an const context:

* Any list literal `<T>[...]` (with or without type parameter) in a const context is a const list literal, with the same meaning as the current `const <T>[...]` syntax.
* Any map literal `<K,V>{...}` (with or without type parameters) in a const context is a const map literal, with the same meaning as the current `const <K,V>{...}`.
* Any const constructor invocation without a `const`, whether `Foo<T>(...)`, `Foo<T>.bar(...)`, or `prefix.Foo<T>.bar(...)` (with ot without type parameters) where `Foo`/`Foo.bar` are const constructors, has the same meaning as the current `const Foo<T>(...)`, `const Foo<T>.bar(...)` and `const prefix.Foo<T>.bar(...)` expressions respectively, and all parameter expressions are also in a const context. Any function call that isn't a constructor invocation is still a compile-time error in a const context.

These rules can be seen as describing an "automatic const insertion strategy" that can be used to convert the new syntax to the existing syntax without changing the meaning of expressions. It allows omitting "const" from list/map/constructor expression when the expression is already required to be a (potentially) compile-time constant expression.

There is a precedence for the constructor invocation syntax: Annotations use the same syntax prefixed by "@" instead of "const", so "@Foo<T>(42)" is interpreted as "const Foo<T>(42)", as if the invocation was in a const context (except that parameters must still be marked as const).
With the "const context" concept introduced, it would be possible to say that annotations are always in a const context introduced by the "@".

## Syntactic Const Contexts
Other source locations that only allow compile time constant expression, could also be automatically be in a const context. The const-requiring locations are:

* Annotation expressions.
* The initializer expression of const variable declarations.
* The default values of optional parameters.
* The case expressions of a switch.
* The right-hand sides of initializer list entries of const constructors.

## Compatibility
The proposal is backwards compatible.  Existing valid code will still be valid and have the same meaning.

In all cases where the "const" prefix is made optional, omitting it would currently be a compile time error. 

Parsing will be complicated - a const-less const constructor invocation with two (or more) type parameters may be syntactically indistinguishable from two comparison expressions:

    const [Foo<int, int>(42)]

This is the same problem that the current proposal for generic functions is suffering from, and it should be solved in the same way (if that is solved).
If we can't find a solution, we should probably not make the `const` optional on expressions like this, only where there are no parsing ambiguities.

## Implementation

### Specification
The Dart specification must be updated to allow const constructor invocations without the initial const prefix.

It may be easier to have a separate limited grammar for const expressions, instead of allowing `Foo<int>(42)` everywhere in the grammar, and then declare it a syntax error if the syntax occurs outside of a const context.

### Implementations
Implementing this proposal requires changes to all Dart parsers to match the changes in the specification.

In most cases, the necessary changes should be restricted to the parser. Further compiler steps should treat the const-less expression as if it has a prefixed `const`, and work normally on that.

## Variations
### Const Constructor Initializer Lists Generalization

The initializer lists of const constructors can be treated specially: When used to create a compile-time constant, the expressions are treated as in a const context, and when they are used to create a runtime instance, the expressions are treated as non-const (with constructor calls having an implicit "new" instead of the missing "const").
There is a number of issues related to this: [Issue 20962][], [Issue 22329][]

Even without this improvement, the proposal still makes sense.

### Const Function Expressions
Top-level and static functions are compile-time constants, but function expressions are not (see [Issue 4596][]). It would be possible to make some function expressions in a const context be compile-time constants (only if they don't refer to any non-static variables - basically if they can be converted to a top-level/static function and the expression replaced by a reference to that function). Since a parameter default value is already in const context, you would be able to write a literal expression directly, solving issue 4596. More generally we could allow "const functionExpression" as an expression, and make the "const" optional in a const context.

Even without this improvement, the proposal still makes sense.

## Patents rights

TC52, the Ecma technical committee working on evolving the open [Dart standard][], operates under a royalty-free patent policy, [RFPP][] (PDF). This means if the proposal graduates to being sent to TC52, you will have to sign the Ecma TC52 [external contributer form][form] and submit it to Ecma.

[Issue 4046]: http://dartbug.com/4046
[Issue 4596]: http://dartbug.com/4596
[Issue 20962]: http://dartbug.com/20962
[Issue 22329]: http://dartbug.com/22329
[DEP Proposal Location]: https://github.com/lrhn/dep-const/
[dart standard]: http://www.ecma-international.org/publications/standards/Ecma-408.htm
[rfpp]: http://www.ecma-international.org/memento/TC52%20policy/Ecma%20Experimental%20TC52%20Royalty-Free%20Patent%20Policy.pdf
[form]: http://www.ecma-international.org/memento/TC52%20policy/Contribution%20form%20to%20TC52%20Royalty%20Free%20Task%20Group%20as%20a%20non-member.pdf
