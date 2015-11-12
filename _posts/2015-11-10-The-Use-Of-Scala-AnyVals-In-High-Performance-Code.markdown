---
layout: post
title:  "The Use of Scala AnyVals in High Performance Code"
date:   2015-11-10 08:09:00 +0000
---
#Overview
I am engaged in a legacy modernisation task on a job that is working out the possible routes that people might of traveled on. The
result of this process is a list of 'Opportunities To Travel' or OTT. If we take a train ticket from Harrogate to Southampton and 
go to the internet and look up 'train times' we will get a result that will go via Leeds or York, and involve London Kings Cross and Waterloo. 
Each of the suggested options on the internet train site is an OTT. 

My observation is that all other things being equal, the memory consumed by a data set strongly determines how long it takes to manipulate it. By compressing the data
we get very real savings in performance and latency (if we care about latency: this project doesn't). In addition it reduces the amount of garbage to be collected, and
on a multi threaded application very often Garbage Collection can take 'most of the CPU time'. In modern CPU architectures 'memory is the new hard drive'. The same algorithm 
performed where a unit of data takes eight bytes will go a lot quicker than when a unit of data takes five hundred bytes.

## Quick stats

|Type|Expected number|
|:--- |---:|
|Locations that can be the source or destination| 2500|
|Location to Location mileages that need to be calculated| 6,250,000|
|Number of OTTs that need to calculated and stored in memory|25,000,000|

Having coded up an Algorithm that worked using traditional Scala data structures (linked lists, immutable maps), it took about a second to do all the OTTs for a single location/location pair.
Given there are over six million location/location pairs, that is going to take a long time to calculate: about a month of CPU time. From looking at the garbage collection, it was pretty clear
that the job couldn't easily be parallelised: even though the alogirithm was single threaded, all eight cores on my machine were busy most of the time. 

The initial data structure for an OTT looked like this:

{% highlight scala %}
case class OttRoute(miles: Int, journey: List[Either[DjsRecord, Walk]])
case class Walk(time: TimeInMinutes, destination: NationalLocationCode)
case class DjsRecord(departure: ServiceSegment, arrival: ServiceSegment, line: String)
trait ServiceSegment{/*this represents a train actually at a station. For example 
                     'York on Service#<id> departing at 09:00' */
{% endhighlight %}
NationalLocationCode was (in this case) actually a long. Let's do a quick memory calculation for an OTT with five steps. The DjsRecords are actually shared among many OttRoutes, so we won't 
worry about their memory usage. The Walks aren't but we'll ignore them for now. That leaves us with an object OttRecord, a list and an int. If the List is a standard Scala :: List then it 
has a :: object for every item in. The Either is another object

|Object|Count|Bytes Each|Total Bytes|
|OttRoute|1|40|40 bytes|
|List |5|40 |200 bytes|
|Either|5|48|240 bytes|

This gives a total of around 440 bytes. Perhaps 500 with a walk. Our goal is to reduce this to roughly 8 bytes, a sixty fold reduction, and the usage of AnyVals and TypeClasses are a big part of how we are going to do this. 
My assertion (which was confirmed in practice) is that this will  translate into a dramatic code speed up. With the 25 million OTTs (who are revisited several times by the alogithm) 500 bytes translates to 
around 12 gigabytes of memory, where as the 8 bytes becomes only 190 MB. All this memory has to go through the CPU several times during the OTT calculations.
 
#Type Classes to the rescue
The working algorithm manipulates an interrogates OTTs. Among its other needs the algorithm needs to ask for the start and end time of the OTT, and the 'from and to' locations (called NationalLocationCode)
{% highlight scala %}
trait ArrivalDeparture[T] {
  def startTime(t: T): TimeInHalfMinutes
  def endTime(t: T): TimeInHalfMinutes
  def fromNlc(t: T): NationalLocationCode
  def toNlc(t: T): NationalLocationCode
{% endhighlight %}
The nice thing about coding against the Type Classes is that I can change the implementation very easily. Another really nice thing is that this is the 'fly weight pattern' so called because
it has a very low memory footprint. The T's don't need to be objects, they can be primitive types. Primitive types tend to consume much less memory. JVM overheads tends to be 16 to 24 bytes per object. 
A primitive data type representing the time since midnight in half minutes can be stored in 2 bytes. Combined with AnyVals this approach gives us all the type safety we want, while still using the primitive types.

#AnyVals to the rescue
We want to talk about AnyVals though, not about Type Classes. 

In the ArrivalDeparture there are two other types referenced: TimeInHalfMinutes and NationalLocationCode. What are these? Remember that reducing memory usage is a very important goal, if not the primary goal, in 
this code. In that sort of world surely we want to be dealing with primitive types? Let's look at TimeInHalfMinutes

{% highlight scala %}
 class TimeInHalfMinutes(val time: Short) extends AnyVal with Comparable[TimeInHalfMinutes] {

  def +(otherTime: TimeInHalfMinutes) = TimeInHalfMinutes(time + otherTime.time)
  def -(otherTime: TimeInHalfMinutes) = TimeInHalfMinutes(time - otherTime.time)

  def <(otherTime: TimeInHalfMinutes) = time < otherTime.time
  def >(otherTime: TimeInHalfMinutes) = time > otherTime.time
  def <=(otherTime: TimeInHalfMinutes) = time <= otherTime.time
  def >=(otherTime: TimeInHalfMinutes) = time >= otherTime.time

  def addSeconds(otherTime: Int) = TimeInHalfMinutes(time + otherTime / 30)
  def add(otherTime: TimeInSeconds) = TimeInHalfMinutes(time + otherTime.seconds / 30)
  def addMinutes(otherTime: TimeInMinutes) = TimeInHalfMinutes(time + otherTime.minutes * 2)
  def addHalfMinutes(otherTime: TimeInHalfMinutes) = TimeInHalfMinutes(time + otherTime.time)
 
  def compareTo(otherTime: TimeInHalfMinutes) = time.compareTo(otherTime.time)

  override def toString = { // some code }
}
{% endhighlight %}
If we ignore the AnyVal this looks just like any other class definition. It has a companion object that we can use as normal. It is strongly typed. It allows us to safely interact with other 
representations of time that might not be type safe if we were using a Short to represent it. This is 'just good programming practice'. The only problem is that if it was an ordinary object it would consume 
many tens of bytes instead of the two bytes for a short.

However this is an AnyVal. The properties of an AnyVal is that it can only have one primitive instance variable. No Vals or Lazy Vals. And at runtime the class is removed and all that is left is the primitive.

This is where we say 'Wow'. We have most of the wins of a class: strong type safety, good method names, scope... And we have no performance or memory overhead. If we want to write code that goes quickly
yet is type safe, this is a powerful tool!

#A simpler AnyVal
Not all AnyVal's need to be complicated and have methods. The following gives a win just from the point of Type Safety. Instead of passing around an Int, I can now pass around a NationalLocationCode. I can 
put them into data structures, use them in method signatures, and the memory overhead (and hence performance) is the same as using primitives.

{% highlight scala %}
class NationalLocationCode(val nlc: Int) extends AnyVal {
  override def toString = s"NLC($nlc)"
}
{% endhighlight %}


#Not Damaging our Algorithmic code
The use of Type Classes means that our algorithms will treat the new data structure exactly as they did the old. There will be no change at all in that code in fact. Unfortunately
after speeding up by compressing data, the next target was the algorithmic code, so its readability got damaged a bit then.

#Compression of the List
The primary tool to help save the memory usage is the realisation that the list of Either[DjsRecord,Walk] can be shared. For example if I go from Harrogate=>York=>KingsCross that is a perfectly
valid OTT from Harrogate => Kings Cross. When I want an OTT from Harrogate => Southampton, I can share the (Harrogate=>York) and (York => Kings Cross) list. In practice this means that (as long
as the code constructs them this way) the OttRoute could be coded as
{% highlight scala %}
case class OttRoute(miles: Int, startOfJourney: Option[OttRoute], 
                    lastLeg: Either[DjsRecord, Walk])
{% endhighlight %}
Well this is a big reduction! Down to three objects: the OttRoute, the option and the Either. Perhaps this is down to 60 bytes. Nearly a whole order of magnitude. We still have most of an order of magnitude to go though.

#Roll your own lists
The first order of magnitude saving wasn't too hard. It took a couple of  days to wire the Type Class through instead of hard coding which type to use, then only minutes to write the type class for this 
new type of OttRoute.

The second one took a lot longer. The new data structure is going to be a single Long. The bits in the long are coded up as follows

|29 bits|A value|
|3 bits|A tag field describing what the value means|
|29 bits|A second value|
|3 bits|A tag field describing what the second value means|

The tag field can be:

|0|The value is a pointer to a ServiceSegment|
|1..5|The value is a list of length 1..5
|7|The value is the time for a walk and the destination, compressed into 29 bits|

This means that there is no longer a separation between Direct Journeys and OTTs. Each is represented at a Scala Long. There has to be a source of the OTTs that this Long represents, and in fact there 
are five: one for each step length. (Thus the tag field represents which array to look the referenced items up in)

This is just an alternative implementation of the standard :: list in Scala. The first 32 bits represent the data value in the list, and the second 32 bits point to the tail of the list. The major difference
is that each OTT is now consuming 8 bytes instead of the original 800 or the much improved 60. This has lost the mileage storage (which would roughly double memory consumption), and experiments showed it was
marginally quicker to recalculate it (by walking the list) than to store it. The 'Either' and the 'Option' are handled almost invisibly by the Tag fields.

It is worth pointing out that due to the Type Classes this data representation is invisible to the algorithmic code. The old data structures can be swapped in and out without any problems.

#Using AnyVals to make this compression readable

Step number one was to deal with the Value and Tag 'fields' inside the Long. Originally the Tag field was just the size, so the name should perhaps be changed to ValueAndTag
{% highlight scala %}
object ValueAndSize {
  def apply(value: Int, size: Int) = new ValueAndSize(value * 8 + (size & 7))
}
class ValueAndSize(val int: Int) extends AnyVal {
  def size = int & 7
  def value = int / 8
  override def toString = s"($value/$size)"
}
{% endhighlight %}
This is really nice. Given an Int (32 bits) wrapped in a ValueAndSize (which takes no time, and consumes no run time resources) we can get access to the size and value in what is idiomatically a nice way. 
Let's extend this idea to the actual Ott itself

{% highlight scala %}
object LongTools {
  val hiDivisor = (Int.MaxValue.toLong + 1) * 2
  val loMask = hiDivisor - 1
  def hi(l: Long): Int = (l / hiDivisor).toInt
  def lo(l: Long): Int = (l & loMask).toInt
  def compose(hi: Int, lo: Int) = hi * hiDivisor + lo
}
object OttAsLong {
  def dj(from: ServiceSegment, to: ServiceSegment) = 
     OttAsLong(from = ValueAndSize(from.id, 0), to = ValueAndSize(to.id, 0))
  def apply(to: ValueAndSize, from: ValueAndSize) = 
     new OttAsLong(LongTools.compose(to.int, from.int))
}
class OttAsLong(val ott: Long) extends AnyVal {
  def toPart = new ValueAndSize(LongTools.hi(ott)) 
  def toSize = toPart.size
  def toValue = toPart.value

  def fromPart = new ValueAndSize(LongTools.lo(ott)) 
  def fromSize = fromPart.size
  def fromValue = fromPart.value

  override def toString = s"OttAsLong($toPart <== $fromPart)"
}
{% endhighlight %}
 Well this is nice. Given an 'object' of type OttAsLong, it is easy to split it into the four data fields. The 'object' only consumes the same memory as a Long (8 bytes) and has useful helper methods
 
 Let's see how the Type Classes use this. (The following is simplified). We'll start with how to got from a ValueAndSize to arrival and departure data. ValueAndPartServiceSegmentsArrivalDepartureLike
 is a Type Class. The use of the AnyVal is in the segment protected method. It reads as cleanly as if ValueAndSize were a standard Scala class 
 
{% highlight scala %} 
 class ValueAndSizeArrivalDepartureLike(serviceSegments: Array[ServiceSegment]) 
       extends LocationArrivalDepartureLike[ValueAndSize] {
  def nlc(t: ValueAndSize) = segment(t).nlc
  def arrival(t: ValueAndSize) = segment(t).arrival
  def departure(t: ValueAndSize) = segment(t).departure
  
  protected def segment(t: ValueAndSize) = 
     if (t.size == 0) serviceSegments(t.value) 
     else throw new SegmentException(t.toString)
}
{% endhighlight %}

Part of the Type Class for the Ott behaviour is shown here. It is simplified, has error handling removed  and doesn't show the code for dealing with walks.  

{% highlight scala %}
trait RawLongOttLike with Ott[OttAsLong]{
  /** Given a ValueAndSize give me an Ott */
  def ottFn: (ValueAndSize) => OttAsLong     
  /** These are the raw train segments that are referenced if  Tag field is zero */ 
  def serviceSegments: Array[ServiceSegment] 
  
  def segmentLike = new ValueAndSizeArrivalDepartureLike(serviceSegments) 

  def fromNlc(t: Ott): NationalLocationCode = segmentLike.nlc(firstStep(t).fromPart) 
  def toNlc(t: Ott): NationalLocationCode = segmentLike.nlc(toDirectJourney(last))

  protected def toDirectJourney(ott: OttAsLong): ValueAndSize = 
    if (toSize == 0) ott.toPart  else  
       toDirectJourney(ottFn(ott.toPart))
  def firstStep(ott: OttAsLong): OttAsLong = 
    if (ott.fromSize == 0) ott else 
       firstStep(ottFn(ott.fromPart))

{% endhighlight %}


#Limitations
Don't put AnyVals into an Array. For obscure reasons (obscure to me anyway) this ends up requiring Scala to wrap the primitive in an object. Goodbye performance and efficiency! My solution to this
is code like the following. ShortArrayAndLength is a native Java class that simply consists of a short[] and a 'size'. 

{% highlight scala %}
class TimeInHalfMinutesArray(val maxSize: Int) extends ShortArrayAndLength(maxSize) {
  def apply(index: Int) = TimeInHalfMinutes(get(index))
  def update(index: Int, value: TimeInHalfMinutes) = set(index, value.time)
}
{% endhighlight %}

Another limitation is that AnyVals don't play well with inheritance. Don't try for code reuse through inheritance with them: it doesn't work. Scala has plenty of tools to deal with this though, so
I didn't find it a problem even with the TimeInMinutes, TimeInHalfMinutes, TimeInSeconds which could easily of been in an inheritance hierarchy.

#Big Wins from AnyVals
For me the biggest win was 'when things went wrong'. The AnyVal has a toString method that allows the compressed data inside it to be displayed. For TimeInHalfMinutes this
was very helpful as "09:30" is much easier to understand than 1140. With the OttAsLong that was absolutely invaluable. 

The type safety for all of these could nearly be met by Type Classes, but nearly isn't the same as actually. For example the AnyVals TimeInMinutes, 
TimeInHalfMinutes and TimeInSeconds are very similar and represented by an Int. The data in them makes a big difference, and my happiness
levels at having them 'protected' by being wrapped in a AnyVal was high. On at least three or four occasions I had the compiler save me from what could of been a very difficult to track
down bug. Perhaps all of these as Type Classes would of been possible but the wiring code would (I think) of not been worth the win.

#Performance Results
The combination of this and a few similar pieces of works took the time to do a 'from/to' calculation from about a second to 40 microseconds. A very significant saving: far more than the memory saving
would imply. A large part of that is the reduced need for garbage collection.  The original Batch Job was estimated to take about a month of CPU time (for obvious reasons I never ran it) with these 
improvements that time was reduced to about 300 seconds. Even more importantly it became possible to parallelise it (a bit: there is still a lot of garbage collection going on) making the final time 
somewhere around 90s on an eight core machine. 

#Summary
AnyVals in some ways give you some of the major wins of Type Classes (The flyweight pattern) without any plumbing issues. They give Type Safety in a really good way without any damage to the code base.
 
 