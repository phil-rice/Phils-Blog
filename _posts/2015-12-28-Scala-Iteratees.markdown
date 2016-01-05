---
layout: post
title: "Scala Iteratees"
date:   2015-12-28 16:00:23 +0000
comments: True
categories:
- scala
- play
---

#Code
The code can be found at [Github](https://github.com/phil-rice/IterateesBlog).

#The problem that Iteratees are solving
I've been using iterators (not iteratees) what feels like forever. They have a number of issues, which mean that every now and then I have to use clunky things like 'break', or just 'roll my own'. One of the classical issues
is what I used to call [the N and a half loop](http://rosettacode.org/wiki/Loops/N_plus_one_half)  This is where you are looking through a list of things, often doing something as you go, until you find the thing you are looking for.

Very often the code is clunky. Iteratees are a good solution to this problem. They also address other issues too, but I'll introduce them with this problem. It's worth pointing out that the scala example 
in the Rosseta code link above didn't actually address this problem
{% highlight scala %} 
object BreakExample {
  import util.control.Breaks._
  def main(args: Array[String]): Unit = {
	 var list = List[Int]()
	    breakable {
	      for (i <- 1 to 10) {
	        list ::= i
	        if (i > 4) break // break out of the for loop
	      }
	    }
	    println(list)
	  }
}
{% endhighlight %}
As you can see from this example the code is iterating over a list (1,2,3,4,5,6,7,8,9,10) doing something with it (folding it into list). You can also see that the code is horrible. 

Perhaps doing this as a fold will help. The following keeps the 'folded variable' as a tuple of the list we are building up, and a 'state' which is 'shall I continue'. And the code is again horrible 
as everything is stuffed together, Without comments or an explaination, it's quite impenetrable. And yet the task is pretty simple.
{% highlight scala %} 
object FoldExample {
  import util.control.Breaks._
  def main(args: Array[String]): Unit = {
    val result = (1 to 10).foldLeft((List[Int](), true)) {
      case (result @ (acc, cont), i) =>
        if (cont) (i :: acc, i <= 4) else result
    }
    println(result._1)
  }
}
{% endhighlight %}
It's also not just collections we want to do this on. We want to be able to consume data from streams, or files, or .. .well anything! 

So it summarise, the problem we are solving is "writing elegant code that folds over 'something'  and allows us to break early"

#Good things to read
I enjoyed this [Iteratees for Normal Humans](http://mandubian.com/2012/08/27/understanding-play2-iteratees-for-normal-humans/) and plagarised it a little. The [Play documentation](https://www.playframework.com/documentation/2.4.4/Iteratees) was helpful as well

#Enumerators
Let's look at something that can produce data for us to fold over. 
The raw thing... the 'something' that we fold over is, is an Enumerator. This is a producer of data. It's backing object can be pretty much anything (a collection, a file, a stream, even an algorithm)
 An Enumerator[E] produces chunks of data of type E and can be of the 3 following kinds:
 
* Input[E] is a chunk of data of type E: for example Input[Int] holds an Int
* Input.Empty means the enumerator is empty: for example an Enumerator streaming an empty file or an empty collection
* Input.EOF means the enumerator has reached its end: for example an Enumerator streaming a file and reaching the end of file.

Importantly the enumerator is non blocking and asynchronous. Like an iterator, it only produces data when called. 

#Iteratees
An Iteratee consumes the data produced by a Enumerator. It describes how the data is consumed, and often produces some resulting value.
 Let's look at a very simple example. One where we iterate over all the elements and treat them all the same. This shows that 
in simple cases, they are similar complexity to using iterators 

Because of the non blocking nature of play, and that a primary use of Iteratees is with relatively slow producing streams, The idiomatic way
of working with iteratees is actually to work with a Future of an Iteratee

{% highlight scala %} 
import play.api.libs.iteratee._
import scala.concurrent.ExecutionContext.Implicits.global

object IterateeExample {
  def main(args: Array[String]): Unit = {
    val sumIteratee = Iteratee.fold[Int, Int](0) { (total, elt) => total + elt }
    val e1 = Enumerator(1, 234, 455, 987)

    val f: Future[Int] = e1.run(sumIteratee) // this is where the magic happens

    f.onSuccess { case x: Int => println(s"run produced $x") }
    println("Done .. but future might not be executed yet")
    Thread.sleep(1000)
    println("Really Done")
  }
}
{% endhighlight %}
Running that gives (usually: it is non deterministic)

> Done .. but future might not be executed yet
>
> run produced 1677
>
> Really Done

#Iteratees hold state
Let's look at how these two lines works
{% highlight scala %} 
    val sumIteratee = Iteratee.fold[Int, Int](0) { (total, elt) => total + elt }
    val f: Future[Int] = e1.run(sumIterator)
{% endhighlight %}
The first line defines 'how the iteratee is going to work'. The second one 'runs' it over the enumerator. Initially an iteratee with a 'context' of 0 is returned. 
Then each time the iteratee consumes something it returns a new new iteratee with the updated value of (total + elt) is returned. The run method wraps the whole
thing in a future.

That's not at all how we are accustomed to thinking about collections. Usually there is a function involved in collection manipulation. That function might be a mapping
function, or a folding function, but it doesn't change during the running of most collection operations. Here however the Iteratee is actually a wrapper around a context, and has
state. Each time we 'consume the next item from the Enumerator' we make a new Iteratee.

There are actually three classes that implement Iteratee:

|Cont[E]|This holds a value of type E: the thing that the Iteratee is building up. In our example above the 'total so far'|
|Done[E]|This tells us that the Iteratee has done it's thing. It holds the result|
|Error|As the name implies this tells us something has gone wrong. There is an exception accessible from it|

#Writing the 'sumIteratee'
The the Iteratee.fold method is a good way of producing Iteratees, but it's quite helpful to write our own 'the hard way'. That gives us a little more insight into how they work
{% highlight scala %} 
object SumIterateeExample {
  import play.api.libs.iteratee._
  import scala.concurrent.ExecutionContext.Implicits.global
  import scala.concurrent.Future

  def sumIteratee: Iteratee[Int, Int] = {
    def step(total: Int)(i: Input[Int]): Iteratee[Int, Int] = i match {
      case Input.EOF | Input.Empty => Done(total, Input.EOF)
      case Input.El(e)             => Cont[Int, Int](i => step(total + e)(i))
    }
    Cont[Int, Int](i => step(0)(i))
  }
}
{% endhighlight %}
This is pretty simple. There are three classes of Input: EOF, Empty and El. If we've run out of data, we return the total so far wrapped in a Done. If there is a new element
then we are going to a new Cont, which says 'carrying on, and here is how to compute the result'. Note that the parameter to Cont is a function that takes an input and returns an Iteratee

The step method in that object is a helper function. It is mostly needed to make sure that the tail recursion happens. In my experience there is usually a step function, although it sometimes
takes different parameters. 

#Solving the N and a half loop problem
The example above was only a little different to how we would deal with a collection. Let's now see how we can deal with the 'exiting early' problem. Let's exit if any value is more than 'maxValue'
{% highlight scala %} 
  def exitEarlyIteratee(maxValue: Int): Iteratee[Int, Int] = {
    def step(total: Int)(i: Input[Int]): Iteratee[Int, Int] = i match {
      case Input.EOF | Input.Empty => Done(total, Input.EOF)
      case Input.El(e) =>
        if (e > maxValue) // Termination condition
          Done(total, Input.EOF) 
        else
          Cont[Int, Int](i => step(total + e)(i))
    }
    Cont[Int, Int](i => step(0)(i))
  }
 {% endhighlight %}
Here we can see the code is quite clean. Compared to the first example we did with the 'break' code in it anyway. One thing that is nice is that we can parameterise this:

{% highlight scala %} 
  def nPlusOne(finish: Int => Boolean)(fn: (Int, Int) => Int): Iteratee[Int, Int] = {
    def step(acc: Int)(i: Input[Int]): Iteratee[Int, Int] = i match {
      case Input.EOF | Input.Empty => Done(acc, Input.EOF)
      case Input.El(e) => if (finish(e))
        Done(acc, Input.EOF)
      else
        Cont[Int, Int](i => step(fn(acc, e))(i))
    }
    Cont[Int, Int](i => step(0)(i))
  }
  def main(args: Array[String]): Unit = {
    val e1 = Enumerator(1, 234, 455, 987)
    val nPlusOneIterator = nPlusOne(_ > 300){_ + _} 
    val f: Future[Int] = e1.run(nPlusOneIterator)
    ...
  }
 {% endhighlight %}

#Using the values produced by Iteratees
 If I want to work with the value in an Iteratee, I mostly use the map function. So for example the following codes returns a future which
is the total generated by the iterator plus 1.  
{% highlight scala %} 
  val f: Future[Int] = e1.run(nPlusOneIterator).map(total => total + 1)
 {% endhighlight %}
 
#Enumeratees
 This is a part of the framework I hardly ever use. The problem it solves is 'suppose I have a sumIterator that works with Ints, but I actually have an input stream that is all Strings representing the Ints, how do I use it'
 {% highlight scala %} 
 object EnumerateeExample {
  def main(args: Array[String]): Unit = {
    val sumIteratee = Iteratee.fold[Int, Int](0) { (total, elt) => total + elt }
    val e1 = Enumerator("1", "234", "455", "987")
    val f: Future[Int] = e1.through(Enumeratee.map(_.toInt)).run(sumIteratee)
    ...
  }
}
 {% endhighlight %}
All the Enumeratee is doing is mapping from whatever the Enumerator is, to the type we want. A little clunky but useful

#Signatures
Iteratees have two generics. The type they consume and the type they produce. In the following E is the type of the input, and A is the type that they are producing
{% highlight scala %} 
 trait Iteratee[E, +A] 
{% endhighlight %}

#Summary
Iteratees are a core part of the Play framework, although well hidden. Understanding them is useful but not essential. They do solve the number of problem of how
to consume data from a source of some sort, constructing things, with early termination and error handling in a asynchronous non blocking way well. If you think about what a
website is... well it's consuming data from a socket, and constructing a HTML page. Some pages need early termination. Some need error handling. In Play all pages should be asynchronous 
and non blocking. 

I mostly use Iteratees when dealing with large files that I don't want to load into memory. I'll show how this is done in the next blog on file uploading.

  
