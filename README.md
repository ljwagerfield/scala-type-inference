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

Put another way: Scala type inference only sees types specified in the *parameter list* (not to be confused with *type parameter list*).

In Scala, if you define the following function:

    def head[L <: List[A], A](list: L): A

Type information will flow from parameters at the call-site to provide `L`. However, because the call-site only sees `L` 
in the *parameter list*  it will actually leave `A` as `Nothing`. This can be confusing since some type systems will 
be able to extract the types. *(I don't know the exact term for this behaviour.)*

The above example can be re-worked by ensuring the type we actually care about (type `A`) is visible in the parameter
list, allowing it to be inferred at the call-site:

    def head[A](list: List[A]): A
    
If there is a genuine need for both `L` and `A`, we would have to ensure both appear in the parameter list:

    def head[A, L[X] <: List[X]](list: L[A]): A
    
### Type members of type parameters cannot be inferred

The following example will not compile:

```
object Example {
  trait AType {
    type B
  }
  
  case object Foo extends AType { type B = Int }
  
  def foo[A <: AType](a: A, b: A#B) = ???

  foo(Foo, 5) // Does not compile
}
```

This is because `A#B` is in the same parameter list as where `A` is attempting to be inferred -- which prevents Scala from inferring `A`.

The problem is that Scala attempts to infer type parameters (e.g. `A`) by inspecting all the usages of `A` within that parameter list. So using `a: A, b: A#B` in the same parameter list prevents Scala from inferring `A` because it's also trying to resolve the type of `A` by inspecting `A#B`, which it can't make sense of. Since Scala stops attempting to infer types in subsequent lists that it's already resolved, the solution is to split the types into multiple parameter lists, making the first param list expose an easy-to-resolve `A`, and then using the already-resolved `A` in subsequent lists to reference type members:

```
object Example {
  ... same as before ...
  
  def foo[A <: AType](a: A)(b: A#B) = ??? // Note: two parameter lists.

  foo(Foo)(5) // Does not compile
}
```
    
### Avoiding f-bounded polymorphism

F-bounded polymorphism should be avoided in most cases - especially in Scala when considering the above behaviour. To understand how to avoid it, we first need to know what makes us require it:

>   F-bounded polymorphism occurs when a type expects *important interface changes* to be introduced in derived types.

This can be avoided by *composing* the expected areas of change instead of attempting to support them as a derivative of the current type. This actually comes back to Gang of Four design patterns:

>   Favor 'object composition' over 'class inheritance'
>   --  (Gang of Four 1995:20)

For example:

    trait Vehicle[V <: Vehicle[V, W], W] {
        def replaceWheels(wheels: W): V
    }

becomes:

    trait Vehicle[T, W] {
        val vehicleDetails: T
        def replaceWheels(wheels: W): Vehicle[T, W]
    }
    
Here, the 'expected change' is the vehicle details (e.g `Bike`, `Car`, `Lorry`). The previous example assumed this would be added through inheritance, requiring an f-bounded type that made inference of `W` impossible for any function using `Vehicle`. The new method, which uses composition, does not exhibit this problem.
    
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
