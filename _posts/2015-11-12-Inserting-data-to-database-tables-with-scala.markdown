---
layout: post
title: "Inserting data to database tables with Scala"
date:   2015-11-11 12:06:23 +0000
---
I have a file full of JSON data that is about two gigabyte long. I want to import it into a database tables There are actually three types of JSON in the file, and as would be
expected the schema is very rigid. Each line of the file is a block of JSON and represents a 'Service'.

#Goals
* I obviously want the process to be 'reasonably quick'. I don't have to worry too much about performance as I will only be doing this a few times a day. 
As long as the time is less then 'a few minutes' I will be happy
* I would like to minimise the load on the database.
* I don't want to use vast amounts of memory: loading the file into memory, then writing it all to the database means that I would need to use a lot of my computers RAM for it

#Getting Going
I started with the code base in my earlier post on 'Accessing text files from Scala'. 
Doing anything on large data sets takes ages. So I want a shorter version of the file to work with. This can be used by the tests as well. This file is called Cif.json. Let's 
start by opening the file and printing it to the screen. This is like 'getting my feet wet'. 

{% highlight scala %}
  def main(args: Array[String]): Unit = {
    Source.fromFile(file).getLines().foreach { println}
{% endhighlight %}


#Getting the JSON
Now I need to be able to use a JSON parser. Normally I would be working in a play project. As I am familiar with the Play JSON parser, I started by just adding the 
following to build.sbt

>enablePlugins(PlayScala)

As well as that I add (or in this case create the file) the following to Plugins.sbt in the project subdirectory

>addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.4.2")

#Filtering and Parsing the JSON
They are distinguisable by the fact that the line contains "JsonScheduleV1".
The following filters and parses the JSON

{% highlight scala %}
    val rawLines = Source.fromFile(file).getLines()
    val scheduleLines = rawLines.filter(_.contains("JsonScheduleV1")).
                                 map(Json.parse(_) \ "JsonScheduleV1")
{% endhighlight %}

#Extracting data from the JSON
Remember each line represents a service, and each service has multiple service segments in them. If I wanted to perform any complicated operations on them I would
probably load them into a domain model. Here though I am just going to put them into a database. Lists of objects as my domain object, as that is the model that
the JDBC database code is going to want to see. 

Let's start by just putting the services into the database. I need a table to put the data into
{% highlight sql %}
create table serviceForBlog(
  id integer primary key,
  uid char(6),
  train_category char(10),
  train_service_code   char(10),
  scheduleDayRuns char(7),
  schedule_start_date date,
  schedule_end_date date
 );
{% endhighlight %}
 And I need to extract the data from the JSON. 
{% highlight sql %}
    val dateFormatter = new SimpleDateFormat("yyyy-MM-dd")
    val rawLines = Source.fromFile(file).getLines()
    val scheduleLines = rawLines.filter(_.contains("JsonScheduleV1")).
                            map(Json.parse(_) \ "JsonScheduleV1")
    val scheduleData = scheduleLines.zipWithIndex.map {
      case (json, id) =>
        def s(j: JsLookupResult) = j.asOpt[String].getOrElse(null)
        def d(j: JsLookupResult) =new java.sql.Date(dateFormatter.parse(s(j)).getTime)
        val schedule = json \ "schedule_segment"
        val trainUid = s(json \ "CIF_train_uid")
        val trainCategory = s(schedule \ "CIF_train_category")
        val trainServiceCode = s(schedule \ "CIF_train_service_code")
        val scheduleDayRuns = s(json \ "schedule_days_runs")
        val scheduleStartDate = d(json \ "schedule_start_date")
        val scheduleEndDate = d(json \ "schedule_end_date")
        List(id, trainUid, trainCategory, trainServiceCode, scheduleDayRuns, 
             scheduleStartDate, scheduleEndDate)
    }
  }
{% endhighlight %}
This code is complicated by the fact that the strings may or may not be present in the JSON. If not it is to be represented by null in the database. The dates are in the format
that is defined by the dateFormatter.


#Connecting to the database
Firstly we need to get access to the database. I use the following two traits
{% highlight scala %}
case class DataSourceDefn(url: String, userName: String, password: String, 
                         classDriveName: String = "org.postgresql.Driver", 
                         maxConnections: Integer = -1)

trait UsingPostgres {
  implicit lazy val defn: DataSourceDefn = 
      DataSourceDefn(url = "jdbc:postgresql:orcats", 
                   userName = "some username", password = "some password")
}
trait UsingDatabase {
  implicit def defn: DataSourceDefn
  protected def createDataSource(implicit dsDefn: DataSourceDefn) = {
    val ds = new BasicDataSource()
    ds.setDriverClassName(dsDefn.classDriveName)
    ds.setUrl(dsDefn.url)
    ds.setUsername(dsDefn.userName)
    ds.setPassword(dsDefn.password)
    ds.setMaxActive(dsDefn.maxConnections)
    ds
  }
  protected def withConnection[X](fn: (Connection => X))(implicit ds: DataSource) = {
    val c = ds.getConnection
    try fn(c) finally c.close
  }
  protected def withPreparedStatement[X](sql: String, fn: (PreparedStatement) => X)(
                          implicit ds: DataSource) = withConnection { connection =>
    val statement = connection.prepareStatement(sql)
    try fn(statement) finally statement.close
  }
{% endhighlight %}
This is pretty straightforwards. The 'which datasource shall I use' is abstracted away and I have a way of getting hold of connections or prepared statements. I also need to
modify my build.sbt about now. This file uses both BasicDataSource from the Apache dbcp project, and the postgres JDBC driver

>libraryDependencies +=   "commons-dbcp" % "commons-dbcp" % "1.4"
> 
>libraryDependencies +=   "org.postgresql" % "postgresql" % "9.4-1201-jdbc41"
  
#Putting the data in the database
The easiest way to put the data in the database is to do it one row at a time. For each item in the list, we can execute a single SQL statement 
{% highlight sql %}
insert into Segement(id, uid, train_category, train_service_code,...etc) 
       values(?,?,?,?,...etc)
{% endhighlight %}
That could be done quite easily with code like this
{% highlight scala %}
  protected def dataToDatabase(tableName: String, columnNames: List[String], 
                         data: Iterator[List[Any]])(implicit ds: DataSource) = {
    val columnsWithCommas = columnNames.mkString(",")
    val questionMarks = columnNames.map(_ => "?").mkString(",")
    val sql = s"insert into $tableName ($columnsWithCommas) values ($questionMarks)"
    for ((list, lineNo) <- data.zipWithIndex)
      withPreparedStatement(sql, { implicit statement =>
        for ((value, index) <- list.zipWithIndex)
          statement.setObject(index + 1, value)
        statement.execute
      })
  }

{% endhighlight %}
Because I am using SQL, I need to do my security check list. Am I opening myself to a SQL Injection attack? I am using string manipulation for the sql statement generation, which is 
a Bad Thing.  In a production system I would expect the stored procedures to be managed separately. However in this case The text in those strings doesn't come from anyone but 'me', 
and because of how I am using it, I am reasonably confident this is safe. I would be nervous going live with this sort of code though: my concern is that the columnNames could
be a vector for attack.

Having decided this approach is alright for now, let's use it by adding this after the JSON filtering, parsing and extracting code

{% highlight scala %}
    val columnNames = List("id", "uid", "train_category", "train_service_code", 
                        "scheduleDayRuns", "schedule_start_date", "schedule_end_date")
    withDatasource { implicit dataSource =>
      dataToDatabase("serviceForBlog", columnNames, scheduleData)
    }
{% endhighlight %}

I ran it on the small file, tested it and felt pleased. Surely the job was almost done! I pointed this at my large file, set it running, went for coffee and when I came back it was
still running... It took about 300s at the end. A quick test showed me that 22s of that is reading the file, and the rest is database time.  

#CATS not RATS
The problem with this approach is that it runs very very slowly. The rough timings above indicated about two thousands row per second. With around 500,000 services to insert, and then 
another four million service segments, this is going to take a while. I can live with the (say) five minutes for the service, but the over an hour for the service segments isn't very attractive.

Why is it taking this amount of time? Well every line in scheduleData requires a TCP/IP round trip across a network, sql parsing, and all that for just one row of data. Databases love 'chunks at a time' 
much more than they do 'row at a time'. In otherwords always be thinking of CATs and not RATs. 

#Batch inserts
Batch inserts are not much harder to work with, and allow us to chunk up our data. 
{% highlight scala %}
  protected def batchDataToDatabase(tableName: String, columnNames: List[String], data: Iterator[List[Any]], chunkSize: Int = 10000)(implicit ds: DataSource) = {
    val columnsWithCommas = columnNames.mkString(",")
    val questionMarks = columnNames.map(_ => "?").mkString(",")
    val sql = s"insert into $tableName ($columnsWithCommas) values ($questionMarks)"
    withPreparedStatement(sql, implicit statement => {
      for (chunk <- data.sliding(chunkSize, chunkSize)) {
        for { list <- chunk } {
          for { (value, index) <- list.zipWithIndex }
            statement.setObject(index + 1, value)
          statement.addBatch()
        }
        statement.executeBatch()
      }
    })
  }
{% endhighlight %}
Instead of taking about 270s to do the database work, it took 15 seconds. That's around 33,000 a second instead of the 2,000 a second. A big improvement 

#Summary
This approach to reading a file of data from a file, and adding the contents to a data base showed the importance of being able to do a Chunk at A Time instead of a Row At a Time. The database time for the
same job went from 270 seconds to 15 seconds. The extra code is not particularly complicated. The only actual issue I have had in practice with this is diagnosing exceptions, which I might discuss in a future blog