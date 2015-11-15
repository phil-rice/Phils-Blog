---
layout: post
title: "Dealing With Combinatorial Explosions When Unit Testing"
date:   2015-11-11 08:06:23 +0000
comments: True
categories:
- scala
- testing
---
In my previous post when on scala test with type classes, I showed how an abstract class could be used, and then each concrete instance of the type class could inherit it. This worked really 
well with such things as Distance:

{% highlight scala %}
abstract class DistanceTests[D: Distance] extends FlatSpec with Matchers {
  val distanceLike = implicitly[Distance[D]]
  implicit def toD(x: Int): D
  def zero: D = 0
  def one: D = 1
  def two: D = 2
  def three: D = 3
  def large: D

  "Distance" should "implement zero and large" in {
    distanceLike.zero shouldBe zero 
    distanceLike.large shouldBe large
  }

  it should "implement addition" in {
    distanceLike.add(one, two) shouldBe three 
  }
  ... more tests

}

class DistanceIntTests extends DistanceTests[Int] { 
   implicit def toD(x: Int) = x; val large = Int.MaxValue }
class DistanceShortTests extends DistanceTests[Short] { 
   implicit def toD(x: Int) = x.toShort; val large = Short.MaxValue }
{% endhighlight %}

However when we moved to more MatrixLike, we needed to instantiate an instance with each concrete class for distance, and each concrete type for the matrix:
{% highlight scala %}
abstract class MatrixLikeTests[D: Distance, MM](implicit matrixLike: Matrix[D, MM]) 
          extends WordSpec with Matchers {
  val distanceLike = implicitly[Distance[D]]
  import matrixLike._
  implicit def toD(x: Int): D
  def zero: D = 0
  def one: D = 1
  def two: D = 2
  def three: D = 3
  "Matrix" when {
    "makeRaw called" should {
      "be full of the value" in {
        val m = makeRaw(5, two)
        for { i <- 0 to 4; j <- 0 to 4 } get(m, i, j) shouldBe two
      }
    }
   ... mores tests
}

abstract class MatrixLikeIntTests[MM](implicit matrixLike: Matrix[Int, MM]) extends MatrixLikeTests[Int, MM] {
  implicit def toD(x: Int) = x;
}
abstract class MatrixLikeShortTests[MM](implicit matrixLike: Matrix[Short, MM]) extends MatrixLikeTests[Short, MM] {
  implicit def toD(x: Int) = x.toShort;
}

class MatrixLikeArrayArrayIntTests extends MatrixLikeIntTests[Array[Array[Int]]]
class MatrixLikeMapMapIntTests extends MatrixLikeIntTests[Map[Int, Map[Int, Int]]]
class MatrixLikeHashMapHashMapIntTests extends MatrixLikeIntTests[HashMap[Int, HashMap[Int, Int]]]
class MatrixLikeMapTupleToIntTests extends MatrixLikeIntTests[Map[(Int, Int), Int]]
class MatrixLikeHashMapTupleToIntTests extends MatrixLikeIntTests[HashMap[(Int, Int), Int]]
class MatrixLikeArrayOfLocationsIntTests extends MatrixLikeIntTests[ArrayOfLocationsAndSize[Int]]
class MatrixLikeIntArrayAndLengthTests extends MatrixLikeIntTests[IntArrayAndLength]

class MatrixLikeArrayArrayShortTests extends MatrixLikeShortTests[Array[Array[Short]]]
class MatrixLikeMapMapShortTests extends MatrixLikeShortTests[Map[Int, Map[Int, Short]]]
class MatrixLikeHashMapHashMapShortTests extends MatrixLikeShortTests[HashMap[Int, HashMap[Int, Short]]]
class MatrixLikeMapTupleToShortTests extends MatrixLikeShortTests[Map[(Int, Int), Short]]
class MatrixLikeHashMapTupleToShortTests extends MatrixLikeShortTests[HashMap[(Int, Int), Short]]
class MatrixLikeArrayOfLocationsShortTests extends MatrixLikeShortTests[ArrayOfLocationsAndSize[Short]]
class MatrixLikeShortArrayAndLengthTests extends MatrixLikeShortTests[ShortArrayAndLength]
{% endhighlight %}														

This gets even worse when I move to the mileage factories as at the moment I have three types of mileage factory, 2 distances and seven matrices... 
that would require me to implement 42 classes. I would like to find a better way to deal with this. I have a number of concerns. The first is the 
work when I add a new Distance, Matrix or MileageFactory, the second is spotting errors of omission. 

#Scala Test Documentation
A good place to start when messing with test fixtures is the documentation. [I started with this](http://doc.scalatest.org/1.8/org/scalatest/FlatSpec.html)

This lead me to doing the following:
{% highlight scala %}
 case class DistanceTestDefn[D: Distance](prefix: String, toDFn: (Int) => D, large: D){
	  val distanceLike = implicitly[Distance[D]]
  }
  class AllDistanceLikeTests extends FlatSpec with Matchers {
 

  def distance[D](defn: DistanceTestDefn[D]) {
    import defn._
    implicit def toDo(x: Int) = toDFn(x)
    def zero: D = 0

    s"Distance$prefix" should "implement zero and large" in {
      distanceLike.zero shouldBe zero
      distanceLike.large shouldBe large
    }
   ...more tests
  }


  val defns = List(DistanceTestDefn("Int", _.toInt, Int.MaxValue), 
                   DistanceTestDefn("Short", _.toShort, Short.MaxValue))
  defns.foreach { defn =>
    s"Distance[${defn.prefix}]" should behave like distance(defn)
  }

}
{% endhighlight %}	

And when I run it, I get the full testing of Distance[Int] and Distance[Short]. The list called 'defns' includes all the things that the tests needs to run. Scala Test allows us to call a function 
with each of these.

There isn't much saving (actually it's slightly worse code) with Distance, because there is no combinatorial explosion. I am hoping to be able to do something similar with Matrix though, and build the combinations
up programatically. Let's see how that might work. My first attempt didn't work:

{% highlight scala %}
case class MatrixLikeTestDefn[D, MM](prefix: String, matrix: MM)(
                              implicit val matrixLike: Matrix[D, MM])

class AllMatrixLikeTests extends WordSpec with Matchers {
  def matrix[D, MM](matrixDefn: MatrixLikeTestDefn[D, MM], 
                    distanceDefn: DistanceTestDefn[D]) {
    import distanceDefn._
    import matrixDefn.matrixLike._
    implicit def toDo(x: Int) = toDFn(x)
    def two: D = 2
    def three: D = 3

    "Matrix" when {
      "makeRaw called" should {
        "be full of the value" in {
          val m = makeRaw(5, two)
          for { i <- 0 to 4; j <- 0 to 4 } get(m, i, j) shouldBe two
        }
      }
      ...other tests
    }
  }
  val distanceDefns = List(DistanceTestDefn("Int", _.toInt, Int.MaxValue), 
                           DistanceTestDefn("Short", _.toShort, Short.MaxValue))
  val matrixDefns = List( .... what goes here .... )
  for { d <- distanceDefns; m <- matrixDefns }
    s"Matrix[${d.prefix},${m.prefix}]" should behave like matrix(d, m)
}
{% endhighlight %}	
The problem is that I specified D in the MatrixLikeTestDefn[D, MM]. This meant I had to make a concrete class for it, and I had my combinatorial explosion back again. 

What I actually wanted was something that looked more like this

 {% highlight scala %}
case class MatrixTestDefn(prefix: String, matrixLike: Distance[_] => Matrix[_, _])

class AllMatrixLikeTests extends WordSpec with Matchers {

  def matrix[D, MM](distanceDefn: DistanceTestDefn[D], matrixDefn: MatrixTestDefn) {
    import distanceDefn._
    val matrixLike = matrixDefn.matrixLike(distanceLike).asInstanceOf[Matrix[D, MM]]
    import matrixLike._
    implicit def toDo(x: Int) = toDFn(x)

    def zero: D = 0
    def one: D = 1
    def two: D = 2
    def three: D = 3

    "Matrix" when {
      "makeRaw called" should {
        "be full of the value" in {
          val m = makeRaw(5, two)
          for { i <- 0 to 4; j <- 0 to 4 } get(m, i, j) shouldBe two
        }
      }
     ..more tests
    }
  }
  val distanceDefns = List(DistanceTestDefn("Int", _.toInt, Int.MaxValue), 
                           DistanceTestDefn("Short", _.toShort, Short.MaxValue))
  val matrixDefns = List(
    MatrixTestDefn("Map[(Int, Int), D]", d => Matrix.asMatrix_Map_Tuple_Like(d)),
    MatrixTestDefn("HashMap[(Int, Int), D]", d => Matrix.asMatrix_HashMap_Tuple_Like(d)),
    MatrixTestDefn("Map[Int, Map[Int, D]]", d => Matrix.asMatrix_Int_Map_Map_Like(d)),
    MatrixTestDefn("HashMap[Int, HashMap[Int, D]]", d => Matrix.asMatrix_Int_HashMap_Tuple_Like(d)),
    MatrixTestDefn("ArrayOfLocationsAndSize[D]", d => Matrix.asMatrix_Array_Like(d)),
    MatrixTestDefn("Array[Array[D]]", d => Matrix.asMatrix_Array_Array_Like(d)),
    MatrixTestDefn("JavaShortArrayAndLength", d => Matrix.MatrixJavaShortArrayAndLengthLike),
    MatrixTestDefn("JavaIntArrayAndLength", d => Matrix.MatrixJavaIntArrayAndLengthLike))

  for { d <- distanceDefns; m <- matrixDefns }
    s"Matrix[${d.prefix},${m.prefix}]" should { behave like matrix(d, m) }

}
{% endhighlight %}
There is a little horrible type casting in it, but my 'type-fu' isn't up to the task of working out how to do this without that. If any reader can
point out a better way I will be a happy person.

This almost worked but didn't. The last two definitions are different to the rest. Matrix.MatrixJavaShortArrayAndLengthLike and  Matrix.MatrixJavaIntArrayAndLengthLike are locked
down to only working with Shorts and Ints respectively.

I now have a choice between
{% highlight scala %}
  val distanceDefns = List(DistanceTestDefn("Int", _.toInt, Int.MaxValue), 
                           DistanceTestDefn("Short", _.toShort, Short.MaxValue))
  val matrixDefns = List(
    MatrixTestDefn("Map[(Int, Int), D]", d => Matrix.asMatrix_Map_Tuple_Like(d)),
    ....
    MatrixTestDefn("Array[Array[D]]", d => Matrix.asMatrix_Array_Array_Like(d)))

  for { d <- distanceDefns; m <- matrixDefns }
    s"Matrix[${d.prefix},${m.prefix}]" should { behave like matrix(d, m) }

  s"Matrix[Short,JavaShortArrayAndLength}]" should { 
          behave like matrix(distanceDefns(0), 
                 MatrixTestDefn("JavaShortArrayAndLength",
                                 d => Matrix.MatrixJavaShortArrayAndLengthLike)) }
  s"Matrix[Int,JavaIntArrayAndLength}]" should { 
         behave like matrix(distanceDefns(1), 
                 MatrixTestDefn("JavaIntArrayAndLength", 
                                 d => Matrix.MatrixJavaIntArrayAndLengthLike)) }
{% endhighlight %}

and
{% highlight scala %}
  val distanceDefns = List(DistanceTestDefn("Int", _.toInt, Int.MaxValue), DistanceTestDefn("Short", _.toShort, Short.MaxValue))
  val matrixDefns = List(
    MatrixTestDefn("Map[(Int, Int), D]", d => Matrix.asMatrix_Map_Tuple_Like(d)),
    ...
    MatrixTestDefn("Array[Array[D]]", d => Matrix.asMatrix_Array_Array_Like(d)),
    MatrixTestDefn("Java[D]ArrayAndLength", d => d match {
      case DistanceLikeForInt   => Matrix.MatrixJavaIntArrayAndLengthLike
      case DistanceLikeForShort => Matrix.MatrixJavaShortArrayAndLengthLike
    }))
{% endhighlight %}

As I want to be able to use this approach for the mileage factories, this second one is more attractive. I want to be able to do all the permutations of all the distances, the matrixes and the mileage factories

#Mileage factories


{% highlight scala %}
case class MileageTestDefn(prefix: String, factoryFn: (Distance[_], Matrix[_, _]) => MileageFactory[_, _])

class AllMileageTests extends WordSpec with Matchers {

  def mileageFactory[D, MM](distanceDefn: DistanceTestDefn[D], matrixDefn: MatrixTestDefn, mileageTestDefn: MileageTestDefn) {
    import distanceDefn.distanceLike
    import distanceDefn.distanceLike._
    implicit def toDo(x: Int) = distanceDefn.toDFn(x)

    val matrixLike = matrixDefn.matrixLike(distanceLike).asInstanceOf[Matrix[D, MM]]
    implicit def toMileage(t: (Int, Int, Int)): MileageEdge[D] = MileageEdge[D](t._1, t._2, t._3)(distanceLike)

    import matrixLike._

    val mf = mileageTestDefn.factoryFn(distanceLike, matrixLike).asInstanceOf[MileageFactory[D, MM]]

    s"${mileageTestDefn.prefix}[${distanceDefn.prefix}, ${matrixDefn.prefix}]" when {
      val mm = mf(3, List())
      "empty" should {
        "have a zero for same from/to" in {
          mm(0, 0) shouldBe zero
          ...
        }
        ... more tests
      }
    }
  }
  import CombinatorialTests._
  for { d <- distanceDefns; m <- matrixDefns; mf <- mileageFactoryDefns }
    s"${mf.prefix}[${d.prefix},${m.prefix}]" should { behave like mileageFactory(d, m, mf) }
}
 {% endhighlight %}

As you can see from the "import CombinatorialTests._" I moved the test definitions into a class so that I could share them:
{% highlight scala %}
object CombinatorialTests {
  val distanceDefns = List(DistanceTestDefn("Int", _.toInt, Int.MaxValue), DistanceTestDefn("Short", _.toShort, Short.MaxValue))
  val matrixDefns = List(
    MatrixTestDefn("Map[(Int, Int), D]", d => Matrix.asMatrix_Map_Tuple_Like(d)),
    MatrixTestDefn("HashMap[(Int, Int), D]", d => Matrix.asMatrix_HashMap_Tuple_Like(d)),
    MatrixTestDefn("Map[Int, Map[Int, D]]", d => Matrix.asMatrix_Int_Map_Map_Like(d)),
    MatrixTestDefn("HashMap[Int, HashMap[Int, D]]", d => Matrix.asMatrix_Int_HashMap_Tuple_Like(d)),
    MatrixTestDefn("ArrayOfLocationsAndSize[D]", d => Matrix.asMatrix_Array_Like(d)),
    MatrixTestDefn("Array[Array[D]]", d => Matrix.asMatrix_Array_Array_Like(d)),
    MatrixTestDefn("Java[D]ArrayAndLength", d => d match {
      case DistanceLikeForInt   => Matrix.MatrixJavaIntArrayAndLengthLike
      case DistanceLikeForShort => Matrix.MatrixJavaShortArrayAndLengthLike
    }))

  val mileageFactoryDefns = List(
    MileageTestDefn("MileageFactory1", (d, m) => new MileageFactory1[Any, Any]()(d.asInstanceOf[Distance[Any]], m.asInstanceOf[Matrix[Any, Any]])),
    MileageTestDefn("MileageFactory2", (d, m) => new MileageFactory2[Any, Any]()(d.asInstanceOf[Distance[Any]], m.asInstanceOf[Matrix[Any, Any]])),
    MileageTestDefn("MileageFactory3", (d, m) => new MileageFactory3[Any, Any]()(d.asInstanceOf[Distance[Any]], m.asInstanceOf[Matrix[Any, Any]])))
}
{% endhighlight %}
While I am wincing at the type casting in it, I don't know how to do this better, so I'll live with it for now. One of my hopes for doing this blog is that someone will say 'didn't you know you could do xxx' and I
can add a new tool to my toolbox.

#Summary
I had a situation where I had a load of things that can be wired together: Distances, Matrixes and MileageFactories. I had multiple implementations of each, and using the 'abstract class plus a single concrete implementation for each mixture I want to test' was
exploding combinatorially. This approach allows me to iterate over all the possible permutations. The configuration is far simpler: I need a test defn for the sum of the different combinations, rather than the product. i.e. Distances 2, Matrices 7, MileageFactory 3 means
12 test defns rather than 42. Every time I add another Distance or Matrix Representation or MileageFactory I only have to do one unit of work rather than many.

Another really nice thing is that when I add another test in mileage factory it will get automatically tested across every permutation.

It's not 'perfect' (The type casting costs me pain every time I look at it) but far better than what I had at the start. 
 