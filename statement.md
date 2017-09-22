# Generic Types and Variance in Kotlin compared to Java


:kotlin: http://kotlinlang.org[Kotlin]
:mygh: https://github.com/s1monw1/kotlin_vertx_example/blob/master/README.md[GitHub]
:liskov: https://en.wikipedia.org/wiki/Liskov_substitution_principle

## Basics - What is Variance?

Many programming languages support the concept of _subtyping_, which allows us to implement hierarchies like "A `Cat` **IS-A**n ``Animal``".
In Java we can either use the `extends` keyword in order to change/expand behaviour of an existing class (inheritance) or use `implements` to provide implementations for an interface.
According to {liskov}[Liskov's substitution principle], every instance of a class `A` can be substituted by instances of its subtype `B`.

The word `variance`, often referred to in mathematics as well, is used to describe how subtyping 
in complex aspects like method return types, type declarations or arrays relates to the direction of inheritance of the involved classes. 
There are three terms we need to take into account: _Covariance_, _Contravariance_ and _Invariance_.


### Variance in Practice (Java)

.Covariance
In Java an overriding method needs to be *covariant* in its return types, i.e. the return types of the overridden and the overiding method must be in line with the direction of the inheritance tree of the involved classes. 
A method `treat():Animal` of class `AnimalDoctor` can 
be overriden with `treat():Cat` in class `CatDoctor`, which extends `AnimalDoctor`.
Another example of covariance would be type conversion, shown here:

```kotlin
public class Animal{}
public class Cat extends Animal{}

(Animal) new Cat() //works fine
(Cat) new Animal() //will not work
```

Subclasses can be cast _up_ the inheritance tree, while downcasting will cause an error here. This is also the case if we take a look at variable declarations. 
It isn't a problem to assign an instance of `Cat` to a variable of type `Animal`, whereas doing the opposite will cause failure.

.Contravariance
*Contravariance* on the other hand describes the exact opposite. In Java this concept only reveals itself when we work with generics, which I'm going to describe in depth later. 
Just to make it clear, we can image another programming language that allows contravariant method arguments in overriding methods footnote:[In Java such a method would be treated as _overloaded_].
Let's say we have a class `ClassB` which extends another class `ClassA` and overrides a method by changing the original parameter type `T'` to its supertype `T`.

`ClassA::method(t:T')`

`ClassB::method(t:T)`

You can see that the type hierarchy of method parameter `t` is contrary to the hierarchy of the surrounding classes. Up versus down the tree, `method` is contravariant in its parameter type.

.Invariance
Last but not least, the easiest one: *Invariance*. It can be observed when we think of overriding methods in Java again like we've just seen in the example before. An overriding method must _accept_ just the same parameters as the overridden method. 
This means we speak of invariance if the types of an aspect like "method parameters" do not differ in super- and subtype.

### Variance of collection types in Java
Another aspect we want to consider is arrays and other kinds of generic collections. 

#### Arrays

Arrays in Java are *covariant*
in their type, which means an array of ``String``s can be assigned to a variable of type "Array of ``Object``".

[[array_variance]]
[source, java]
.Array Variance
----
Object [] arr = new String [] {"hello", "world"};
----

Also, arrays are *covariant* in the types that they hold. This means you can add ``Integer``s, ``String``s or whatever kind of ``Object`` to an `Object []`.

[source, java]
----
Object [] arr = new Object [2];
arr[0] = 1;
arr[1] = "2";
----

This seams to be quite handy but can cause errors at runtime. Looking at example <<array_variance>> again: The variable is of type `Object []` but the referenced object is a `String []`.
What happens if we pass the variable to a method expecting an array of ``Object``s? This method might want to add an `Object` to the array, which seems legit because the paramter is expected to be of type `Object []`.
It will cause an `ArrayStoreException` at runtime, easily shown here:

[[runtimeerr_array]]
[source, java]
.Array Runtime Error
----
Object [] arr = new String [] {"hello", "world"};
arr[1] = new Object(); //will throw Exception; java.lang.ArrayStoreException: java.lang.Object
----

#### Generic Collections

As of Java 1.5 we can use _Generics_ in order to tell the compiler which elements are supposed to be stored in our collections (i.e. List, Map, Queue, Set).
Unlike arrays, generic collections are *invariant* in their parameterized type by default. This means you can't substitute a `List<Animal>` with a `List<Cat>`. It won't even compile.
As a result it is not possible to run into unexpected runtime errors like it is when working with covariant arrays. As a drawback, we are much more inflexible regarding subtyping of collections. 

Fortunately, the user can specify the variance of type parameters himself when using generics, which we call _use-site variance_.

.Covariant collections

The following code example shows how we declare a _covariant_ list of `Animal` and assign a list of ``Cat`` to it.

[source, java]
----
List<Cat> cats = new ArrayList<>();
List<? extends Animal> animals = cats;
----

Anyways, such a covariant list is still different to an array, because the covariance is encoded in its type parameter. We can only _read_ from the list, whereas _adding_ is prohibited. The list is said to be a *Producer* of ``Animals``.
The generic type `? extends Animal` footnote:[`?` is the "wildcard" character] only indicated that the list contains _any_ type with an upper bound of `Animal`, which could mean list of `Cat`, `Dog` or any other animal.
This approach turns the runtime error encountered in <<runtimeerr_array>> into a compile error.

[source, java]
----
List<Cat> cats = new ArrayList<>();
List<? extends Animal> animals = cats;
animals.add(new Cat()); //will not compile
----

.Contravariant collections

It is also possible to work with _contravariant_ collections. Such a list can be declared with the generic type parameter `? super Animal`, which means a lower bound of type `Animal`. 
Such a list may be of type `List<Animal>` itself or a list of any super type of `Animal`, even ``Object``.
Like with covariant lists, we do not know for sure which type the list really represents (indicated by the wildcard again). 
The difference is, we can not _read_ from a contravariant list, since it is unclear if we will get ``Animal``s or just plain ``Object``s. But now we
can _write_ to the list as we know that _at least_ ``Animal``s may be added. This allows us to safely add ``Cat``s as well as ``Dog``s. Such a list is said to be a *Consumer*.

[source, java]
----
List<Animal> animals = new ArrayList<>();
List<? super Animal> contravariantAnimals = animals;
contravariantAnimals.add(new Cat());
contravariantAnimals.add(new Dog());
Animal pet = contravariantAnimals.get(0); // will not compile
----

TIP: Joshua Bloch created a rule of thumb in his fantastic book "Effective Java": 
"Producer-``extends``, consumer-``super`` (*PECS*)"

## Variance of collections types in Kotlin

After we've seen what variance in general means and how Java makes use of these concepts, I'd like to get to this blog post's point. 
Kotlin is different to Java regardings generics and also arrays in a few ways and it might look odd to an experienced Java developer at first glance.

The first and maybe easiest difference is: Arrays in Kotlin are *invariant*. This means, as opposed to Java, it is _not possible_ to assign an `Array<String>` to a reference variable
of type `Array<Object>`. This ensures compile time safety and prevents runtime errors like you may encounter in Java with its covariant arrays.
But is there a way to safely work with subtyped arrays? Sure, there is - we'll look at it next.

### Declarion-site Variance
As we've seen, Java uses so called "wildcard types" to make generics variant, which is said to be "the most tricky part[s] of Java's type system" footnote:[see http://kotlinlang.org/docs/reference/generics.html#type-projections]. The whole thing is called "use-site variance".
Kotlin does not use these at all. Instead, in Kotlin we use _declarion-site_ variance. 
Let's recall the initial problem again: Let's imagine, we have a class `ReadableList<E>` with one simple producer method `get():T`.
Java prohibits to assign an instance of `ReadableList<String>` to a variable of type  `ReadableLis<Object>` because generic types are invariant by default.
To fix this, the user can change the variable type to `ReadableList<? extends Object>` and everthing works fine.
Kotlin approaches this problem in a different way. The type `T` can be marked as 'only produced' with the *out* keyword, so that the compiler instantly gets it: ``ReadableList`` is never gonna consume any `T`, which makes `T` covariant. 

[[kotlin_out]]
[source, kotlin]
.Kotlin `out`
----
abstract class ReadableList<out T> {
    abstract fun get(): T
}

fun workWithReadableList(strings: ReadableList<String>) {
    val objects: ReadableList<Any> = strings // This is OK, since T is an out-parameter
    // ...
}
----

As you can see in "<<kotlin_out>>" the type `T` is annotated as an `out` type via _declaration-site variance_ - it is also referred to as *variance annotation*.
The compiler does not prohibit us to use T as an _covariant_ type.
Of course there is also a complementary annotation to mark generic types as consumers, i.e. makes them _contravariant_: *in*.
This approach has also been used in _C#_ sucessfully for some years already.

TIP: The Kotlin rule to memorize: *Consumer in, Producer out* (CIPO)

### Use-site Variance, Type projections

Unfortunately, it is not always sufficient to have the opportunity of declaring a type parameter `T` as either *out* or *in*.
Just think of arrays for example. An array offers methods for adding and receiving objects, so it cannot be either co- or contravariant in its type parameter.
As an alternative Kotlin also allows use-site variance, which is very similar to Java using the already defined keywords *in* and *out*:

- `Array<in String>` corresponds to Java's `Array<? super String>`
- `Array<out String>` corresponds to Java's `Array<? extends Object>`

[source, kotlin]
.Type projection Example
----
fun copy(from: Array<out Any>, to: Array<Any>) {
 // ...
}
----

This example shows how `from` is declared as a consumer of its type and thus the method cannot do 'bad things' like adding to the `Array`. 
This concept is called *type projection* since the array is restricted in its methods: only those methods that return type parameter `T` may be called.

## Bottom line
With this post I wanted to provide same basic information on the quite complex aspects of _variance_ in the context of generics. Mostly, Java was used 
to demonstrate the concepts of co-, contra- and invariance, which are hard to understand in connection with Java's wildcard types.
I've shown how Kotlin tries to simplify the whole thing using different approaches (declaration-site variance) and more obvious keywords (`in`, `out`).

In my oppionion Kotlin really makes it better than Java and also eliminates another problem, which is covariant arrays. Allowing declaration-site variance simplifies a lot of client code 
where it's not necessary to use complex declarations like we're used to in Java. Also even if we have to fall back on use-site variance its a bit simpler.

I hope this makes sense to you :-)

This Kotlin template lets you get started quickly with a simple one-page playground.

```kotlin runnable
fun main(args: Array<String>) {
    println("Hello, World!")
}
```
