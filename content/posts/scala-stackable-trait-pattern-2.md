---
title: "Scala Stackable Trait Pattern - 2"
date: 2019-05-13T14:27:58+05:30
draft: false
categories: ["scala"]
---
Scala 2.12  
In the last [post](/posts/scala-stackable-trait-pattern), I talked about linearization and how the super call is resolved. In this post, I will talk more about the application.

``` scala
import scala.collection.mutable.ArrayBuffer

trait Queue {
  def put(i: Int)

  def get(): Int
}

class ArrayQueue extends Queue {
  private val q = ArrayBuffer[Int]()
  override def put(i: Int): Unit = {
    q += i
  }

  override def get(): Int = q.remove(0)
}

trait Doubling extends Queue {
  abstract override def put(i: Int): Unit = {
    super.put(i * 2)
  }
}

trait Incrementing extends Queue {
  abstract override def put(i: Int): Unit = {
    super.put(i + 1)
  }
}

trait Filtering extends Queue {
  abstract override def put(i: Int): Unit = {
    if (i >= 0) {
      super.put(i)
    }
  }
}

//eg 1
class DoublingArrayQueue extends ArrayQueue with Doubling
val dq = new DoublingArrayQueue()
dq.put(5)
dq.get()

prints 10
//**************************

// eg 2
val aq = new ArrayQueue() with Incrementing with Doubling with Filtering
aq.put(5)
aq.get()

//prints 11
```
Stackable modifications to the classes. This is also known as mixins, we can mix any of the three modifications and obtain a new class.
There are in total 16 possibilities. (16 = 3P0 + 3P1 + 3P2 + 3P3 because order matters).
We can modify the behaviour of the existing class without making changes to it or creating a new class.
This combination of modifiers is only allowed for members of traits, not classes,
and it means that the trait must be mixed into some class that has a concrete definition of the method in question.

**Decorator Pattern**  
Decorator design pattern allow us to add functionality to an object (not the class) at runtime,
and we can apply this customized functionality to an individual object based on our requirement and choice.
Decorator design patterns are most often used for applying single responsibility principles since we divide the functionality into classes with unique areas of concern.

This pattern is similar in structure to the decorator pattern, except it involves decoration for the purpose of class composition instead of object composition.
Stackable traits decorate the core traits at compile time, similar to the way decorator objects modify core objects at run time in the decorator pattern.

            Base
              |
           ___|___           
          |       |
          |       |
        Core    Stackable

The core traits (or classes) implement the abstract methods defined in the base trait, and provide basic, core functionality.
Each stackable overrides one or more of the abstract methods defined in the base trait, using Scala's abstract override modifiers,
and provides some behavior and at some point invokes the super implementation of the same method.
In this manner, the stackables modify the behavior of whatever core they are mixed into

Following is the implementation of same problem statement in Java 8

``` scala
import java.util.LinkedList;

interface Queue {
    Integer get();

    void put(Integer value);
}

class BasicQueueImpl implements Queue {

    LinkedList<Integer> ll = new LinkedList<>();

    @Override
    public Integer get() {
        return ll.getFirst();
    }

    @Override
    public void put(Integer i) {
        ll.add(i);
    }
}


abstract class QueueWrapper implements Queue {
    protected Queue queue;

    void Queue(Queue queue) {
        this.queue = queue;
    }
}


class IncrementingQueue extends QueueWrapper {
    private Queue queue;

    public IncrementingQueue(Queue queue) {
        this.queue = queue;
    }

    @Override
    public Integer get() {
        return queue.get();
    }

    @Override
    public void put(Integer value) {
        queue.put(value + 1);
    }
}

class DoublingQueue extends QueueWrapper {
    private Queue queue;

    public DoublingQueue(Queue queue) {
        this.queue = queue;
    }

    @Override
    public Integer get() {
        return queue.get();
    }

    @Override
    public void put(Integer value) {
        queue.put(value * 2);
    }
}

class FilteringQueue extends QueueWrapper {
    private Queue queue;

    public FilteringQueue(Queue queue) {
        this.queue = queue;
    }

    @Override
    public Integer get() {
        return queue.get();
    }

    @Override
    public void put(Integer value) {
        if(value >= 0) queue.put(value * 2);
    }
}

public class Main {
    public static void main(String[] args) {
        DoublingQueue doublingIncrementingQueue = new DoublingQueue(new IncrementingQueue(new BasicQueueImpl()));
        doublingIncrementingQueue.put(5);
        System.out.println(doublingIncrementingQueue.get());
    }
}
// prints 11
```

**References**  
1] https://www.artima.com/scalazine/articles/stackable_trait_pattern.html  
2] https://dzone.com/articles/decorator-design-pattern-in-java
 
