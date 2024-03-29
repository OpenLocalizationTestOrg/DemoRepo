# C# Language Design Notes for Oct 7, 2013

## Agenda
We looked at a couple of feature ideas that either came up recently or deserved a second hearing.
1.	Invariant meaning of names <_scrap the rule_>
2.	Type testing expression <_can’t decide on good syntax_>
3.	Local functions <_not enough scenarios_>
4.	nameof operator <_yes_>

## Invariant meaning of names
C# has a somewhat unique and obscure rule called “invariant meaning in blocks” (documented in section 7.6.2.1 of the language specification) which stipulates that if a simple name is used to mean one thing, then nowhere in the immediately enclosing block can the same simple name be used to mean something else.

The idea is to reduce confusion, make cut & paste refactoring a little more safe, and so on.

It is really hard to get data on who has been saved from a mistake by this rule. On the other hand, everyone on the design team has experience being limited by it in scenarios that seemed perfectly legit.

The rule has proven to be surprisingly expensive to implement and uphold incrementally in Roslyn. This has to do with the fact that it cannot be tied to a declaration site: it is a rule about use sites only, and information must therefore be tracked per use – only to establish in the 99.9% case that no, the rule wasn’t violated with this keystroke either.

### Conclusion
The invariant meaning rule is well intentioned, but causes significant nuisance for what seems to be very little benefit. It is time to let it go.

## Type testing expressions
With declaration expressions you can now test the type of a value and assign it to a fresh variable under the more specialized type, all in one expression. For reference types and nullable value types:
``` c#
if ((var s = e as string) != null) { … s … } // inline type test
```
For non-nullable value types it is a little more convoluted, but doable:
``` c#
if ((var i = e as int?) != null) { … i.Value … } // inline type test
```
One can imagine a slightly nicer syntax using a `TryConvert` method:
``` c#
if (MyHelpers.TryConvert(e, out string s)) { … s … }
```
The signature of the `TryConvert` method would be something like
``` c#
public static bool TryConvert<TSource, TResult>(TSource src, out TResult res);
```
The problem is that you cannot actually implement `TryConvert` efficiently: It needs different logic depending on whether `TResult` is a non-nullable value type or a nullable type. But you cannot overload a method on constraints alone, so you need two methods with different names (or in different classes).

This leads to the idea of having dedicated syntax for type testing: a syntax that will a) take an expression, a target type and a fresh variable name, b) return a boolean for whether the test succeeds, c) introduce a fresh variable of the target type, and d) assign the converted value to the fresh variable if possible, or the default value otherwise.

What should that syntax be? A previous proposal was an augmented version of the “is” operator, allowing an optional variable name to be tagged onto the type:
``` c#
if (e is string s) { … s … } // augmented is operator
```
Opinions on this syntax differ rather wildly. While we agree that some mix of “is” and “as” keywords is probably the way to go, no proposal seems appealing to everyone involved. Here are a few:
``` c#
e is T x
T x is e
T e as x
```
(A few sillier proposals were made:
``` c#
e x is T
T e x as
```
But this doesn’t feel like the right time to put Easter eggs in the language.)

### Conclusion
Probably 90% of cases are with reference (or nullable) types, where the declaration-expression approach is not too horrible. As long as we cannot agree on a killer syntax, we are fine with not doing anything.

## Local functions
When we looked at local functions on Apr 15, we lumped them together with local class declarations, and dismissed them as a package. We may have given them somewhat short shrift, and as we have had more calls for them we want to make sure we do the right thing with local functions in their own right.

A certain class of scenarios is where you need to declare a helper function for a function body, but no other function needs it. Why would you need to declare a helper function? Here are a few scenarios:

* Task-returning functions may be fast-path optimized and not implemented as async functions: instead they delegate to an async function only when they cannot take the fast path.
* Iterators cannot do eager argument validation so they are almost always wrapped in a non-iterator function which does validation and delegates to a private iterator.
* Exception filters can only contain expressions – if they need to execute statements, they need to do it in a helper function.

Allowing these helper functions to be declared inside the enclosing method instead of as private siblings would not only avoid pollution of the class’ namespace, but would also allow them to capture type parameters, parameters and locals from the enclosing method. Instead of writing
``` c#
public static IEnumerable<T> Filter<T>(IEnumerable<T> s, Func<T, bool> p)
{
    if (s == null) throw new ArgumentNullException("s");
    if (p == null) throw new ArgumentNullException("p");
    return FilterImpl<T>(s, p);
}
private static IEnumerable<T> FilterImpl<T>(IEnumerable<T> s, Func<T, bool> p)
{
    foreach (var e in s)
        if (p(e)) yield return e;
}
```
You could just write this:
``` c#
public static IEnumerable<T> Filter<T>(IEnumerable<T> s, Func<T, bool> p)
{
    if (s == null) throw new ArgumentNullException("s");
    if (p == null) throw new ArgumentNullException("p");
    IEnumerable<T> Impl() // Doesn’t need unique name, type params or params
    {
        foreach (var e in s)          // s is in scope
            if (p(e)) yield return e; // p is in scope
    }
    return Impl<T>(s, p);
}
```
The underlying mechanism would be exactly the same as for lambdas: we would generate a display class with a method on it. The only difference is that we would not take a delegate to that method.

While it is nicer, though, it is reasonable to ask if it has that much over private sibling methods. Also, those scenarios probably aren’t super common.

### Inferred types for lambdas
This did bring up the discussion about possibly inferring a type for lambda expressions. One of the reasons they are so unfit for use as local functions is that you have to write out their delegate type. This is particularly annoying if you want to immediately invoke the function:
``` c#
((Func<int,int>)(x => x*x))(3); // What??!?
```
VB infers a type for lambdas, but a fresh one every time. This is ok only because VB also has more lax conversion rules between delegate types, and the result, if you are not careful, is a chain of costly delegate allocations.

One option would be to infer a type for lambdas only when there happens to be a suitable `Func<…>` or `Action<…>` type in scope. This would tie the compiler to the pattern used by the BCL, but not to the BCL itself. It would allow the BCL to add more (longer) overloads in the future, and it would allow others to add different overloads, e.g. with ref and out parameters.

### Conclusion
At the end of the day we are not ready to add anything here. No local functions and no type inference for lambdas.

## The nameof operator
The `nameof` operator is another feature that was summarily dismissed as a tag-along to a bigger, more unwieldy feature: the legendary `infoof` operator.
``` c#
PropertyInfo info = infoof(Point.X);
string name = nameof(Point.X); // "X"
```
While the `infoof` operator returns reflection information for a given language element, `nameof` just returns its name as a string. While this seems quite similar at the surface, there is actually quite a big difference when you think about it more deeply. So let’s do that!

First of all, let’s just observe that when people ask for the `infoof` operator, they are often really just looking to get the name. Common scenarios include throwing `ArgumentException`s and `ArgumentNullException`s (where the parameter name must be supplied to the exception’s constructor), as well as firing `Notify` events when properties change, through the `INotifyPropertyChanged` interface.

The reason people don’t want to just put strings in their source code is that it is not refactoring safe. If you change the name, VS will help you update all the references – but it will not know to update the string literal. If the string is instead provided by the compiler or the reflection layer, then it will change automatically when it’s supposed to.

At a time when we are trying to minimize the use of reflection, it seems wrong to add a feature, `infoof`, that would contribute significantly to that use. `nameof` would not suffer from that problem, as the compiler would just replace it with a string in the generated IL.  It is purely a design time feature.

There are deeper problems with `infoof`, pertaining to how the language element in question is uniquely identified. Let’s say you want to get the `MethodInfo` for an overload of a method `M`. How do you designate which overload you are talking about?
``` c#
void M(int x);
void M(string s);
var info = infoof(M…); // Which M? How do you say it?
```
We’d need to create a whole new language for talking about specific overloads, not to mention members that don’t have names, such as user defined operators and conversions, etc.

Again, though, `nameof` does not seem to have that problem. It is only for named entities (why else would you want the name), and it only gives you the name, so you don’t need to choose between overloads, which all have the same name!

All in all, therefore, it seems that the concerns we have about `infoof` do not apply to `nameof`, whereas much (most?) of its value is retained. How exactly would a `nameof` operator work?

### Syntax
We would want the operator to be syntactically analogous to the existing `typeof(…)`. This means we would allow you to say `nameof(something)` in a way that at least sometimes would parse as an invocation of a method called `nameof`. This is ambiguous with existing code only to the extent that such a call binds. We would therefore take a page out of `var`’s book and give priority to the unlikely case that this binds as a method call, and fall back to `nameof` operator semantics otherwise. It would probably be ok for the IDE to always highlight `nameof` as a keyword.

The operand to `nameof` has to be “something that has a name”. In the spirit of the `typeof` operator we will limit it to a static path through names of namespaces, types and members. Type arguments would never need to be provided, but in every name but the last you may need to provide “generic dimension specifiers” of the shape `<,,,>` to disambiguate generic types overloaded on arity.

The grammar would be something like the following:
> _nameof-expression:_
> * `nameof`   `(`   _identifier_   `)`
> * `nameof`   `(`   _identifier_   `::`   _identifier_   `)`
> * `nameof`   `(`   _unbound-type-name_   `.`   _identifier_   `)`

Where _unbound-type-name_ is taken from the definition of `typeof`. Note that it always ends with an identifier. Even if that identifier designates a generic type, the type parameters (or dimension specifiers) are not and cannot be given. 

### Semantics
It is an error to specify an operand to `nameof(…)` that doesn’t “mean” anything. It can designate a namespace, a type, a member, a parameter or a local, as long as it is something that is in fact declared.

The result of the `nameof` expression would always be a string containing the final identifier of the operand. There are probably scenarios where it would be desirable to have a “fully qualified name”, but that would have to be constructed from the parts. After all, what would be the syntax used for that? C# syntax? What about VB consumers, or doc comments, or comparisons with strings provided by reflection? Better to leave those decisions outside of the language.

### Conclusion
We like the `nameof` operator and see very few problems with it. It seems easy to spec and implement, and it would address a long-standing customer request. With a bit of luck, we would never (or at least rarely) hear requests for `infoof` again.

While not the highest priority feature, we would certainly like to do this.


