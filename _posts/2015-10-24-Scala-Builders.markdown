---
layout: post
title: "Scala Builders for Java Classes"
date:   2015-10-24 11:06:23 +0000
comments: True
categories:
- scala
---

I am currently in the unhappy position of having to use a Java data structure to hold my data. The data structure  will be accessed several billion times in under two minutes, and long[] is  sufficiently quicker than Array[Long], that the pain is worth it. The longs in the long[] are actually going to be an AnyVal class OttAsLong.

This is quite a common pattern for high performance code. The performance of most high speed algorithms is dominated by cache usage. This approach is giving me type safety for no run time cost, and allows the code that doesn't need to be performant (almost all of it) fully type safe access to the data

[The code for this can be found here](https://github.com/phil-rice/ScalaBuildersBlog)

{% highlight java %}
public class LongArrayAndLength {
	private long[] data;
	private int size = 0;
	public LongArrayAndLength(long[] data, int size) {
		this.data = data;
		this.size = size;
	}
	public LongArrayAndLength(int maxSize) {
		this.data = new long[maxSize];
	}
	public int add(long l) {
		data[size++] = l;
		return size - 1;
	}
	public long get(int index) {
		return data[index];
	}
	public int getSize() {
		return size;
	}
}
{% endhighlight %}

I want to be able to do the very nice Scala operations such as map, fold and so on on this object. As the only time I will be doing this will be times when I don't have
the same worry about performance (testing, reporting, diagnostics) I don't have the same paranoid worries about performance, and instead revert to my normal values of 
clarity and ease of writing.

The AnyVal I will be using to wrap the long in is below. The details don't really matters, although as you can see it uses the long to represent a 'from' and a 'to' and the 'from' and 'to' represent a value and a size. All we need worry about is that it has a 'toString' so we will be able to see if our
methods are working.
{% highlight scala %} 
class OttAsLong(val ott: Long) extends AnyVal {
  def toPart = new ValueAndSize(LongTools.hi(ott)) 
  def fromPart = new ValueAndSize(LongTools.lo(ott)) 
  override def toString = s"OttAsLong($toPart <== $fromPart)"
}
class ValueAndSize(val int: Int) extends AnyVal {
  def size = int & 7
  def value = int / 8
  override def toString = s"($value/$size)"
}
{% endhighlight %}
# Iterators
Let's start off with the easy stuff. Before we can do map and fold, let's do 'find', 'for each' and so on. For those all that is needed is
{% highlight scala %} 
object ScalaBuildersBlog {
  implicit def asIterator[X](x: LongArrayAndLength) = new Iterator[OttAsLong] {
    var index = 0
    def hasNext = index < x.getSize - 1
    def next = {
      val result = x.get(index)
      index += 1
      new OttAsLong(result)
    }
  }
}
{% endhighlight %}
We can try out this new code
{% highlight scala %} 
  def main(args: Array[String]): Unit = {
    val l = new LongArrayAndLength(100)
    l.add(1)
    l.add(2)
    l.add(3)
    l.foreach(println(_))
  }
{% endhighlight %}
And to our happiness we get the results  
{% highlight scala %} 
OttAsLong((0/0) <== (0/1))
OttAsLong((0/0) <== (0/2))
OttAsLong((0/0) <== (0/3)))
{% endhighlight %}
What's happening here is that the implicit asIterator is being used to turn the LongArrayAndLength into an Iterator[OttAsLong]. As the Iterator has the method 'toList' on it, it is being
used in the println.

# Map and Fold
Well iterating is all very well, but I want to be able to map, fold, flatMap. sort, filter, groupBy and ... and lots of things. All the nice operators that would allow me to use this as a 
first class data structure in Scala. The main question I need to answer to do this is 'what do I have to implement to make this magic happen'. Let's start with a look at the signature of 'map' in Traversable
{% highlight scala %} 
  override def map[B, That](f: A => B)(
                   implicit bf: CanBuildFrom[Traversable[A], B, That]): That
{% endhighlight %}
Well that's pretty straightforwards. We can see what we need to do. We need to implement CanBuildFrom[LongArrayAndLength, B, LongArrayAndLength]. Let's look at the signature
{% highlight scala %} 
trait CanBuildFrom[-From, -Elem, +To] {
  def apply(from: From): Builder[Elem, To]
  def apply(): Builder[Elem, To]
}
{% endhighlight %}
And now we are starting down the rabbit hole. CanBuilderFrom immediately pushes us to look at Builder. How far does this rabbit hole go? The answer is not very far
{% highlight scala %} 
trait Builder[-Elem, +To] extends Growable[Elem] {
  def +=(elem: Elem): this.type
  def clear()
  def result(): To
  def sizeHint(size: Int) {}
 }
{% endhighlight %}

Both CanBuilderFrom and Builder look pretty straightforwards, especially if I am not worried about performance
{% highlight scala %} 
  implicit object LongArrayAndLengthCanBuildFrom extends 
        CanBuildFrom[LongArrayAndLength, OttAsLong, LongArrayAndLength] {
    def apply(from: LongArrayAndLength): LongArrayAndLengthBuilder =
       new LongArrayAndLengthBuilder(from)
    def apply(): LongArrayAndLengthBuilder = ???
  }
  class LongArrayAndLengthBuilder(from: LongArrayAndLength) extends 
       Builder[OttAsLong, LongArrayAndLength] {
    var target = from.toVector
    def clear() {target = Vector()}
    def +=(elem: OttAsLong) = { target = target :+ elem; this}
    def result() = new LongArrayAndLength(from.map(_.ott).toArray, from.size)
  }
{% endhighlight %}

Now we can do the following
{% highlight scala %} 
  def main(args: Array[String]): Unit = {
    val l = new LongArrayAndLength(100)
    l.add(1)
    l.add(2)
    l.add(3)
    println(l.flatMap(x => List(x.fromPart, x.toPart)).toList)
  }
{% endhighlight %}

# Summary
It was remarkably easy to give my own data structure the goodness of the Scala collections framework. I had to do the following
* implement a 'asIterator'
* implement a 'Builder'
* implement a 'CanBuildFrom'
None of which took more than a few minutes to do. 

If the Java class had been a Scala class, I could of implemented these in the companion object which would of meant I didn't need to do some import magic to use this,
however as it's a Java class we need to make a dedicated object.


