Scala Type Inference
====================

Scala's type inference can feel restrictive at times, especially when coming from languages like C# and Haskell where 
types *always* seem to be resolvable. The switch is naturally frustrating.

However, there are genuine reasons why Scala cannot resolve certain types - and the answer is typically due to some other Scala feature which forbids type inference from working in such a way. The likelihood is that Scala type inference won't improve because it *can't 'improve'* ([read comments here][scala-type-inference-blog]).

So, it's best to understand how to deal with it :)

Nuances of Scala
----------------

Scala uses left-to-right type inference: information flows across the parameter *lists* (*not* the individual parameters),
through the method body, and out to the result. This contrasts bidirectional or full type inference systems such as
Hindley-Milner (used in Haskell) which may appear more intuitive and less restrictive to users.

### Type parameters of type parameters cannot be inferred

Put another way: Scala type inference only *sees* types specified in the *parameter list* (not to be confused with *type parameter list*).

In Scala, if you define the following function:

    def head[L <: List[A], A](list: L): A

Type information will flow from parameters at the call-site to provide `L`. However, because the call-site only sees `L` 
in the *parameter list*  it will actually leave `A` as `Nothing`. This can be confusing since some type systems will 
be able to extract the types. *(Unfortunately I don't know the exact term for this behaviour.)*

The above example can be re-worked by ensuring the type we actually care about (type `A`) is visible in the parameter
list, allowing it to be inferred at the call-site:

    def head[A](list: List[A]): A
    
If there is a genuine need for both `L` and `A`, we would have to ensure both appear in the parameter list:

    def head[A, L[X] <: List[X]](list: L[A]): A
    
### Previous parameters are not used to infer future parameters

Type information only flows across parameter *lists*, not *parameters*. In other words: Scala does not use previous 
parameters to infer future parameter types, but does use previous *parameter lists*. This is important to understand 
in cases where the same type is used multiple times in a parameter list, but the call-site is only able to provide type 
information for one parameter:

    def getN[A](list: List[A], List[A] => A): A
    getN(List(1, 2, 3), l => l.head) // Compile error: 2nd parameter cannot provide type information for `A`.

This is called 'per-list inference' and contrasts 'per-parameter inference'. Both have their advantages and disadvantages, as outlined [here][scala-vs-csharp]. One such advantage is paraphrased below:

>   For example, the following will infer `A` as the upper-bound of the two parameters, if it exists:
>
>       def foo[A](x : A, y : A)
>
>   With per-parameter inference, you'd have to write:
>
>       def foo[A, B >: A, C >: A](x : B, y : C)

Applied to the previous example, we can force the inference engine to infer a type from a single parameter by isolating 
that parameter into its own parameter list - i.e. by currying the function:

    def getN[A](list: List[A])(List[A] => A): A

    getN(List(1, 2, 3)(l => l.head) // Compiles.

[scala-type-inference-blog]: http://pchiusano.blogspot.co.uk/2011/05/making-most-of-scalas-extremely-limited.html
[scala-vs-csharp]: http://www.scala-lang.org/old/node/7083
