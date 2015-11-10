---
layout: post
title: "Scala Functional Builder for Cobol Copy Book Readers"
date:   2015-11-10 11:06:23 +0000
---
#Overview
I have to read some data from a mainframe system. The files are defined using ['cobol copy books'](http://www.tutorialspoint.com/cobol/cobol_data_layout.htm) 
The performance of this reading is not very important, and I am going to have a number of different files to read

##What's gone before
I was surprised to find very little in the way of conversion projects, other than commercial offerings. The ones that existed didn't seem to be maintained. The ones I looked
at and considered using where JRecord and CB2Java. A little playing with them, and I came to the conclusion that working out what was going on was going to be hard. I had actually
written a Cobol Copy Book to Java class convertor in the past, and knew that I needed to only do a small part of the work for these specific files.

What I wanted to do was to take the opportunity to learn how to use the same style of programming that Scala Play gives us with the JSON library. Here is the raw copy book description
for the data about a loction
{% highlight cobol %}
       01  STNS-REC.                                                    00002001
           05  LOC-LOC                  PIC 9(04).                      00003001
           05  LOC-ACODE                PIC X(03).                      00004001
           05  LOC-NAME                 PIC X(32).                      00005001
           05  LOC-COLOUR               PIC X(02).                      00006001
           05  LOC-ZONE                 PIC 9(03).                      00007001
           05  LOC-STATUS               PIC X(02).                      00010001
           05  LOC-REVENUE              PIC 9(05).                      00020001
           05  LOC-MIN-TIME             PIC S9(04) COMP.                00040001
           05  LOC-MAX-Time             PIC S9(04) COMP.                00050001
{% endhighlight %}

In the past I have done this with reflection. I would end up with a class like:
{% highlight scala %}
case class StationDetails1(@Num(4) stnLoc: Int,
                           @X(3) acode: String,
                           @X(32) name: String,
                           @X(2) colour: String,
                           @Num(3) zone: Int,
                           @X(2) status: String,
                           @Num(5) revenue: Int,
                           @C(4) minInterchange: Int,
                           @C(4) maxInterchange: Int)
{% endhighlight %}
This has a very big win that the cobol copy book definitions are very tightly linked to the field names. I've had a number of problems with it though: the major problem
is that just a little shuffle of position of a field can have impact on the code, and often I will want to do things like changing the types. If for example I actually wanted something
like the following as my domain object, it would be challenging to do with the annotation approach.   

{% highlight scala %}
class Location(val loc: Int) extends AnyVal
class Colour(val colour: String) extends AnyVal
class Zone(val zone: Int) extends AnyVal
class Status(val status: String) extends AnyVal
class Revenue(val revenue: String) extends AnyVal
class Time(val i: Int) extends AnyVal

case class StationDetails(stnLoc: Location, acode: String, name: String,
        colour: Colour, zone: Zone, status: Status, revenue: Revenue, 
        minInterchange: Time, maxInterchange: Time)
{% endhighlight %}

#Play Framework's JSON parser
This is a lovely piece of software engineering. If we were to use the same style of code we would end up with something like the following. Actually
I think it would be a little nicer, as they would manage to finesse the 'map' code away, but I'll look at how to do that in the future.

{% highlight scala %}
object StationDetails {
  import CopyBookReader._
  implicit val locReader = (Num(4)).map(new Location(_))
  implicit val colorReader = (X(2)).map( new Colour(_))
  implicit val zoneReader = (Num(4)).map(new Zone(_))
  implicit val statusReader = (X(2)).map(new Status(_))
  implicit val revenueReader = (Num(4)).map(new Revenue(_))
  implicit val timeReader = (C(4)).map(new Time(_))
  
  implicit val stationDetailsReader = (
    locReader and X(3) and X(32) and
    colorReader and zoneReader and statusReader and
    revenueReader and timeReader and timeReader)(StationDetails.apply _)
}
{% endhighlight %}
What we see in this code is the stationDetailsReader is using other readers to get data which it then uses to create a StationDetails. We have a lot of type safety here: the 
locReader produces things of type Location, the colorReader things of type Colour and so on. It is the goal of this piece of work: to make something like that. I would argue 
it isn't (quite) as readable as the annotations, but the flexibility and ability to calculate intermediate values, enrich values and make things type safe is well worth it.

#How does Play do this?
The answer is by heavy use of implicits and type classes. The most important trait is the FunctionalCanBuild[M[_]] trait. I have to admit to having spent a number of hours 
scratching my head and working on code before I "got it".  Let's look at how we can use it. The answer is 'really easily'. I have to tell the framework how to compose objects
together, and that's about it.

#The simple CopyBookReaders
Firstly we need a thing that is going to be the basic building block we are going to build. For the moment we don't need to worry what the CopyBookStream is, other than the fact that a CopyBookReader[X] will 
take one and produce a result of type X. In bad old mutable programming style the stream is in fact a mutable stream... But we will ignore that too! The methods on this are
unimportant for the framework until we come to the one place where we worry about how to wire things together.
{% highlight scala %}
trait CopyBookReader[X] {
  def apply(implicit copyBookStream: CopyBookStream): X
  def map[Y](f: X => Y) = FunctorCopyBookReader(this, f)
}
{% endhighlight %}
The FunctorCopyBookReader is explained below. For now I need a way to define the X / Num and C notation. These are the simple CopyBookReaders that

{% highlight scala %}
object CopyBookReader {
  def Num(length: Int): CopyBookReader =new NumCopyBookReader(length)
  def C(length: Int): CopyBookReader = new CompCopyBookReader(length)
  def X(length: Int): CopyBookReader = new StringCopyBookReader(length)
}
{% endhighlight %}
The NumCopyBookReader / CompCopyBookReader and StringCopyBookReader are the workhorses that actually takes bytes from the CopyBookStream
and turn them into values. 

{% highlight scala %}
trait FixedLengthCopyBookReader[X] extends CopyBookReader[X] {
  val buffer = new Array[Byte](length)
  val default: X
  def apply(implicit copyBookStream: CopyBookStream): X = {
    val bytes = copyBookStream.read(buffer)
    if (copyBookStream.hasMoreData)
      reads(buffer)
    else
      default
  }
  def reads(bytes: Array[Byte]): X
  def length: Int
}

case class NumCopyBookReader[X](length: Int) extends FixedLengthCopyBookReader[Int] {
  val default = 0
  def reads(bytes: Array[Byte]) = {
    val result = bytes.foldLeft(0)((acc, v) =>
      acc * 10 + (v & 0x0f))
    result
  }
}
case class StringCopyBookReader[X](val length: Int, trimIt: Boolean = true) 
             extends FixedLengthCopyBookReader[String] {
  val default = ""
  def reads(bytes: Array[Byte]) = {
    val result = new String(bytes,  "cp1047")
    if (trimIt) result.trim else result
  }
}
case class CompCopyBookReader[X](compLength: Int) 
               extends FixedLengthCopyBookReader[Int] {
  lazy val length = compLength / 2
  if (length % 2 != 0) throw new RuntimeException("Not supported")
  val default = 0
  def reads(bytes: Array[Byte]) = {
    val result = bytes.foldLeft(0)((acc, v) => acc * 256 + (v & 0xff))
    result
  }
}
{% endhighlight %}

#Composing the CopyBookReaders
Now I have a thing I want to build, I have to tell the Functional building framework how to build it. This will let us use the lovely 'and' notation
that we saw in our earlier code samples.

{% highlight scala %}
object CopyBookReader {
  implicit object FuncBuilder extends FunctionalCanBuild[CopyBookReader] {
    import play.api.libs.functional.~
    import scala.language.higherKinds
    def apply[A, B](ma: CopyBookReader[A], mb: CopyBookReader[B]) = 
      new CopyBookReader[A ~ B] {
	      def apply(implicit stream: CopyBookStream) = 
	         new ~(ma.apply(stream), mb.apply(stream))
	    }
  }
}
{% endhighlight %}

and... that's it. This is the one place that tells us how to wire things together. The Play framework does the rest of the heavy lifting. The code in the apply method is a 
little odd with class names like '~', but all it actually does is say if that if you have two copy readers, then call the first before calling the second.

Let's look at how to use the CopyBookReaders. 
{% highlight scala %}
  def foldLeft[Acc, X: CopyBookReader](file: String)(initial: Acc)(
                                       foldFn: (Acc, X) => Acc): Acc = {
    implicit val reader=implicitly[CopyBookReader[X]]
    val inputStream = new FileInputStream(file)
    var acc = initial
    try {
      implicit val stream = new CopyBookStream(inputStream)
      while (stream.hasMoreData) {
        stream.firstRead = true
        val next = reader.apply
        acc = foldFn(acc, next)
      }
      acc
    } finally { inputStream.close }
  }
 def list[X:CopyBookReader](file: String) = 
         foldLeft[List[X], X](file)(List[X]())((acc, x) => x :: acc)
 def print[X: CopyBookReader](file: String) = 
         foldLeft[Int, X](file)(0) { (acc, v) => println(v); acc + 1 }
{% endhighlight %}
This is implementing foldLeft across a file. The items in the file are all of type X, and the accumulator can be any type we want. The only magical plumbing is in the X:CopyBookReader which
makes sure that a variable of CopyBookReader[X] is in scope and passed around. Here we see a main method that prints all the stations details contained in a file

{% highlight scala %}
def main(args: Array[String]): Unit = {
  CopyBookReader.print[StationDetails]("some file name")
}
{% endhighlight %}

#The FunctorCopyBookReader
This is how a Int is turned into Location, or a String into a Colour. This turned out to be very straightforwards:

{% highlight scala %}
case class FunctorCopyBookReader[X, Y](reader: CopyBookReader[X], mapFn: X => Y) 
                                       extends CopyBookReader[Y] {
  def apply(implicit copyBookStream: CopyBookStream) = {
    mapFn(reader.apply(copyBookStream))
  }
}
object CopyBookReader {
  implicit object FunctorCopyReader extends Functor[CopyBookReader] {
    def fmap[A, B](m: CopyBookReader[A], f: A => B) = 
         new FunctorCopyBookReader[A, B](m, f)
  }
   ...
}
{% endhighlight %}

#The primitive readers
The following shows how the Num copy book reader is implemented. The other primitives are very similar

{% highlight scala %}
trait FixedLengthCopyBookReader[X] extends CopyBookReader[X] {
  val buffer = new Array[Byte](length)
  val default: X
  def apply(implicit copyBookStream: CopyBookStream): X = {
    val bytes = copyBookStream.read(buffer)
    if (copyBookStream.hasMoreData)
      reads(buffer)
    else
      default
  }
  def reads(bytes: Array[Byte]): X
  def length: Int
}
case class NumCopyBookReader[X](length: Int) extends FixedLengthCopyBookReader[Int] {
  val default = 0
  def reads(bytes: Array[Byte]) = {
    val result = bytes.foldLeft(0)((acc, v) =>
      acc * 10 + (v & 0x0f))
    result
  }
}
{% endhighlight %}

#The CopyBookStream
This is quite horrific with the use of flags and immutable state. The coupling between it and the foldLeft function is distasteful too.  I would be interested if 
anyone has a better way of doing this sort of thing.
{% highlight scala %}
class CopyBookStream(inputStream: InputStream) {
  // The problem these flags are solving is 'how do I know I am at the end of file' 
  var firstRead: Boolean = true
  var hasMoreData: Boolean = true
  def read(bytes: Array[Byte]) = { 
    if (hasMoreData) {
      val l = inputStream.read(bytes)
      if (l == -1) {
        if (firstRead)
          hasMoreData = false
        else throw new EndOfStreamException
      }
    }
  }
}
{% endhighlight %}

#Summary
I expected it to be very hard to do this work, and in the end (to be fair after hours of head scratching) it turned out to be the following

* Define the trait CopyBookReader
* Create the primitive CopyBookReaders 'read a string of length X', 'read a number of length x' and the CopyBookStream 
* Define how to compose two CopyBookReaders
* and that was mostly it

The work was actually a pleasure to do, and is very easy to use. I've used it now for half a dozen types of Cobol files. Each time I do it
I've had to add a new CopyBookReader. For example I had to add a ListCopyBookReader, where there is a value that defines how many times another field is used. It
was remarkable easy to write and worked first time.
 
{% highlight scala %}
case class ListMicroParser[X](repeatLength: Int, listReader: CopyBookReader[X])
                              extends CopyBookReader[List[X]] {
  def repeatCountReader = CompMicroParser(repeatLength)
  def apply(implicit copyBookStream: CopyBookStream) = {
    val count = repeatCountReader.apply
    (1 to count).map(i => listReader.apply).toList
  }
}
{% endhighlight %}




