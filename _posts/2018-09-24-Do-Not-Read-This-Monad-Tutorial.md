![alt text](https://pbs.twimg.com/media/Db4szC7W0AAKvHh.jpg:large)

In an attempt to break the cycle, this monad tutorial isn't any better than the others, and honestly, you shouldn't bother reading it at all. Reading oodles of tutorials won't make you understand monads. Tutorials never made us understand how to write code in the first place, but breaking everything and building weird stuff did. So instead reading this tutorial... write your own. That feeling of terror knowing your beautiful brainchild will be subject to the gnashing of teeth that is the internet will motivate you try it yourself, to dig up all the details, break everything and answer all the questions your readers will inevitably have. I found inspiration in [Dan Piponi's](http://blog.sigfpe.com/2006/08/you-could-have-invented-monads-and.html) idea that I could have invented monads myself and [Brent Yorgey's](https://byorgey.wordpress.com/2009/01/12/abstraction-intuition-and-the-monad-tutorial-fallacy/) assertion that monads are not, in fact, burritos. 

And now I present to you, my brainchild.

## The Issue of Ugliness ##

In scala we really like stringing together our method calls like this. It's easy to read and easy to debug.
``` scala
List(1, 2, 3, 4, 5)
  .map(_ + 10)
  .filter(_ % 2 == 0)
  .take(2)
```

Instead of a regular list, the hypothetical app we're working on really needs a List where these functions return both the list result and a debug string from each of these functions.
``` scala
object DebugList {
  //variable argument syntax so it works like List.apply
  def apply[A](a: A*): DebugList[A] = DebugList(List(a: _*))
}

case class DebugList[A](l: List[A]) {
  def map[B](f: A => B): (DebugList[B], String) =
    (DebugList(l map f), "mapped")

  def filter(f: A => Boolean): (DebugList[A], String) =
    (DebugList(l filter f), "filtered")

  def take(n: Int): (DebugList[A], String) =
    (DebugList(l take n), s"took $n")
}
```

DebugList is used like this ...which is so ugly.
``` scala
val (dl1, str1) = DebugList(1, 2, 3, 4, 5).map(_ + 10)
val (dl2, str2) = dl1.filter(_ % 2 == 0)
val (dl3, str3) = dl2.take(2)
```
But this is what our app needs, so we're gonna try and make it work.

The problem is that we have to do all this clunky matching on the tuples. Plus if we want to see our debug string in order, we need to do something like ```s"$str1 $str2 $str3"``` to read, print, or write them. Since the tuple is the issue let's try putting the tuple in another class so we can write functions like flatmap to do the gross stuff for us.

### Exercise 1 
Fill in the definition of flatMap and map for this case class. Make sure the resulting debug object has a string with both the contents from this debug object and from the result of f.
``` scala
case class Debug[A](a: A, s: String) {
  def flatMap[B](f: A => Debug[B]): Debug[B] = ???

  def map[B](f: A => B): Debug[B] = ???
}
```

Next we'll need a constructor in the companion object that gives us a way to turn regular old A objects into Debug[A] objects.

### Exercise 2
Fill in the unit function:
``` scala
case object Debug {
  def apply[A](x: A): Debug[A] =
    unit(x)

  def unit[A](x: A): Debug[A] = ???
}
```

### Exercise 3
Now that we've refactored that tuple, we can make a BetterDebugList with functions that return our newly refactored Debug type instead of tuples. Fill in the body of these functions:
``` scala
object BetterDebugList {
  //variable argument syntax so it works like List.apply
  def apply[A](a: A*): BetterDebugList[A] = BetterDebugList(List(a: _*))
}

case class BetterDebugList[A](l: List[A]){
  def map[B](f: A => B): Debug[BetterDebugList[B]] = ???

  def filter(f: A => Boolean): Debug[BetterDebugList[A]] = ???

  def take(n: Int): Debug[BetterDebugList[A]] = ???
}
```

BetterDebugList can now be used like this:
``` scala
val debug = Debug(BetterDebugList(1, 2, 3, 4, 5))
  .flatMap(_.map(_ + 10))
  .flatMap(_.filter(_ % 2 == 0))
  .flatMap(_.take(2))
```

Whoa! that's looks pretty similar to how we originally used List. No more tuple matching!

In order to hide those explicit flatMap calls we can use for comprehensions because they're prettier but do exactly the same thing.
``` scala
val debug = for {
  w <- Debug(BetterDebugList(1, 2, 3, 4, 5))
  x <- w.map(_ + 10)
  y <- x.filter(_ % 2 == 0)
  z <- y.take(2)
} yield z
```

Now you can get that debug string out like this:
``` scala
println(debug.s)
```

## Surprise! You made a monad.
Just like many other functional programming tools, a monad takes legitmately useful code that might otherwise be very awkward to use and makes it feel more natural. Notice how we can use this same Debug class to make a debugable version of any other class we want. 

In Scala we use monads all the time because they are so natural. ```List``` and ```Option``` are both monads that we see in nearly every beginner Scala tutorial.

## Ok but what is a monad?
A monad has to have...  
1 - flatMap (sometimes called bind)  
2 - unit 
3 - follow the three monad laws  

In Scala monads, apply usually implements unit as well as handles more cases that don't count as unit.
``` scala
List(1)       //unit
List(1, 2, 3) //not unit. This doesn't match the signature A => M[A]
Option(5)     //unit
Some(5)       //not unit. This does not return an Option. It returns a Some.
```

In order to be a monad it has to follow the three monad laws too. These laws just make sure we can refactor our code in the way we expect and have predictable results.

### The Monad Laws
*f and g are functions  
m is an instance of a monad which is also called a "monadic action"*

1 - Right Identity  
``` scala
unit(z).flatMap(f) == f(z)
```

2 - Left Identity  
``` scala
m.flatMap(unit) == m
```

3 - Associativity  
``` scala
m.flatMap(f).flatMap(g) == m.flatMap(x => f(x).flatMap(g))
```

Let's look at examples of these law definitions using the Monad List. Imagine how weird using List would be if these statements were not always true:

1 - Right Identity  
``` scala
List(2).flatMap(x => List(x * 5)) == List(2 * 5)
```

2 - Left Identity  
``` scala
List(2).flatMap(List(_)) == List(2)
```

3 - Associativity  
``` scala
List(2).flatMap(w => List(w, w)).flatMap(y => List(y * 2)) == 
List(2).flatMap(x => List(x, x).flatMap(z => List(z * 2)))
```

## Break the Monad Laws

The FMCounter class counts how many times flatMap has been called on it. It looks like a monad, but it breaks some of the 3 laws. 

### Exercise 4
Find out which laws it breaks.
``` scala
case object FMCounter {
  def unit[A](a: A): FMCounter[A] =
    FMCounter(a, 0)

  def append(str: String, end: String): FMCounter[String] =
    unit(str + end)
}

case class FMCounter[A](a: A, counter: Int) {
  def flatMap[B](f: A => FMCounter[B]): FMCounter[B] = {
    val FMCounter(b, k) = f(a)
    FMCounter(b, counter + k + 1)
  }

  def map[B](f: A => B): FMCounter[B] =
    FMCounter(f(a), counter)
}
  ```

Because it breaks these laws, there are multiple ways to correctly write the same code that result in different flatMap counts. All that means is that it's probably not the solution we're looking for. But also that it's not a monad.

## Conclusion
If you were faced with a specific problem like stringing together functions that return tuples, you really might have invented Monads yourself. Monads are simply a tool to make otherwise clunky solutions feel more natural. We use Monads all the time already so it's worth understanding why they're so good at what they do.