---
layout: post
title: "Unit Testing Scala Type Classes"
date:   2015-11-11 13:06:23 +0000
comments: True
---
I was checking my first few blogs to see if they 'work' when I cut and paste the code, and discovered to my horror that actually they didn't.
I had cut and paste the code out of 'working tested code', but hadn't checked the code samples properly. By check I mean 'have Unit Tests'. 

I haven't read much on this sort of testing: Testing of code with generics in it, so I thought I would write something about it

[The code in here can be found here](https://github.com/phil-rice/MileageForBlog)

#Test framework for Scala
The default test framework for Scala is Scala-Test as far as I can see. I have used it for a couple of years and like it quite a lot. I've been using JUnit since the 1990s and still return to it
for some jobs, but most new tests I write in Scala Test. The IDE integration is 'awkward' and keeps 'failing', meaning that I have to be happy to use it from SBT, but that's not actually a big burden.
The IDE integration is constantly improving, and I'm hoping that one year soon it will be as good as that for JUnit.

One of the things I dislike intensely in a project is bringing in an open source project that comes with a hundred other open source projects. This almost inevitably leads
to the problems, and I don't like those sort of problems. Both Scala-test and  JUnit are very lightweight and doesn't cause me problems. 

To use scala test I add the following to Build.sbt

>libraryDependencies +=   "org.scalatest" %% "scalatest" % "2.2.4" % "test"

#Simple Typeclass tests
Our distance class is really simple. Distance is
trait Distance[D] {
{% highlight scala %}
  def zero: D
  def large: D
  def random: D
  def add(d1: D, d2: D): D
  def lessThan(d1: D, d2: D): Boolean
  def makeArray(size: Int, value: D): Array[D] 
  def makeArrayArray(size: Int, value: D): Array[Array[D]]
}
{% endhighlight %}
And the implementation for Int is pretty simple

{% highlight scala %}
object Distance {
  implicit object DistanceLikeForInt extends Distance[Int] {
    def zero: Int = 0
    def large = Int.MaxValue
    def random = Random.nextInt(100000)
    def add(d1: Int, d2: Int) = d1 + d2
    def lessThan(d1: Int, d2: Int) = d1 < d2
    def makeArray(size: Int, value: Int) = Array.fill(size)(value)
    def makeArrayArray(size: Int, value: Int) = Array.fill(size)(Array.fill(size)(value))
  } 
{% endhighlight %}
  
Is it actually worth testing this? Well the answer is that actually there is a bug in the way the above code interacts with the matrix code. Arguably it's a bug in the above code. The bug is in the method add. 
It was only by the Unit tests of matrix that I found it, and then I had to add to the DistanceLike tests to make sure any future implements had it. I'll explain the bug later, just see if you
can see it now.

Anyway let's make the rough test framework. I am going to make an abstract test called 'DistanceTests' which will be parameterised by D and then make concrete classes for DistanceTests[Int] and DistanceTests[Short]. The reflection
in scala-test will sweep them up, so that's all the plumbing I will have to do

Let's look at this approach
{% highlight scala %}
abstract class DistanceTests[D: Distance] extends FlatSpec with Matchers {
  ... tests will go here
}

class DistanceIntTests extends DistanceTests[Int]
class DistanceShortTests extends DistanceTests[Short]
{% endhighlight %}

#Writing tests when you don't know the data type
Let's write our first few tests. The first few methods are zero/large/random and add. I am not interested in trying to test Random: it's hard and I think unlikely to give me many wins. My first stab at these tests is

{% highlight scala %}
 val distanceLike = implicitly[Distance[D]]
"Distance" should "implement zero and large" in {
   distanceLike.zero shouldBe zero
    distanceLike.large shouldBe large
  }

  it should "implment addition" in {
    distanceLike.add(one, two) shouldBe three
  }
  it should "implement less than" in {
    distanceLike.lessThan(one, two) shouldBe true
    distanceLike.lessThan(two, one) shouldBe false
    distanceLike.lessThan(one, one) shouldBe false
  }
  it should "implment addition" in {
    distanceLike.add(one, two) shouldBe three
  }
{% endhighlight %}
Well that reads well. But obviously doesn't compile. How do we code up the numbers zero, one, two etc... Scala implicits come to our rescue. These implicits that turn one type into another are a tool we will use a lot in testing
{% highlight scala %}
  implicit def toD(x: Int): D
  def zero: D = 0
  def one: D = 1
  def two: D = 2
  def three: D = 3
  def large: D
{% endhighlight %}
Well that's interesting. The implicit toD (not defined in DistanceLike[D]) will do the conversion of 0,1,2,3 to zero,one,two,three. This makes our two concrete classes slightly bigger. 

{% highlight scala %}
class DistanceIntTests extends DistanceTests[Int] { 
  implicit def toD(x: Int) = x; val large = Int.MaxValue 
}
class DistanceShortTests extends DistanceTests[Short] { 
   implicit def toD(x: Int) = x.toShort; val large = Short.MaxValue 
}
{% endhighlight %}

#Running the tests
 
In SBT I use the command '~test' quite a lot. This compiles and runs everything whenever the code base changes. The other things I do quite a lot is right click on the package on my IDE and 'run package as scala test'. Both work
 
 And these tests just pass. Something that makes me nervous, so I added a '1 shouldBe 2' to a couple of tests and reran then just to check that the tests were actually being executed!
 
#A few more tests
 We need to check that the Distance methods makeArray and makeArrayArray are implemented. Again these are easy
{% highlight scala %}   
   "Distance" should "implement makeArray" in {
    val array = distanceLike.makeArray(5, one)
    array.length shouldBe 5
    for (i <- 0 to 4) array(i) shouldBe one
  }
  
  it should "implement makeArrayArray" in {
    val array = distanceLike.makeArrayArray(5, one)
    array.length shouldBe 5
    for (i <- 0 to 4) {
      val a = array(i)
      a.length shouldBe 5
      for (j <- 0 to 4) a(j) shouldBe one
    }
  }
 {% endhighlight %}
 And these worked too.
 
 Job done. It didn't take long, and if I decide to implement Distance for Floating point numbers all I need to do is add the following and that TypeClass will be tests
{% highlight scala %}
class DistanceFloatTests extends DistanceTests[Float] { 
    implicit def toD(x: Int) = x.toFloat; 
    val large = /*whatever we put in the Distance[Float] Type Class */ 
}
{% endhighlight %}

#Matrix Like tests
I mostly used the same approach, and very quickly had this 
{% highlight scala %}
abstract class MatrixLikeTests[D: Distance, MM](
        implicit matrixLike: Matrix[D, MM]) extends WordSpec with Matchers {
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
    "put is called" should {
      "return the result" in {
        val m = makeRaw(5, three)
        val m2 = put(m, 0, 0, zero)
        val m3 = put(m2, 0, 1, one)
        val m4 = put(m3, 1, 0, two)
        get(m4, 0, 0) shouldBe zero
        get(m4, 0, 1) shouldBe one
        get(m4, 1, 0) shouldBe two
        for { i <- 2 to 4; j <- 2 to 4 } get(m4, i, j) shouldBe three
      }
    }
  }
}

abstract class MatrixLikeIntTests[MM](implicit matrixLike: Matrix[Int, MM]) 
               extends MatrixLikeTests[Int, MM] {
  implicit def toD(x: Int) = x;
  
  "A mileage matrix" when {
    "empty" should {
      "have a zero for same from/to" in {
        val mm = mf(3, List())
        mm(0, 0) shouldBe zero
        ...
      }
      ... more tests
   }
}
abstract class MatrixLikeShortTests[MM](implicit matrixLike: Matrix[Short, MM]) 
               extends MatrixLikeTests[Short, MM] {
  implicit def toD(x: Int) = x.toShort;
}

class MatrixLikeArrayArrayIntTests extends MatrixLikeIntTests[Array[Array[Int]]]
class MatrixLikeMapMapIntTests extends MatrixLikeIntTests[Map[Int, Map[Int, Int]]]
...
{% endhighlight %}
All pretty straight forwards and passed first time.

#Mileage Factory tests
I started this saying that I didn't have working mileage calculations. The mileage calculations are more complicated than distance. There is no doubt in 'shall we test this'. It's complicated and already
has a bug! Let's look at how the same approach would work
{% highlight scala %}
abstract class MileageTests[D: Distance, MM](implicit matrixLike: Matrix[D, MM]) 
               extends WordSpec with Matchers {
  val mf = new MileageFactory1[D, MM]
  
}
abstract class MileageIntTests[MM](implicit matrixLike: Matrix[Int, MM]) 
               extends MileageTests[Int, MM] {
  implicit def toD(x: Int) = x; 
}
abstract class MileageShortTests[MM](implicit matrixLike: Matrix[Short, MM]) 
               extends MileageTests[Short, MM] {
  implicit def toD(x: Int) = x.toShort; 
}

class MilageArrayArrayIntTests extends MileageIntTests[Array[Array[Int]]]
class MilageArrayArrayShortTests extends MileageShortTests[Array[Array[Short]]]

class MilageMapMapIntTests extends MileageIntTests[Map[Int, Map[Int, Int]]]
class MilageMapMapShortTests extends MileageShortTests[Map[Int, Map[Int, Short]]]

class MilageHashMapHashMapIntTests extends MileageIntTests[HashMap[Int, HashMap[Int, Int]]]
class MilageHashMapHashMapShortTests extends MileageShortTests[HashMap[Int, HashMap[Int, Short]]]
{% endhighlight %}
As you can see there is a lot of cut and paste at the bottom of that. Each permutation of distance and MM has to be hard coded. This is because of the way scala-test picks up tests to run: it likes concrete classes. With some work I suspect I
could come up with a strategy to avoid it, but actually I don't mind this. It doesn't take long.

#First Mileage Matrix tests
{% highlight scala %}
abstract class MileageTests[D: Distance, MM](implicit matrixLike: Matrix[D, MM]) extends WordSpec with Matchers {
  val mf = new MileageFactory1[D, MM]
  implicit def toMileage(t: (Int, Int, Int)): MileageEdge[D]
  implicit def toD(x: Int): D
  implicit val distanceLike = implicitly[Distance[D]]
  import distanceLike._

  "A mileage matrix" when {
    val mm = mf(3, List())
    "empty" should {
      "have a zero for same from/to" in {
        mm(0, 0) shouldBe zero
        mm(1, 1) shouldBe zero
        mm(2, 2) shouldBe zero
      }
      "have a large value for other mileages" in {
        for { i <- 0 to 2; j <- 0 to 2 if i != j } {
          mm(i, j).getClass shouldBe large.getClass
          mm(i, j) shouldBe large
          mm(j, i) shouldBe large
        }
      }
    }

    "all combinations given" should {
      "return 0 for form/to being the same" in {
        val mm = mf(3, List((0, 1, 10), (0, 2, 20), (1, 2, 25)))
        mm(0, 0) shouldBe 0
        mm(1, 1) shouldBe 0
        mm(2, 2) shouldBe 0
      }
      "return the stated combinations" in {
        val mm = mf(3, List((0, 1, 10), (0, 2, 20), (1, 2, 25)))
        mm(0, 1) shouldBe 10
        mm(0, 2) shouldBe 20
        mm(1, 2) shouldBe 25
      }
      "return the inverse combinations" in {
        val mm = mf(3, List((0, 1, 10), (0, 2, 20), (1, 2, 25)))
        mm(1, 0) shouldBe 10
        mm(2, 0) shouldBe 20
        mm(2, 1) shouldBe 25
      }
    }
    "some combinations not given" should {
      val mm = mf(3, List((0, 1, 10), (0, 2, 20)))
      "do the sum of links if longer" in {
        mm(2, 1) shouldBe 30
        mm(1, 2) shouldBe 30
      }
    }
  }
}

abstract class MileageIntTests[MM](implicit matrixLike: Matrix[Int, MM]) 
              extends MileageTests[Int, MM] {
  implicit def toD(x: Int) = x; 
  implicit def toMileage(t: (Int, Int, Int)) = MileageEdge(t._1, t._2, t._3)
}
abstract class MileageShortTests[MM](implicit matrixLike: Matrix[Short, MM]) 
               extends MileageTests[Short, MM] {
	implicit def toD(x: Int) = x.toShort; 
  implicit def toMileage(t: (Int, Int, Int)) = MileageEdge(t._1, t._2, t._3.toShort)
}

class MilageArrayArrayIntTests extends MileageIntTests[Array[Array[Int]]] 
class MilageMapMapShortTests extends MileageShortTests[Map[Int, Map[Int, Short]]]
{% endhighlight %}
And...the behaviour of these tests was just embarressing... -2147483648 was not equal to 2147483647. A warm red glow hit my cheeks and my forehead hit the table. The problem was that I was adding numbers to 'large' and then comparing them. Adding
anything to large made them overflow, and they became a negative number. In otherwards I had to do the following 

{% highlight scala %}
  "Distance" should "implment addition with large to always return large" in {
	  distanceLike.add(large, large) shouldBe large
	  distanceLike.add(one, large) shouldBe large
	  distanceLike.add(large, two) shouldBe large
  }
{% endhighlight %}
And when they failed I changed the implementation in Distance

{% highlight scala %}
trait Distance[D] {
  def add(d1: D, d2: D): D = if (d1== large|| d2==large) large else addRaw(d1, d2) 
  def addRaw(d1: D, d2: D): D
}

object Distance {
  implicit object DistanceLikeForInt extends Distance[Int] {
    def addRaw(d1: Int, d2: Int) = d1 + d2
  }
{% endhighlight %}

 
#Summary
The Type Classes were remarkably easy to write unit tests for. An abstract class for the behaviour, and a concrete class for each implementation. As ever implicits helped make the tests easy to write and just as easy to read
 