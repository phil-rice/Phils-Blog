---
layout: post
title: "Real World Play - Sub Modules"
date:   2015-11-22 16:00:23 +0000
comments: True
categories:
- scala
- play
- refactoring
---

This follows the work 

* [started here](/scala/play/2015/11/16/getting-started-with-play.html)
* [tested here](/scala/play/testing/2015/11/22/getting-going-with-play-and-testing.html) 

The code at the start of this work can be found  [https://github.com/phil-rice/notary1](https://github.com/phil-rice/notary1), and the code at the 
end can be found here [https://github.com/phil-rice/notary2](https://github.com/phil-rice/notary2) 

It is very worthwhile at least [scanning this](https://www.playframework.com/documentation/2.4.x/SBTSubProjects) before reading this 

#Review

At the moment we have a (small) monolithic Scala Play application. There are two controllers: One that handles the index page and displays hello world, and another that does a very simple login
story. Both have tests.

#Goal
Most play applications end up too big to be one monolithic whole. This is going to show an example of splitting a project up. We are going to end up with the following modules

|root|This is the 'main application'. It will hold the configuration for the play server, and code about the application as a whole. For now it will also hold the integration tests|
|notary|This will hold the Hello World Controller, and the unit tests for it. This is the one that will 'mature' into being about the notary||
|users|This will hold the Login Controller, the model code associated with users and logging in, and the unit tests for the same. It is quite likely to split into two: the access control and things about the users|
|common|Common aspects of the website will go here. Things like templates shared by all projects|
|utilities|String and map manipulations are the usual suspects for code that will go here|

#Overview of Build

I quite like the approach that [I learned from Cake Solutions](http://www.cakesolutions.net/). In the project subdirectory of the main application we put a Dependancies.scala file which defines
all the libraryDependancies. It is starts looking like this

## Dependancies.scala in project directory

{% highlight scala %}
import sbt._
object Dependencies {
  // Versions
  val scalaVersionNo = "2.11.7"
  val scalaPlusPlayTestVersion = "1.4.0-M3"

  // Libraries
  val scalaPlusPlay = "org.scalatestplus" %% "play" % scalaPlusPlayTestVersion % Test

  //Repositories
  val playRepositories = Seq(
    "scalaz-bintray" at "http://dl.bintray.com/scalaz/releases")

  // Projects Dependencies
  val rootDependencies = Seq(scalaPlusPlay)
  val notaryDependancies =  Seq(scalaPlusPlay)
  val userDependencies =  Seq(scalaPlusPlay)
  val commoneDependencies = Seq(scalaPlusPlay)
  val utilitiesDependencies = Seq(scalaPlusPlay)
}
{% endhighlight %}
This is mostly empty at the moment. As each project grows this helps deal with the inevitable dependency mess as the project grows.
 The dependencies for each project are listed in the Seq called xxxDependencies, where xxxis the name of the module. 

##build.sbt in build directory

This again is a little bulky at this stage. Each of the modules has a lazy val defining where the module is. Root is the name for 'the whole thing', and has the
notation 'aggregate' which lists all the other projects. This means that if root is the default 'project' in sbt, when a command is given it is given to all the aggregated
modules. The 'dependsOn' notation says that the code defined in one module is used in another
 
{% highlight scala %}
import Dependencies._

lazy val commonSettings = Seq(
  version := "0.1.0",
  scalaVersion := scalaVersionNo,
  javacOptions ++= Seq("-source", "1.8", "-target", "1.8"),
  javaOptions ++= Seq("-Xmx4G", "-XX:+UseConcMarkSweepGC"),
  resolvers ++= playRepositories
)

lazy val root = (project in file(".")). 
                settings(commonSettings: _*).
                settings(
                   libraryDependencies ++= rootDependencies
                ).enablePlugins(PlayScala).
                dependsOn( user, notary, utilities).
                aggregate( user, notary, utilities)
				  
lazy val notary = (project in file("module/notary")).
                settings(commonSettings: _*).
                settings(
                   libraryDependencies ++= notaryDependencies
                ).enablePlugins(PlayScala).
                dependsOn(common, utilities)
				  
lazy val user = (project in file("module/user")).
                settings(commonSettings: _*).
                settings(
                   libraryDependencies ++= userDependencies
                ).enablePlugins(PlayScala).
                dependsOn(common, utilities)

lazy val common = (project in file("module/common")).
                settings(commonSettings: _*).
                settings(
                   libraryDependencies ++= commonDependencies
                ).enablePlugins(PlayScala).
                enablePlugins(PlayScala).
                dependsOn( utilities)

lazy val utilities = (project in file("module/utilities")).
                settings(commonSettings: _*).
                settings(
                   libraryDependencies ++= utilitiesDependencies
                )
{% endhighlight %}

#Testing

We have the new project structure. Even though we don't yet even have a modules subdirectory, we can type 'test' at the sbt console which runs our existing
tests against the monolithic structure. This checks that we have all the dependencies we need at the root level, and that the build.sbt/Dependancies.scala code compiles.

#Building the Eclipse project

I use eclipse, so I use the eclipse plugin. Typing eclipse actually makes eclipse projects that match this module structure. I'm going to make my life easy and delete the 
project from eclipse, and then import the new project structure from a git repository. Somewhat to my surprise this worked first time.

#Moving files around

##Notary module

The summary of what I am going to do is 'move the files to the notary that belong there, and sort out the routes'

* Copy routes to directory module/notary/conf and rename to hello.routes
* Move NotaryController to directory module/notary/app/controller/notary 
* Edit NotaryController so that it's package is controller.notary
* Edit notary.routes to only have the routes for the notary module
* Move the unit tests (I leave the integration tests in the root)
* Edit routes in the main /conf directory to no longer have the routes for the notary module, and to have a new line in it:

{% highlight scala %}
->   /   notary.Routes
{% endhighlight %}

One very nice thing about Play is that the compiler is our friend, making sure that the last two steps are done properly. Unlike a number of other frameworks
I've worked with, it's hard to leave 'no longer needed' lines in the routes file, as the compiler complains. The combination of the above gives me a new play 
application with a routes file no longer called 'routes' but called 'hello.routes'. 

## User module

This is very similar to the Notary module

The files I mess with are 

* Copy routes to directory module/user/conf and rename to user.routes
* Move LoginController to directory module/user/app/controller/user
* Edit LoginController so that it's package is controller.user
* Move my model file 'Access' to directory module/user/app/model
* Edit login.routes to only have the routes for the hello module
* Edit routes in the main /conf directory to no longer have the routes for the hello module, and to have a new line in it:

{% highlight scala %}
->   /   user.Routes
{% endhighlight %}
Again the tests don't even compile properly at the moment, but the application will run, and the login behaviour and index page work.

## Tests

The integration tests are left in the 'root' project. The tests for login are moved to the 'test' sub directory of the login module, and the tests for hello are moved
to the 'test' sub directory of the hello module. To test they still work, use the command 'test' in SBT.

# Assets

We now have working code, but only because the application is so simple. Most play modules are going to need icons, images and other resources. At the moment all three of our routes files
have the following
{% highlight scala %}
# Map static resources from the /public folder to the /assets URL path
GET   /assets/*file   controllers.Assets.versioned(path="/public", file: Asset)
{% endhighlight %}
This will 'sort of' work, as long as all the assets are in the /public of the main root module. It will work because the first time this line is encountered it will set this up,
and the rest will be ignored. This isn't what we want though: we want the assets to be stored in the module they are associated with.

The important points, are to have the controllers in a package named after the module, to have a different Assets controller for each project, and to have the module
route file have an assets that includes the "Get /assets/<module>". [I found this question very helpful](https://groups.google.com/forum/#!topic/play-framework/z3OKPdfyocM) 

This is how we make an assets controller
{% highlight scala %}
package controllers.xxx
import play.api.http.LazyHttpErrorHandler
object XxxAssets extends controllers.AssetsBuilder(LazyHttpErrorHandler)
{% endhighlight %}

And here is what we put in the xxx.routes 
{% highlight scala %}
GET   /xxx/assets/*file   controllers.xxx.xxxAssets.versioned(path="/public/lib/xxx", file: Asset)
{% endhighlight %}

So as an example hello.routes will end up looking like this

{% highlight scala %}
GET   /                     controllers.hello.NotaryController.index
GET   /hello/assets/*file   controllers.hello.HelloAssets.versioned(path="/public/lib/hello", file: Asset)
{% endhighlight %}

I remove the 'assets' link from the main routes file, so it just looks like this:
{% highlight scala %}
->   /   notary.Routes
->   /   user.Routes
{% endhighlight %}

# Making the tests work

Running 'test' in SBT exposes some minor compilation issues: classes being in the wrong package. Once fixed the tests pass. Note that the tests are not checking any of the assets, and
not even manual inspection will do that yet, as we don't have any yet! We'll remedy that next blog

# Review

I now have the following directory structure

| / |build.sbt is in here, along with chromedriver.exe, and LICENSE and README files|
| /test|The browser integration tests go here|
| /conf|routes and application.conf is here. The routes file only consists of includes|
| /project|Dependancies.scala and plugins.sbt|
| /modules|
| /modules/notary|
| /modules/notary/app/controller/notary|NotaryController is in here|
| /modules/notary/conf|notary.routes is in here|
| /modules/user|
| /modules/user/app/controller/user|LoginController is in here|
| /modules/user/conf|user.routes is in here|
| /modules/user/test|The unit tests for the Hello World goes here

Because the project is (extremely) simple, we haven't touched on a topic that would probably come in, in a real world application: Linking between the modules.
For example if the index page was html and had a hyperlink to the login page, we have a need to refer to a route defined in one application from another. We'll look
at how to do that in a later blog.

#Next steps

We need to add some views so that we actually have something that looks like a website rather than a RESTful interface. Among other things this should allow us to check that the assets wiring 
for the sub modules actually works! 

