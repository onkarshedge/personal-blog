---
title: "Scala Stackable Trait Pattern"
date: 2019-05-09T14:05:36+05:30
draft: false
categories: ["scala"]
---
Scala 2.12  
In Scala you can extend a class, trait, object with multiple traits. This pattern is known as mixins or stackable traits. 
 But before that let's understand how Scala solves the diamond inheritance problem. One word answer is by **`linearization`**. Consider the following traits present.
 
 ```
    trait A {
     println("trait A constructed")
     def foo(): Unit = println("A printed")
    }
 
    trait B extends A {
     println("trait B constructed")
     override def foo(): Unit = println("B printed")
    }
    
    trait C  extends A {
     println("trait C constructed")
     override def foo(): Unit = println("C printed")
    }
    
    trait D {
     println("trait D constructed")
     def foo(): Unit
    }
    
    class Alphabet extends B with C
    val alphabet = new Alphabet
    alphabet.foo()
    
```
The output of the following code will be
```
trait A constructed
trait B constructed
trait C constructed
alphabet: Alphabet = Alphabet@e1c45a
C printed
```
As we can see the foo method call was resolved to C.foo() The linearization order is the reverse of the construction order. Construction order can be seen from the output. Hence,
Linearization order = C -> B -> A. How is the construction order resolved ?  
From left to right traits are constructed, but for the trait to be constructed all of its base traits must be constructed first. If a trait is already constructed,
it is not reconstructed again. Hence first for B, A was constructed then B and when we came to C, C's parent A was already constructed, so only C was left.

However, if B and C both the traits didn't extend A, then `class Alphabet extends B with C` would throw a compile time error
```
class Alphabet inherits conflicting members:
 method foo in trait B of type => Unit  and
 method foo in trait C of type => Unit
 (Note: this can be resolved by declaring an override in class Alphabet.)
 class Alphabet extends B with C
```

**`super.foo()`**. The above example did not have any super calls. Let's see what happens if there is a super.foo() call.
```
trait A {
  println("trait A constructed")
  def foo(): Unit = {
    println("A printed")
  }
}

trait B  extends A {
  println("trait B constructed")
  override def foo(): Unit = {
    println("B printed")
    super.foo()
  }
}

trait C extends A {
  println("trait C constructed")
  override def foo(): Unit = {
    println("C printed")
    super.foo()
  }
}

trait DBase {
  println("trait DBase constructed")
  def foo(): Unit = println("Dbase printed")
}

trait D extends DBase {
  println("trait D constructed")
  override def foo(): Unit = {
    println("D printed")
    super.foo()
  }
}

class Alphabet extends B with A with C with D {
  override def foo():Unit = {
    println("alphabet")
    super.foo()
  }
}

val alphabet = new Alphabet
alphabet.foo()

```
The construction order will be A -> B -> C -> DBase -> D. Hence the linearization order will be the reverse, D -> DBase -> C -> B -> A.
Thus, the super call is dynamically resolved according to the linearization order, D's super.foo will call DBase's foo but since DBase's foo does not have a super.foo it cannot call C's foo.
The output of above code will be.

```
trait A constructed
trait B constructed
trait C constructed
trait DBase constructed
trait D constructed
alphabet
D printed
Dbase printed
```

But if the class Alphabet is implemented as such
```
class Alphabet extends DBase with B with A with C with D {
  override def foo():Unit = {
    println("alphabet")
    super.foo()
  }
```
The linearization order will be D ->  C -> B -> A -> DBase (reverse of construction order). Now D's super corresponds to C and not DBase.
A does not have a super call, hence DBase foo is never called. The output will be 
```
trait DBase constructed
trait A constructed
trait B constructed
trait C constructed
trait D constructed
alphabet: Alphabet = Alphabet@61e4c2f6
alphabet
D printed
C printed
B printed
A printed
```

I will talk more about the application of this stackable trait pattern in [part-2](/posts/scala-stackable-trait-pattern-2)

**References**  
1] https://stackoverflow.com/questions/34242536/linearization-order-in-scala  


