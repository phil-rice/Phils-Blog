---
layout: post
title:  "Type Classes in Scala: A Real World Application"
date:   2015-11-09 11:06:23 +0000
comments: True
---

#Overview
I am engaged in a legacy modernisation task on a job that is working out the possible routes that people might of traveled on. 
The modernisation project will take many tens of man years of developer time, and because there are significant sums of money 
involved in the results, the results of the modernised system have to be able to exactly duplicate the behaviour of the legacy.

One activity that is undertaken a lot is ‘what is the shortest mileage between location X and location Y. 
This is an activity that it would be ‘good thing’ to do quickly. Even taking a microsecond to undertake it would mean over 8 minutes would be taken up by the mileage calculations, 
and a goal is to have the whole batch done done in under two minutes. Part of this activity, which is only executed once, is to either calculate the mileage data, or to load the 
results (whichever is quicker).

[Most of the code in here can be found here] (https://github.com/phil-rice/MileageForBlog)

There is a [discussion on Unit Testing this sort of code](/2015/11/11/Unit-Testing-Scala-Type-Classes.html) which changes a little of the code in the Distance Type Class.

## Quick stats

|Type|Expected number|
|:--- |---:|
|Locations that can be the source or destination| 2500|
|Location to Location mileages that need to be calculated| 6,250,000|
|Number of lookups that will be done| 500,000,000|

There are three entities that need considering in this profiling example, and I will be looking at how Type Classes can help ease the task of evaluating options. The entities are

|Distance|How far between locations. Resolution is 1/100 of a mile|
|Mileage Matrix|Given two locations this matrix holds the distance between those two locations.| 
|Mileage Matrix Factory|The code used to create a Mileage Matrix|

The profiling technique used is two fold. As it happens my primary one gives an early win, and that is to listen to the CPU fan in my computer. A poor algorithm (even single threaded) 
spams all eight cores with the garbage collection and causes the computer to heat up. My secondary mechanism is more qualitative and relies on just timing how long it takes. This ‘timing 
how long it takes’ is outside the scope of this text, but needs to worry about class loading times, JVM optimisation and garbage collection.

# Scala Type Classes
One of my goals is to compare the operation of Ints and Shorts. The world has profiling has changed quite a lot with modern CPU architectures. For example a primary change is the impact 
of the ‘working set’ in the memory caches. Shorts take less memory (especially when there millions of them, and I am interested in seeing if we can see the impact of the reduced
memory size on actual performance.

In Object orientated languages if we want to have a MileageMatrixFactory[D], where D is an Int or a short, and do operations like adding and comparing value it can be quite complicated. If
we look at this in Java, the approach would have to be

* Wrapping the Int or Short in another class: not desirable given the comments on memory above. This class can have add and less than operations
* Passing what are effectively functions into the MileageMatrixFactory that perform add and less than. The performance impact of this approach can be very high as we shall see later
* Having MileageMatrixFactory<D extends Number>, we would then have to use calls like intValue on the object, and very probably the compiler would be doing autoboxing which would impact heavily on our approach

In Scala however there is an idiomatic way to handle this problem that doesn't result in wrapping the raw data types with objects. This is called 'Type Classes'. We start using the Distance 
Type Class by defining what operations we want to perform on distance. In addition it defines 'zero distance', as that turns out to very useful.
{% highlight scala %}
trait Distance[D] {
  def add(d1: D, d2: D): D
  def lessThan(d1: D, d2: D): Boolean
  def zero: D
}
{% endhighlight %}

Once we have this, we can actually define 'Distance-like' behaviour for Ints and Shorts. Because this is done in the Distance companion object, these type classes will automatically be detected 
if our code 'wants one' for Ints or Shorts. The code is pretty simple   
{% highlight scala %}
object Distance {
  implicit object DistanceLikeForInt extends Distance[Int] {
    def add(d1: Int, d2: Int) = d1 + d2
    def lessThan(d1: Int, d2: Int) = d1 < d2
    def zero=0
  }
  implicit object DistanceLikeForShort extends Distance[Short] {
    def add(d1: Short, d2: Short) = d1 + d2
    def lessThan(d1: Short, d2: Short) = d1 < d2
    def zero=0
  }
}
{% endhighlight %}
I first came across this pattern as the Flyweight pattern in the Gang Of Four. In Java I very rarely use it because the code becomes painful to read. In Scala that is much less so.

## Using the distance type class
This shows the first way that we might use the type class. The implicit distanceLike tells the compiler that there must be a Distance[D] implicitly declared in scope. As long as D is an Int or Short
then that DistanceLikeForXXX will be plumbed in for use
 
{% highlight scala %}
object UsingDistanceDemo {
  def addDistance[D](distances: D*)(implicit distanceLike: Distance[D]) = {
    import distanceLike._
    distances.foldLeft(zero)(add)
  }
  
  def main(args: Array[String]): Unit = {
    println(addDistance(1, 2, 3))
    println(addDistance[Int](1, 2, 3))
    println(addDistance[Short](1, 2, 3))
  }
}
{% endhighlight %}
In the main method there are three calls. It is worth noting the simplicity of the first call. It says 'add up distances 1, 2 and 3'. The code that does the adding up is very simple as well. The second
line in the main method shows how to explicitly say 'and the distances are Ints', while the third says 'and the distances are Shorts'. In the first two lines DistanceLikeForInt was plumbed in. In 
the third line DistanceLikeForShort was plumbed in

## Simpler type class notation
This code is a little verbose though. The text "(implicit distanceLike: Distance[D])" gets in the way of reading. It turns out we can simplify the way to pass the 'distanceLike' around. The method 
isSumOfDistancesLessThen below shows how this can be done.  This new syntax below is quite exciting in that the method or class signatures read a lot better. 
It is possible to chain Type classes as well. D could be a Distance and a Measurement for example, in which case the method signature would be def isSumOfDistancesLessThen[D:Distance:Measurement]. 
With the new syntax it is sometimes important (not always: many times the method delegates to another method) to be able to get the 'distanceLike' implicit, and the code shows how to do so. 'implicitly[Distance[D]]' 
says 'get me the implicit value for Distance[D]' 
{% highlight scala %}
object UsingDistanceDemo {
  def isSumOfDistancesLessThen[D: Distance](targetDistance: D)(distances: D*) = 
    implicitly[Distance[D]].lessThan(addDistance(distances:_*), targetDistance)
}
{% endhighlight %}

In practice I found that adding methods to distance like zero, large and random, made it really useful while testing. I could write a test framework for 'testing distancelike behaviour' and then just inherit from
it with the concrete class. Another useful thing was being able to make arrays of things with distancelike behaviour. In Java it is hard to create an array for a something when you don't know what concrete type 
it is. Normally you end up having to pass some plumbing around: In Java I usually pass around the class of the type. With this pattern, I only put 'create the array' code in concrete classes, so no more plumbing. Mind
you for most code I don't care: I only use Arrays when I am worrying about highly performant code, as part of a profiling activity. 

By the end of the profiling, many more methods were added to Distance, ending up with 
{% highlight scala %}
trait Distance[D] {
  def zero: D
  def large: D
  def random: D
  def add(d1: D, d2: D): D
  def lessThan(d1: D, d2: D): Boolean
  def makeArray(size: Int, value: D): Array[D] 
  def makeArrayArray(size: Int, value: D): Array[Array[D]] 
}
{% endhighlight %}

# A side note on AnyVals
During this profiling work a constant worry is about memory usage. There will be millions of these Distance like objects 'D' around, and there will be half a billion calls involving them. In Java any time that you put a short or an int into 
a generic data structure, then it needs to wrapped with a boxing object of type Short or Integer. Scala has a very very nice feature called AnyVals that means that at run time variables of type Int or Short vanish and
are replaced by the primitive types. 

# Representing the Mileage result
The resulting data structure that is wanted is logically a mapping from a pair of locations to the distance between them. There are a number of data structures that we might to use.
These include the following (D will be something with a Distance[D] type class). In practice it will be an Int or a Short.
{% highlight scala %}
Map[(Int, Int), D]
Array[Array[D]] 
{% endhighlight %}
Other types will be used as well, but just looking at these two will show the basics.

Like the Distance type class, we begin with a trait that defines the properties needed. The first iteration was below. It defines the ability to make an empty structure pre-populated with a value, the ability to get an element, and the ability to put 
data into the structure. There is no requirement for the structure to be mutable or immutable. The Maps for example will be an immutable structure with all the nice properties that implies, and the ones backed by arrays definitely won't be!

It is important to note that this type class is more complicated than Distance. Indeed purists might claim it isn't a Type Class at all (something I'm not even slightly worried about). 
MM is the data structure that has Matrix like behaviour, and D is the chosen way of representing distance. 
{% highlight scala %}
trait Matrix[D, MM] {
  def makeRaw(numberOfLocations: Int, value: D): MM
  def get(hs: MM, l1: Int, L2: Int): D
  def put(hs: MM, l1: Int, l2: Int, d: D): MM
{% endhighlight %}

With Distance we went straight to the Distance companion object and coded up implicit objects for Int and Short. Here however we have the issue that the Matrix type class is actually parameterized by D 
(where D has Distance like behaviour). To deal with that we produce classes that are parameterised by 'D:Distance'.  It turns out that creating Arrays of type D in Scala requires plumbing, and using
type classes made it very easy to do that plumbing by adding makeArrayArray(numberOfLocations,value) a method on the Distance[D] type class

{% highlight scala %}
class Matrix_Map_Tuple_Like[D] extends Matrix[D, Map[(Int, Int), D]] {
  type MM = Map[(Int, Int), D]
  def makeRaw(numberOfLocations: Int, value: D) = { 
       val locs = 0 to numberOfLocations - 1; 
       (for { l1 <- locs; l2 <- locs } yield ((l1, l2) -> value)).toMap }
  def get(mm: MM, l1: Int, l2: Int) = mm((l1, l2))
  def put(g: MM, l1: Int, l2: Int, d: D) = g + ((l1, l2) -> d)
}
class Matrix_Array_Array_Like[D: Distance] extends Matrix[D, Array[Array[D]]] {
  type MM = Array[Array[D]]
  def makeRaw(numberOfLocations: Int, value: D): MM = 
      implicitly[Distance[D]].makeArrayArray(numberOfLocations, value)
  def get(g: MM, l1: Int, l2: Int) = g(l1)(l2)
  def put(g: MM, l1: Int, l2: Int, d: D) = { g(l1)(l2) = d; g }
}
{% endhighlight %}

Having made the parameterised Matrix traits or abstract classes, we can now create the Matrix companion object. This is quite interesting as the way we do this avoids the combinatorial 
explosion of needing a separate object for each combination of Matrix and Distance. As the work progressed, it was found necessary to break this pattern as Java classes
were added that used Java primitives. Examples of that can be found in the GitHub repository, but are as simple as these
{% highlight scala %}
object Matrix {
  implicit def asMatrix_Map_Tuple_Like[D: Distance] = new Matrix_Map_Tuple_Like[D]
  implicit def asMatrix_Array_Array_Like[D: Distance] = new Matrix_Array_Array_Like[D]
 ...
}
{% endhighlight %}

#Calculating the mileage.
The [Floyd Warshell algorithm](https://en.wikipedia.org/wiki/Floyd%E2%80%93Warshall_algorithm) is simple to use and understand and does exactly what is needed. I did have to edit the wiki page (a very scary moment) as there was an error on it involved a
 ‘<’ sign.  
 
 Unfortunately as we shall see the really nice 'D: Distance' notation cannot be used with Matrixes. It would be nice if we could have MileageFactory[D: Distance: Matrix[D]], but Scala (as 
 far as I know) has no mechanism for this. If I'm wrong please let me know! Even so the code is not too ugly
{% highlight scala %}
case class MileageEdge[D: Distance](from: Int, to: Int, distance: D)

abstract class MileageFactory[D: Distance, MM](implicit matrixLike: Matrix[D, MM]) {
  protected val distanceLike = implicitly[Distance[D]]
  import distanceLike._
  def apply(locationSize: Int, edges: Traversable[MileageEdge[D]]): Mileage[D, MM]
  protected def createAndInitialiseMatrix(locationSize: Int, edges: Traversable[MileageEdge[D]]) = {
    val withLocToLocBeingZero = (0 to locationSize - 1).
      foldLeft(matrixLike.makeRaw(locationSize, large))(
        (hs, loc) => put(hs, loc, loc, zero))
    edges.foldLeft(withLocToLocBeingZero) {
      case (mm, MileageEdge(from, to, distance)) =>
        put(put(mm, from, to, distance), to, from, distance)
    }
  }
}
}

/** D is how the distance is measured, and MM is the mileage matrix representation  */
class MileageFactory1[D: Distance, MM](implicit matrixLike: Matrix[D, MM]) extends MileageFactory[D, MM] {
  import distanceLike._
  import matrixLike._
  def apply(locationSize: Int, edges: Traversable[MileageEdge[D]]) = {
    val locs = 0 to locationSize - 1
    var mm = createAndInitialiseMatrix(locationSize, edges)
 for { k <- locs; i <- locs; j <- locs } {
      val newDistance = add(get(mm, i, k), get(mm, k, j))
      if (lessThan(newDistance, get(mm, i, j)))
        mm = put(mm, i, j, newDistance)
    };
    new Mileage(mm)
  }
}
{% endhighlight %}
The code has been split up into an abstract and concrete class as several versions of MileageFactory are going to be evaluated. Making the initial value takes very little time, so there is little point 
varying it. Almost all the time is in the three deep nested loop of k, i and j.

# Profiling
The actual code used for profiling tends to be messy involving lots of 'warm up' code, as the performance varies when the JVM has run the loops a few tens of thousands times, and to try and isolate
the garbage created by one run from another. Simplified code (with the timing code taken out) can be seen here though:
{% highlight scala %}
  def makeSimpleEdges(locationSize: Int) = (0 to locationSize).
     map(_ => (Random.nextInt(locationSize), Random.nextInt(locationSize)))

  def makeEdges[D: Distance](locationSize: Int) = { 
    var result = List[(Int, Int)]()  // a use of var... if someone has a cool way of doing this without it let me know
    while (result.size < locationSize) {
      result = (result ++ makeSimpleEdges(locationSize)).distinct
    }
    val distanceLike = implicitly[Distance[D]]
    result.take(locationSize).map(kv => MileageEdge[D](kv._1, kv._2, distanceLike.random))
  }
  
  def makeRandomLocations(locationSize: Int, timesToUse: Int) = {
    def randomLocs = (1 to timesToUse).map(_ => Random.nextInt(locationSize))
    val loc1 = randomLocs
    val loc2 = randomLocs
    (loc1, loc2)
  }

  def makeAndUse[D: Distance, MM](locationSize: Int, timesToUse: Int, 
          factory: MileageFactory[D, MM])(implicit graphLike: Matrix[D, MM]) = {
    val edges = makeEdges(locationSize)
    val mileage = factory.apply(locationSize, edges) // timing this will give us the 'making time
    val (loc1, loc2) = makeRandomLocations(locationSize, timesToUse)
    for (i <- 0 to timesToUse - 1)  // timing this loop gives us the 'usage time'
      mileage(loc1(i), loc2(i))
  }

  def main(args: Array[String]): Unit = {
    makeAndUse(100, 50000000, new MileageFactory1[Int, Array[Array[Int]]])
    makeAndUse(100, 50000000, new MileageFactory1[Short, Map[(Int,Int), Short]])
  }
{% endhighlight %}
An important line is the plumbing line which is the lines in the main method. Both say make with 100 locations, and read from
it fifty million times. In the first the Distance is an Int, and the matrix is modeled by an Array[Array[Int]]. In the second 
distance is modeled by a Short and and the matrix by a map from the two locations to the distance.

# Results
The results were more interesting than I expected. Like always when I start a Profiling activity, I 'know' how to speed it up. And 
like always my 'I know' turns out to be 'I was wrong'. My simple model was that 'speed is dominated by memory usage in the new world'.
That meant that data structures with Shorts in them should be quicker than those with Ints. I expected the Array[Array[X]] to be slower than 
Array[X] as the first approach has two memory look ups, and the second only one. I was wrong on both counts.

The three mileage factories (MF1, MF2 and MF3) have different ways of writing the really tight loops in Scala. The implementations are not that important (but can be seen in the Git Repository), 
what is interesting is that with the faster matrix representations there was about a 30% time saving. It depends on the application, and how much time the loop takes in total as to whether the rewriting is worth it

All times are in in miliseconds. MF1/MF2/MF3 are how long it took to create the mileage matrix with the relevant factory. The using time is how long it will take to do half a billion random accesses

|Data Structure|Locations|MF1|MF2|MF3|Using|
|:---|---:|---:|---:|---:|---:|
|Array[Array[Int]]|100|32|24|20|1,720|
||1000|18,619|15,124|13,341|4,000|
||2500|290,282|237,047|207,028|**12,000**|
|Array[Array[Short]]]|100|49|43|24|2,500|
||1000|47,049|44,000|26,120|8,000|
||2500|745,606|706,760|415,435|**21,000**|
|ArrayOfLocationsAndSize[Int]]|100|42|35|24|4,700|
||1000|37,540|22,881|19,766|8,000|
||2500|581,839|355,816|315,056|**17,000**|
|ArrayOfLocationsAndSize[Short]]|100|59|48|43|7,700|
||1000|61,294|60,266|56,748|8,000|
||2500|987,020|992,660|887,243|**24,000**|
|ShortArrayAndLength|100|37|33|30|1,740|
||1000|25,008|23,164|19,488|3,100|
||2500|408,497|377,078|303,566|**10,000**|
|IntArrayAndLength|100|24|16|16|1,780|
||1000|19,600|17000|13000|4,000|
||2500|310,505|276,784|217,474|**10,000**|

Because of the ease of adding new Matrix types, I actually have a quite a few other representations too. The last two (ShortArrayAndLength and IntArrayAndLength) are both Java classes using
short[] and int[]. It is worth noting that they are only just faster than Array[Array[Int]] in the time to access the data


#Expected Results 

## Java native code is much faster
Some of the Matrix like classes I created were native Java using short[] and int[]. They were just faster than using Scala values. I rewrote the MatrixFactory in 
Java, and that was perhaps an order of magnitude faster. The speed up is (I think) because in native Java there were no virtual method calls
so the Just In Yime compiler could in-line everything. Passing closures, or functions, into the heart of loops that are to run fast may not work for high performance code!

##The bigger the Matrix the slower the access times
Once again this is because of the cost of the cache. If the matrix is small then it fits into the cache (there are levels of cache). As the matrix gets bigger it gets slower to access it as 
(because of the random nature of access) the time to get the data into the cache dominates everything. A good example of this can be seen looking at the 'Using' column for 'ShortArrayAndLength'. 
This varies in time from one second to ten seconds.

One observation about this test is that actually it isn't representative: it's worst case. In the actual job the data access is anything but random. 

## Nice programing cost
The use of ‘nice programming practices’ comes at a cost. Code like this which is in MileageFactory1
{% highlight scala %}
 for { k <- locs; i <- locs; j <- locs } {
      val newDistance = distanceLike.add(matrixLike(mm, i, j), matrixLike(mm, k, j))
      if (distanceLike.lessThan(matrixLike(mm, i, j), newDistance))
        mm = matrixLike.put(mm, i, j, newDistance)
    };
{% endhighlight %}
Takes *maybe* thirty percent more as the following (which is in MileageFactory3). I would assert than in most cases the clarity of code isn’t worth the speed up
{% highlight scala %}
 def apply(locationSize: Int, edges: Traversable[MileageEdge[D]]) = {
    var mm = createAndInitialiseMatrix(locationSize, edges)
    var k = 0;
    while (k < locationSize) {
      var i = 0;
      while (i < locationSize) {
        var j = 0;
        while (j < locationSize) {
          mm = matrixLike.addIfBetter(mm, k, i, j)
          j += 1
        }
        i += 1
      }
      k += 1
    }
    new Mileage[D, MM](mm)
  }
{% endhighlight %}
The Java equivalent though was pretty clean (actually better than the Scala code for this, in my opinion), and as I say goes about ten times faster
{% highlight java %}
for (int k = 0; k < vertexCount(); k++)
	for (int i = 0; i < vertexCount(); i++)
		for (int j = 0; j < vertexCount(); j++) {
			int d = get(i, k) + get(k, j);
			if (d < get(i, j))
				put(i, j, d);
		}
	}
}
{% endhighlight %}

#Surprise Results

## Shorts were worse than Ints
Look at the time difference between the Java memory structures using short[] and those using Array[Short]. I have no idea why at the moment, other than to suspect
that the Scala compiler could do better. 

## Array[Array[Int]] was better than Array[Int]
I predicted incorrectly which would work better: an Array[Array[Int]] or ArrayOfLocationsAndSize[Int] I thought the second would be going through less memory locations. 
I still don’t know the reason that there is such a big difference between these two models: I am guessing it might be because of the inability of the JVM to inline some of 
the method calls, but that’s just speculation 

# The value of measuring instead of guessing
My guess at the start of this was that the ShortArrayAndLength would be the fastest. It uses the least memory, and the time to do the calculation was probably far shorter than two array look ups. 
It actually ended up taking a long time. I didn’t think that the move to Java would be worth doing. I was wrong on both grounds. Whenever I have started profiling work I have been surprised at 
the results, and this was no exception. Hence my mantra of ‘measure don’t guess’ when it comes to this sort of code optimisation.

# What did I actually implement in the end
I used a Java data structure to create the data, using Ints, and the copied those into a data structure with an underlying Java structure. I am currently working out whether to use int[] or short[]. 
In the actual application access to the mileage matrix isn’t random and tends to cluster. I am currently in the process of measuring this in the full batch job. Initial indications are the the short[] 
is much faster. This data structure I saved to a file, so that when running the batch job I just had to read the file and populate the data structure. This takes a few seconds instead of the minute or so that the 
optimised Java code takes, or the around five minutes that the standard Scala code takes

# How much time did this save
The whole Batch job started at about 300 seconds single threaded. Doing this process made me rewrite the access code to use native Java arrays. Doing that actally reduced that time to 210s: a significant saving. 
That is a much better saving that the above figures would indicate, and I think that is because of the non random nature of the access queries. The other difference was moving to multiple cores. With
eight cores the 300 seconds still took 200 seconds, whereas with the new structure it took about 120s. My understanding of these results is that Garbage Collection is a big overhead, and gets bigger the more cores that are used.

#Benefit of the Type Classes
Without doubt because of the ease of adding new data representations I evaluated many more Matrix types than I would of otherwise. I could say 'I wonder what this would do' and just add another one. I
personally gained a much better idea of how the data structure and access methods affected execution times.
