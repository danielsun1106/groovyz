Groovyz
=======

POC of Type Class for Groovy.

How to Run
==========

```
  groovy sample.groovy
```

Groovy 2.4.x or higher is recommended.

Providing type class library
===============================

Type classes
-------------------------

* interface Monoid&lt;T> {
* interface Functor&lt;F> {
* interface Applicative&lt;A> extends Functor&lt;A> {
* interface Monad&lt;M> extends Applicative&lt;M> {
* interface Show&lt;T> {
* trait Eq&lt;T> { // One of eq or neq, or both should be overridden.
* trait Ord&lt;T> extends Eq&lt;T> {

Sample Type Class Instances
-----------------------------------

* class OptionalMonoid&lt;T> implements Monoid&lt;Optional&lt;T>> {
* class ListFunctor implements Functor&lt;List> {
* class OptionalFunctor implements Functor&lt;Optional> {
* class ListApplicative extends ListFunctor implements Applicative&lt;List> {
* class ListMonad extends ListApplicative implements Monad&lt;List> {
* class IntShow implements Show&lt;Integer> {
* class StringShow implements Show&lt;String> {
* class ListShow&lt;T> implements Show&lt;List&lt;T>> {
* class IntEq implements Eq&lt;Integer> {
* class IntOrd implements Ord&lt;Integer> {


Why I write this?
==================

Type class is a language construct which is available in languages Haskell, Scala and Rust and so on.
As my personal impression, it was difficult to understand.
So I'd like to explain through implementing it on super self-customizable language, Groovy.

Note: This code implement only the core concept of Type class, and details should be different in real other languages.

What is Type Class?
=====================

Type class enables us to restrict Generic(polymorphic) type parameters to what operations can be done on it.
It also give the guarantee what operation can be done when using the type parameters in function which takes the type parameter as a type of its parameter.

From another point of view, type class provides a way to restrict and guarantee type parameter **without using class inheritance**.

For example, following code using method generics in Groovy,

```
static<T> void runSomething(T a) {
   a.run()
}
```

When we want to restrict the type parameter T, we can specify implement Runnable

```
static<T implements Runnable> void runSomething(T a) {
   a.run()
}
```

By using class inheritance, we can restrict and guarantee that method can be called to parameter T a.
This is not bad, but the motivation of introduce type class is avoiding class inheritance.

Why we want to avoid class inheritance for restriction on generics parameter?
===================================================================================

[TBD]

4 steps of applying type class
=================================================

We'd like to explain mechanism of our Groovy type class with following 4 steps:

* (1) Define a type class
* (2) Instantiate that type class
* (3) Define a function that takes generic type parameters and restrict the type parameter with type class.
* (4) Call the function with type class instance that selected automatically from the scope.


(1) Define a type class
-----------------------------

Define Monoid&lt;T> type class as an interface.

```
@TypeChecked
interface Monoid<T> {
    def T mappend(T t1, T t2);
    def T mempty();
}
```

Use 'def T' instead of just 'T' because of syntax restriction.

You can see this is a normal Interface. You can inherit from this to define another type class, for example Monad extends Functor.
Probably you might be confused because we use class inheritance here. It doesn't be avoiding class inheritance.

But in here, we only use class for just a place to put method and never use/define instance variable. If we define full AST transformation for type class, methods can be static and search method along with inheritance relation in some way (@InheritConstructor AST Transformation already do it).
Using inheritance and instance method here is only for convenience. Exactly what we do here is only define a set of operations on generics type T. Reference to instance through 'this' is needles here.

In Haskell type class functions are implemented as static functions, and in Scala those are method of singleton (object). So in both of case, `this` reference is meaningless.

To sum up,

* Define set of operations as methods of class. This class is used as restriction of Generics parameter (=type class). This class and derived class never have/use instance variable.
* Defined Methods can call each other.
* Method body can be defined if we needed. You can use trait to define method body.

(2) Instantiate that type class
-----------------------------

Instantiation of type class is, to declare that the type satisfies a type class restrictions, and fill up abstract methods body.

Following code declares instantiate String as a Monoid type class.

```
def instance_Monoid_java$lang$String_ = new Monoid<String>() {
    @Override String mappend(String i1, String i2) {
        i1+i2
    }
    @Override String mempty() {
        ""
    }
}
```

The name of the variable which holds a type class instance, in this case `instance_Monoid_java$lang$String_`, should be follow on some naming convention. Groovy's scope handling resolves type class instance. See step (4).

Until here we use only plain normal Groovy code. There is nothing-special thing.

(3) Define a function, which takes generic type parameters and restrict the type parameter with type class.
-----------------------------

Now, let's define generic function which takes a generic parameter on which the type class restriction applied:

```
@TypeChecked
class Functions {
    static <T> T mappend(T t1, T t2, Monoid<T> dict=Parameter.IMPLICIT) { dict.mappend(t1,t2) }
    static <T> T mempty(Monoid<T> dict=Parameter.IMPLICIT) { dict.mempty() }
}
```

The parameter `Monoid dict=Parameter.IMPLICIT` is a declaration of the type class restrictions in this implementation, and the meaning in here is:
T of returned value and the parameter t1, t2 should be satisfied Monoid type class restriction.
You might think it's verbose that call to `mappend` calls same name method `mappend`.
In Haskell, functions on type class instance are already static, but this implementation uses functions as instance method, so this step means to expose to static space.

Passed parameter `dict` is used as namespace of functions. You can call any method on the namespace. You probably want to use Groovy's `with` phrase here like:

```
    .... {
       dict.with { mappend(t1,t2) }
    }
```


(4) Call the function with type class instance, which selected automatically from the scope.
-----------------------------

Finally, call defined generic functions using Type Class.

```
import static Functions.*
 :
@TypeChecked(extensions='ImplicitParamTransformer.groovy')
  :
        String s = mempty()
        assert s == ""
        assert mappend("a","b") == "ab"
```

This code can be run under STC(Static Type Checking).

Arguments for parameter Monoid type instance is not to appeal but it is choosen and supplied implicitly by `ImplicitParamTransformar` custom type checker(type checking extension). `ImplicitParamTramsformer` does followings:

* If a static method call uses generics,
  * Check the definition of the static method.
  * And if the method have a parameter which default initial value is `Paraemter.IMPLICIT`
  * Resolve generics paramter types and return type.
     * Do simple and limited target type inference
  * Apply resolved generics types to actual type of parameter which takes `Parameter.IMPLICIT` initial value.
  * Determine actual type of parameter, which takes `Parameter.IMPLICIT`
  * Encode that type to variable name with following rule:
     * Add prefix `instance_`
     * replace &lt; and &gt; to _
     * replace . in FQCN to `$`
  * Modify AST to provide a variable, which has the type-encoded name, as the parameter that takes `Parameter.IMPLICIT` initial value

For example, when calling `mappend("a", "b")`, correspond static method definition which have the name `mappend` is:

```
    static <T> T mappend(T t1, T t2, Monoid<T> dict=Parameter.IMPLICIT) { dict.mappend(t1,t2) }
```

So infer the generics type `T`(for return type and parameter type of t1,t2) is `String`, and `Monoid<T>` realize it to be `Monoid<java.lang.String>`, so encoded variable name sould be `instance_Monoid_java$lang$String_`. This call is finally converted to:

```
mappend("a","b",instance_Monoid_java$lang$String_ )
```

Even if the function takes no parameter, we can infer by target type inference, so `String s=empty()` would be converted to:

```
String s = mempty(instance_Monoid_java$lang$String_ )
```

In Type class of Scala, it searches by type and ignore variable name. But this implementation doesn't do it. Instead leave it to Groovy's variable scope rule.

Type constructor as type parameter(Higher-kind Generics)
----------------------------------------------------------------------------

Groovy generics is a different from Java. In Java generics, generics type parameter type can't be a type parameter which takes type parameter.
In the other words, type constructor can't be pass as type parameter. But Groovy can do it.
For example, `List` of `List<T>`, or `Optional` of `Optional<T>` can't be pass as generics parameter in Java but Groovy can.

In this type class sample library, we use this feature like following:

```
@TypeChecked
interface Applicative<A> extends Functor<A> {
    public <T> A<T> pure(T t)
    public <T,R> A<R> ap(A<Function<T,R>> func, A<T> a) // <*> :: f (a -> b) -> f a -> f b 
}
```

This type of generics is especially vital to represent highly abstract type like `Monad`, so on.

TODOs
=========

* Type Class instance name auto generation.
* Get generics parameter / return value information of static function other than import static declaration (it's fatal for modularization!)


Summary
===============

Groovy's custom type checker is very powerful technology not only for type checking but also implement a kind of language feature like Type Class.
But it might be abuse of for 'type checker'. I hope extended custom type checker or new feature which can be use easily like custom type checker, which can be specified various compile phase and can handle more AST node match events.

Enjoy!

