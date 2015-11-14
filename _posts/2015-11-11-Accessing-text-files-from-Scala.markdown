---
layout: post
title: "Accessing text files from Scala"
date:   2015-11-11 12:06:23 +0000
comments: True
---
I have a file full of JSON data that is about two gigabyte long. I want to import it into a database table. This triggered me to think
a little about the best way of accessing text files from Scala. My needs are simple

* I obviously want the process to be 'reasonably quick'. I don't have to worry too much about performance as I will only be doing this a few times a day. 
As long as the time is less then 'a few minutes' I will be happy
* I would like to minimise the load on the database.
* I don't want to use vast amounts of memory: loading the file into memory, then writing it all to the database means that I would need to use a lot of my computers RAM for it

Importantly I don't need any of the 'go find line number x', or 'mark' and 'reset' functionality. In essence I want to be able to do Map/Reduce (or Map/FoldLeft) on the 
lines of the file. Quite often the result of the fold will be Unit as I will be doing things like 'sending the data to a service', 'printing it to a report' or 'sending it to a database'

[The code for this can be found here](https://github.com/phil-rice/HelloSpark)  

#Getting Going
As ever the first thing to do is to set up my IDE (Eclipse). To do this I create a directory until ~/git, copy a .gitIgnore and a build.sbt file from 'somewhere else', edit the
build.sbt and run 'sbt eclipse'.  The SBT file I am using is 

>version := "0.1.0" 
>
>scalaVersion := "2.11.7"
>

Let's start by opening a file and printing it to the screen. This is like 'getting my feet wet'.
# Loading the file 
With my Java background I immediately started playing with FileReaders. 
{% highlight scala %}
trait UsingFiles {
  def withFile(filename: String)(fn: String => Unit) {
    val fileReader = new FileReader(filename);
    try {
      val bufferedReader = new BufferedReader(fileReader);
      var line = bufferedReader.readLine
      while (line != null) {
        fn(line)
        line = bufferedReader.readLine
      }
    } finally { fileReader.close }
  }
}
{% endhighlight %}
And... wow... that's nasty code. Reads like C. Is there a more 'Scala' way of doing it?  
{% highlight scala %}
  Source.fromFile(filename).getLines.foreach(println)
{% endhighlight %}
Nice and simple. I don't even have to create my own trait to wrap it. But I have two things to worry about. Where are my try/finally's: am I leaking operating 
system resources? Just as importantly can I use map and foldLeft on getLines without the whole of the file being pulled in to memory? 

#Testing for resources
What happens if you don't explicitly call close on the Source?
{% highlight scala %}
object TestingSource {
  def file = "src/main/resources/SmallFile.txt"
  val chunkSize: Int = 100000
  def test(fns: (BufferedSource) => Unit*) = {
    for (i <- 1 to Int.MaxValue) {
      if (i % chunkSize == 0) println(i)
      fns.foreach { fn => fn(Source.fromFile(file)) }
    }
  }
  def main(args: Array[String]): Unit = {
     test(_.getLines, _.getLines().next(), _.iter, _.iter.next)
  }
} 
{% endhighlight %}
 When the program finished I decided that I didn't have to worry too much about them. Perhaps there is a leakage of file handles, but this gave me medium confidence. I did 
 look into the source code for Source, and couldn't see a try finally block which leaves me worried enough that I wouldn't use this in code I had to run over a long time period
 without further investigation. But for things that I only use occasionally it looks fine
 
# Using map and fold
  
I would like to be able to code like the following on large text files 

{% highlight scala %}
 def main(args: Array[String]): Unit = {
    Source.fromFile(file).getLines().map(_.length).foreach(println)
 }
 {% endhighlight %}
But... and it's a big but ... what is the lifecycle of the objects. Do all the lines get loaded into memory, then the map get called on each one, then the println get called?
Well we can find out: 
{% highlight scala %}
 def main(args: Array[String]): Unit = {
    var count = 0
    Source.fromFile(file).getLines().map { x => count += 1; count }.
       foreach(_ => println(count))
    Source.fromFile(file).getLines().foldLeft(0) { 
       (acc, x) => println((count, acc)); count+=1; acc + 1 }
 }
 {% endhighlight %}
Happily this printed out the numbers from 1 to the number of lines in the file, followed by some tuples in which both numbers changed. implying that the map and foldLeft 
are indeed called in a lazy fashion

#Summary
I don't need to use lots of Java style code with BufferedFileReaders. Unless it is long duration product code I can safely use the Source.fromFile without worrying about
resource leakage, and the map and fold functions work sufficiently lazily that I can use them with large files safely 

 
 