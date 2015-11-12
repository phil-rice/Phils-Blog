---
layout: post
title: "Getting going with Apache Spark"
date:   2015-11-12 19:06:23 +0000
---

#Overview
My last few blogs have been about reading a large amount of data from a JSON file and inserting it into a database. I thought I would a little introduction to Apache Spark
as a way of quickly adding parallelism to the import.

#Getting Going
I need to add the apache spark dependancies to my project in my build.sbt
>libraryDependencies += "org.apache.spark" %% "spark-core"%  "1.4.1"

#Hello World
This project is about files, so before going into the complexity of large files and the JSON parser, let's get our feet wet with a simple application. Let's
print a file with some line numbers.

{% highlight scala %}
object HelloSpark {
  def file = "src/main/scala/org/validoc/helloSpark/HelloSpark.scala"

  def main(args: Array[String]): Unit = {
    val lines = Source.fromFile(file).getLines().
      zipWithIndex.map { case (line, index) => f"$index%2d $line" }
    lines.foreach { println }
}
{% endhighlight %}

Pretty simple stuff. When we run it, we get the source code of the HelloSpark file with line numbers.

Let's see now how we can farm the work of formatting these lines across multiple processors. Clearly overkill. But it will allow us to 
see how the Apache Spark code works without worrying about the code

{% highlight scala %}
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("HelloSpark").setMaster("local")
    val sc = new SparkContext(conf)
    val lines = sc.textFile(file).
      zipWithIndex.map { case (line, index) => f"$index%2d $line" }
    lines.foreach { println }
  }
{% endhighlight %}
Let's look at this. First of all we create some configuration. The appName is 'HelloSpark', and the url of the Apache Spark cluster is 'local', meaning the local machine
We turn the configuration into a SparkContext called sc, and the rest of the code is pretty self explanatory. The really nice part about it is that it looks remarkably like the
original code. 

#Boom
I ran the program this 'Hello World' example, and somewhat to my horror it didn't work. I got 
{% highlight scala %}
Exception in thread "main" com.fasterxml.jackson.databind.JsonMappingException: Could not find creator property with name 'id' (in class org.apache.spark.rdd.RDDOperationScope)
 at [Source: {"id":"0","name":"textFile"}; line: 1, column: 1]
{% endhighlight %}
 
Some google-fu led me to http://stackoverflow.com/questions/31039367/spark-parallelize-could-not-find-creator-property-with-name-id which lead to the following 
lines for the build.sbt

>dependencyOverrides ++= Set(
>  "com.fasterxml.jackson.core" % "jackson-databind" % "2.4.4"
>)


If we run it now there is an awful lot of logging data, but our program is in fact printed to the console, in the middle of scary looking words like 'Hadoop' and 'BlockManager' and 'DagScheduler'. 

#Making this more readable
There was a lot of noise in that code. Let's make some traits that hides it
{% highlight scala %}
trait SparkOps {
  def sparkTitle: String
  def sparkUrl: String

  def withSpark[X](fn: SparkContext => X) = {
    val conf = new SparkConf().setAppName(sparkTitle).setMaster(sparkUrl)
    val sc = new SparkContext(conf)
    fn(sc)
  }

  def withFile[X](fileName: String)(fn: SparkContext => RDD[String] => X)(
                 implicit sc: SparkContext) = fn(sc)(sc.textFile(fileName))

  def withSparkAndFile[X](fileName: String)(fn: SparkContext => RDD[String] => X) =
    withSpark { implicit sc => withFile(fileName)(fn) }
}

trait LocalSparks extends SparkOps {
  def sparkUrl = "local"
  def sparkTitle = getClass.getSimpleName
}
{% endhighlight %}
I like this pattern for exploratory code. There are two concerns 'Doing things with Spark' and 'Configuring Spark with a url and a title'. This allows gash code
to be written quickly: extend 'LocalSparks', while proper code can inject the url and title through containers, environment variables, or whatever mechanism is needed. Even
JNDI if you are feeling masochistic .

This allows our main method to look like 
{% highlight scala %}
  def main(args: Array[String]): Unit = {
    withSparkAndFile(file) { sparkContext =>
      rawLines =>
        val lines = rawLines.zipWithIndex.map { case (line, index) => f"$index%2d $line" }
        lines.foreach { println }
    }
  }
{% endhighlight %}

#Moving onto large files with JSON
I already have my 'master slave import program'


##BOOM
I ran the program after adding these to check it still worked, and somewhat to my horror it didn't. I got 
Exception in thread "main" com.fasterxml.jackson.databind.JsonMappingException: Could not find creator property with name 'id' (in class org.apache.spark.rdd.RDDOperationScope)
 at [Source: {"id":"0","name":"textFile"}; line: 1, column: 1]
 
Fortunately some google-fu led me to http://stackoverflow.com/questions/31039367/spark-parallelize-could-not-find-creator-property-with-name-id which lead to the following 
lines for the build.sbt

>dependencyOverrides ++= Set(
>  "com.fasterxml.jackson.core" % "jackson-databind" % "2.4.4"
>)
And now the program works. Changing the file to point to the JSON file, worked as well 
{% highlight scala %}
  def main(args: Array[String]): Unit = {
    val jsonFile = "src/main/resources/Cif.json"
    withSparkAndFile(jsonFile) { sparkContext =>
      rawLines =>  rawLines.foreach { println }
    }
  }
{% endhighlight %}

#Parsing the code using JSON
I already have a program that reads in the file and inserts the data to the database. This looks like

{% highlight scala %}
  def main1(args: Array[String]): Unit = {
    val chunkSize = 50000
    val rawLines = Source.fromFile(file).getLines()
    val scheduleLines = rawLines.filter(_.contains("JsonScheduleV1")).map(Json.parse(_) \ "JsonScheduleV1")

    val scheduleData = scheduleLines.zipWithIndex.map { mapFn }

    val masterSlaveDefn = new MasterSlaveDefn(
      "service_for_blog",
      List("id", "uid", "train_category", "train_service_code", "scheduleDayRuns", "schedule_start_date", "schedule_end_date"),
      "service_segment_for_blob",
      List("service_id", "tiploc", "departure", "arrival"))

    withDatasource { implicit dataSource => masterSlaveInsert(masterSlaveDefn, scheduleData) }
  }
{% endhighlight %}
This is a couple minor refactors of the code I used in the last blogs. All the verbose JSON parsing I put into a function called mapFn. The importance bit of the code is the for loop.

#Moving to Spark
The first go at moving to spark requires us to think a little (a lot) above the life cycle of objects. Any thing that cannot be serialised can't be sent over the wires. So for example the database
connections have to be declared on each node. Fortunately a RDD has a useful method called 'forEachPartition' and we can put our database access code there. 
{% highlight scala %}
    val chunkSize = 50000
    withSparkAndFile(file) {
      implicit sc =>
        rawLines =>
          val scheduleLines = rawLines.filter(_.contains("JsonScheduleV1")).map(Json.parse(_) \ "JsonScheduleV1")

          val scheduleData = scheduleLines.zipWithIndex.map { mapFnLong }

          val masterSlaveDefn = new MasterSlaveDefn(
            "service_for_blog",
            List("id", "uid", "train_category", "train_service_code", "scheduleDayRuns", "schedule_start_date", "schedule_end_date"),
            "service_segment_for_blob",
            List("service_id", "tiploc", "departure", "arrival"))

          scheduleData.foreachPartition { chunk =>
            withDatasource { implicit ds => masterSlaveInsert(masterSlaveDefn, chunk, chunkSize) }
          }
    }
{% endhighlight %}
#Boom
So what happens when I run it? In the middle of a ton of debugging and logging information I get the following
{% highlight scala %}
java.lang.NoSuchMethodException: org.apache.hadoop.fs.FileSystem$Statistics.getThreadStatistics()
{% endhighlight %} 
I little google fu and I grimace as I read https://issues.apache.org/jira/browse/SPARK-5350 It sounds like I have dependancy issues. Even though the build.sbt and plugins.sbt is simple, I have about a hundred jars being used
by the application.
  
Some searching shows me that I am using Hadoop 2.2.0, play 2.4.2 and spark 1.4.1. A quick trip to the maven repository indicates to me that there is a new release of Spark that is built using Hadoop 2.2.0

>libraryDependencies += "org.apache.spark" % "spark-core_2.11" % "1.5.1"
>
>libraryDependencies += "org.apache.hadoop" % "hadoop-common" % "2.2.0"
The good old 'clean' no... really clean. Followed by a compile and a run gives 

{% highlight scala %}
java.lang.SecurityException: class "javax.servlet.ServletRegistration"'s signer information does not match signer information of other classes in the same package
{% endhighlight %} 
Probably the only thing I hate more than a dependancy issue is a security issue.  

Essentially I have over a hundred jars, some aren't working with each other. Most of them are needed because I wanted the Play JSON  library. That brought in web servers and all sorts of stuff. I decided to
simplify the build.sbt, and deleted the plugin.sbt. Which didn't help at all

Finally I found http://stackoverflow.com/questions/30299976/scala-with-spark-javax-servlet-servletregistrations-signer-information-does and followed the advice of changing where javax.servlet appears in the eclipse
export order. Grimacing a little I fix it: I suspect this will come back to bite me later. However it fixed the problem

#Next Boom
{% highlight scala %}
Exception in thread "main" org.apache.spark.SparkException: Task not serializable
Caused by: java.io.NotSerializableException: org.validoc.helloSpark.MasterSlaveDefn
{% endhighlight %} 
Easily fixed. I could either move the object so it isn't transfered over the wires to the core loop or just do the following 
{% highlight scala %}
class MasterSlaveDefn(val tableName1: String, val columnNames1: List[String],
                      val tableName2: String, val columnNames2: List[String]) extends Serializable
{% endhighlight %} 

#It works!
The data is imported. Only my CPU fan didn't switch on, the usage never got above 20%, the time taken was identical with being single threaded (four to five minutes). What's going on? My first thoughts are
'how many partitions is the data being split into', followed closely by 'am I hitting a database connection pool limit' and even worse 'is there some funny database locking going on'.

But no it was the 
{% highlight scala %}
trait LocalSparks extends SparkOps {
  def sparkUrl = "local"
  def sparkTitle = getClass.getSimpleName
}
{% endhighlight %} 
which should of been
{% highlight scala %}
trait LocalSparks extends SparkOps {
  def sparkUrl = "local[6]"
  def sparkTitle = getClass.getSimpleName
}
{% endhighlight %} 
With this I improved my import to 100s. At this point my performance monitors indicated that my hardrive was running at nearly 100%. My first approach: single threaded writing each 
ling JSON to the database with single insert statements took about an hour. The Batch inserts took that down to four or five minutes and this just about thirded it. To improve it further I
would probabaly have to play with reducing the amount of data being sent to the database: changing the format of the data so the (for example) the arrival and departure were two byte numbers
and the tiploc was a number that looked up into another table. I'm quite happy with the less than two minutes though

#Summary
It wasn't painless using Apache Spark. It never is. The pain over dependancies happens a lot when you use complicated projects that use a lot of open source projects. The serialization was simple in this example
but on occasion requires writing your own.  However there was little code change, and the speed up was certainly worth the small amount of effort that it took
with one insert

  
  