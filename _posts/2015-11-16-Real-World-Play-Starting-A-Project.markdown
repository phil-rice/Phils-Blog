---
layout: post
title: "Real World Play - Starting A Project"
date:   2015-11-16 12:00:23 +0000
comments: True
categories:
- scala
- play
---

There are a number of 'getting going' tutorials on Play. They are good. For example this one [On the Scala Play website about a Todo list](https://www.playframework.com/documentation/2.0.4/ScalaTodoList) takes you 
happily through getting going, all the way to deploying on Heroku.  If you haven't done a tutorial like that, I suspect that most of this will not make a lot of sense. I strongly recommend you go through that first 

The examples aren't very 'real though'. All the Play applications I've worked on have had several play modules, and several non play modules for business logic. I have Akka actors that do activities behind the scene
and these have to coordinate with the website. I have to put security on the application to allow different access to different parts of the website. I have to meter my database access so that I can
see which database calls are being made, what sort of query plans the database is using.  

I'd like to show how to make that sort of play application.  

The code for this can be found here https://github.com/phil-rice/Notary with the tag 'Blog1'


#Digital notary
The example website I'll make will still be simple. It allows a file to be uploaded, and the application will simply tell us if it's seen that file before. This is the very simplest digital notary it is possible to 
make. The business drivers for the user are 'proving I've seen this document before'. So for example an email with an attachment can have the digest of that attachment added to it, and have that registered with a
third party. The business case isn't very strong at this level, but neither is the 'todo list'!

#Rough overview
I am very focused on security in most of my applications. The whole Data Protection issue is getting more and more important. I have found in the past that adding security in at the start is actually quite easy,
while adding it in after wards can be challenging, frustrating and error prone. This is rather like the idea of Testable code. If you write the tests as you go, then you write testable code. If you write the security
as you go, you end up with securable code.

|Set up my environment|Get the machine I am working on ready for work on this project|
|Hello World|I get the simplest play application possible working|
|Split into multiple modules|I make this split into a main application, and then a single sub play application: one for each business area, and a single business logic application|
|Add in a users and groups story|Security is very important and has to be built in at the beginning. A really rough users and groups allows me to...|
|Add in a simple security story|Every single web access should go through the security story. As developers are working on the website, they have to ensure that the security story is dealt with|
|Do the minimal viable product|Something that will do something of value for the users|

There are a number of things that have to be done as well. Typically there will be an email story: notifying users of events. If we have an email story we need to consider spammers (more security). Typically
there will be a database. I like securing my database as well, while not damaging my ability to implement code quickly. 

#Setting up my environment

##Encrypted Drive
I tend to keep all my development on a Veracrypt encrypted drive. My threat model is simple: I take my laptop with me whereever I go. There is a reasonable chance that it will stolen (two have been stolen in the
last ten years) and I often have some confidential data associated with files. Even if it's as simple as my sonatype login details for pushing to Maven. The encrypted drive seems to give little
or no overheads and removes a whole category of worry. Even with a fully encrypted drive I would use the Veracrypt local drives as I can enable and disable whether I have access to project on a piecemeal basis. Veracrypt
is very straightforwards to download and use. [You can find it here](https://veracrypt.codeplex.com/)

##Setting up play activator
Upgrading from play versions is, I have found, not a trivial thing. I've had to move my primary website from 2.3 to 2.4 and it was rather painful. As 2.5 is now out, I am going to start using that. Making
a new play application isn't something I do everyday. I've done it maybe a dozen times, all with different versions of Play, and every time I go back to the basics.
[The scala documentation is here](https://www.playframework.com/documentation/2.4.x/Installing) I tend not to use the activator at all once I've got HelloWorld working, but it's good for this initial phase

* [Download the activator from here](https://typesafe.com/get-started)
* Extract it onto my encrypted drive
* Launch a terminal, navigate to the activator and and execute 'activator ui'

##Setting up Hello World
I am going to be using a version control system for my files. So I make a /git subdirectory on the encrypted drive, and check that I have git installed. [If not  get it from here](https://git-scm.com/downloads).


In order to get the activator to make me the 'scaffolding', I need to have to have it running (activator ui) and go to http://localhost:8888 on my browser. The main thing you can do 
with the activator at the start is to create a new project from a template. I  navigated to the 'play scala intro', as that look to be a template that made a simple play application. 
The activator aked  me where to put it, and I selected mygit directory. There are rumours that you can do this initial stage all without the activator, but this is so fast, and I do the 
job so rarely, that I've not felt the need to mess around. I often think I have the skill to do it myself, but then I get bitten by 'version number hell'.  At the end of the day none of the things that have 
been created will be kept unmodified, and we need to understand them all. This is just a bit of scaffolding to get me going.

Being curious I go look at the new application in the file explorer. It put it under 'play-scala-intro' under /x/git, so I rename it to /x/git/notary, where x is my encrypted drive.

##Seeing if it works
I could of seen if it worked in activator, but instead I'll use the command line and make a load of changed. I navigated to the /x/git/notary subdirectory to see what's there. And... there are a lot of files!
 The first files I am interested in now are the build.sbt and  app directory  

##Simplifying the build
When it first came out the book [The Pragmatic Programmer](http://www.amazon.co.uk/The-Pragmatic-Programmer-Andrew-Hunt/dp/020161622X) made a great impression on me. If you haven't read it, I recommend stopping reading
this, buying it from Amazon and spending the next couple of weeks reading it several times. One of the best words of advice in it is 'Beware Evil Wizards'. I've just used a Wizard to do a ton of work, but
this is my application. I have to understand every single file in it. I don't want anything in there that I don't understand. It did do a ton of work for me, but I want to get rid of the extra things
it did that I don't need and don't want.

###build.sbt
Let's look at build.sbt, and it's dependent subdirectory 'project'. build.sbt is pretty simple. I edit the name
{% highlight scala %}
name := """play-scala-intro"""

version := "1.0-SNAPSHOT"

lazy val root = (project in file(".")).enablePlugins(PlayScala)

scalaVersion := "2.11.6"

resolvers += "Scalaz Bintray Repo" at "https://dl.bintray.com/scalaz/releases"

libraryDependencies ++= Seq(
  "com.typesafe.play" %% "play-slick" % "1.1.0",
  "com.typesafe.play" %% "play-slick-evolutions" % "1.1.0",
  "com.h2database" % "h2" % "1.4.177",
  specs2 % Test
)     

// Play provides two styles of routers, one expects its actions to be injected, the
// other, legacy style, accesses its actions statically.
routesGenerator := InjectedRoutesGenerator

fork in run := true
{% endhighlight %}
Well there are some things in here that I don't like. I'm not wanting to use Scalaz at the moment, and I won't be linking to databases any time soon. I wince when I read the comments on 
the routers: I'm used to the legacy style so I have a new learning to do. I suspect that the new way is actually better, but I still have to learn how to use it. I'm not wanting any of those
other libraries at the moment, other than perhaps the specs2 % test. The 'fork in run' looks unique to activator, and I've never needed it before,  so I edit the file to read.

{% highlight scala %}
name := "notary"

version := "1.0-SNAPSHOT"

lazy val root = (project in file(".")).enablePlugins(PlayScala)

scalaVersion := "2.11.6"

resolvers += "Scalaz Bintray Repo" at "https://dl.bintray.com/scalaz/releases"  //only added this line at the end. You can add it now

libraryDependencies ++= Seq(
  specs2 % Test
)     
{% endhighlight %}

It turned out that at the end I have to restore the resolvers statement to get Play to deploy successfully on Heroku.

Looking in the project subdirectory, I see a build.properties file and check its contents. It sets the sbt version, which I am expecting and happy with, and adds some other 'template uuid', which I don't need. 
So it becomes
{% highlight scala %}
sbt.version=0.13.8
{% endhighlight %}

### plugins.sbt

plugins.sbt is in the project subdirectory. Among it's other responsibilities it is how we use the play plugins that were enabled in the build. It looks like this
{% highlight scala %}
// The Play plugin
addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.4.3")

// web plugins
addSbtPlugin("com.typesafe.sbt" % "sbt-coffeescript" % "1.0.0")
addSbtPlugin("com.typesafe.sbt" % "sbt-less" % "1.0.6")
addSbtPlugin("com.typesafe.sbt" % "sbt-jshint" % "1.0.3")
addSbtPlugin("com.typesafe.sbt" % "sbt-rjs" % "1.0.7")
addSbtPlugin("com.typesafe.sbt" % "sbt-digest" % "1.1.0")
addSbtPlugin("com.typesafe.sbt" % "sbt-mocha" % "1.1.0")
{% endhighlight %}
As I don't need coffeescript, less or any of the others just yet, I comment them all out, except the "com.typesafe.play" % "sbt-plugin". I'll probably bring some of the back later, but I'll bring them back 
when I need them. I like knowing what I have, and I don't recognise half of these.

### sbt-ui.sbt

I laughed when I read this and deleted the file.

{% highlight scala %}
// This plugin represents functionality that is to be added to sbt in the future
addSbtPlugin("org.scala-sbt" % "sbt-core-next" % "0.1.1")
{% endhighlight %}
### play-fork-run.sbt

This is another file that gives me a nice warm feeling as I delete it
{% highlight scala %}
// This plugin adds forked run capabilities to Play projects which is needed for Activator.
addSbtPlugin("com.typesafe.play" % "sbt-fork-run-plugin" % "2.4.3")
{% endhighlight %}

## The App subdirectory

As we know a lot of code has been created for us under the app directory. Pretty much I don't want any of it. I delete lots of directories and files and end up with this

/app/
  /assets
     /stylesheet (empty directory)
     /javascript (empty directory)
  /controllers
      NotaryController.scala (I renamed PersonController.scala)
      
### NotaryController
Now I need to look at NotaryController.
{% highlight scala %}
package controllers

import play.api._
... some more imports
import scala.concurrent.{ ExecutionContext, Future }
import javax.inject._

class PersonController @Inject() (repo: PersonRepository, val messagesApi: MessagesApi)
                                 (implicit ec: ExecutionContext) extends Controller with I18nSupport{

  /** The mapping for the person form.  */
  val personForm: Form[CreatePersonForm] = Form {
    mapping(
      "name" -> nonEmptyText,
      "age" -> number.verifying(min(0), max(140))
    )(CreatePersonForm.apply)(CreatePersonForm.unapply)
  }

  /** The index action.  */
  def index = Action {
    Ok(views.html.index(personForm))
  }
  ...
{% endhighlight %}

Well some bits are fine. PersonRepository was obviously the link to the model. [I like the messagesApi](https://www.playframework.com/documentation/2.4.x/ScalaI18N). I'm almost certainly going
to be using execution contexts... but I want a simple start. So I edit it to
{% highlight scala %}
package controllers
import play.api._
import play.api.mvc._

object NotaryController extends Controller{
  def index = Action {Ok("Hello World") }
}
{% endhighlight %}

##The conf subdirectory

I change this to

/conf
  /evolutions
     /default (empty directory)
  application.conf
  logback.xml (left alone)
  message (empty file)
  routes
    
##Routes

I edit the routes file to
{% highlight scala %}
# Routes
# This file defines all application routes (Higher priority routes first)
# ~~~~

# Home page
GET     /                           controllers.NotaryController.index

# Map static resources from the /public folder to the /assets URL path
GET     /assets/*file               controllers.Assets.versioned(path="/public", file: Asset)
{% endhighlight %}
This a little different from what I was expecting. The 'Assets.versioned' syntax is new to me, so a quite bit of google-fu leads me to a 
[description of assets on the play framework website](https://www.playframework.com/documentation/2.4.x/Assets) Under the section marked  
"Reverse routing and fingerprinting for public assets" there is an interesting tidbit

>sbt-web brings the notion of a highly configurable asset pipeline to Play e.g. in your build file:
>
>pipelineStages := Seq(rjs, digest, gzip)
>The above will order the RequireJs optimizer (sbt-rjs), the digester (sbt-digest) and then compression (sbt-gzip). Unlike many sbt tasks, these tasks will execute in the order declared, one after the other.
>
>In essence asset fingerprinting permits your static assets to be served with aggressive caching instructions to a browser. This will result in an improved experience for your users given that subsequent visits to your site will result in less assets requiring to be downloaded. Rails also describes the benefits of asset fingerprinting.
>
>The above declaration of pipelineStages and the requisite addSbtPlugin declarations in your plugins.sbt for the plugins you require are your start point. You must then declare to Play what assets are to be versioned. The following routes file entry declares that all assets are to be versioned:
>
>GET  /assets/*file  controllers.Assets.versioned(path="/public", file: Asset)

Well this looks nice. My build.sbt doesn't have that line at the moment, and I won't add it just yet, but I'll be coming back to here later

## application.conf

This is an important file. The only thing I want to do to it is remove the references to slick in it.

#Running the first Notary

In the Notary Controller and routes file we only have one thing defined: the index page that should return 'Hello World'. Let's try it.

Step 1: Run SBT in /x//git/notary
Step 2: 'clean' (we need to get rid of any junk lying around from things we have deleted)
Step 3: 'run'
Step 4: Navigate browser to http://localhost:9000

In my case I got 'Hello World' on the screen! Hurrah we have a Play framework 2.4 application running.

#Get working in IDE

I use Eclipse as my preferred IDE. My SBT is already set up with the following in ~/.sbt/0.13/plugins/plugins.sbt

>resolvers += "Typesafe repository" at "http://repo.typesafe.com/typesafe/releases/"
>
>addSbtPlugin("com.typesafe.sbt" % "sbt-pgp" % "0.8")
>
>addSbtPlugin("com.typesafe.sbteclipse" % "sbteclipse-plugin" % "4.0.0")  

So in order to work with Eclipse I need to do the following
1: Make this a git repository (I want Eclipse to know this is a git project)
2: Use the sbt command 'eclipse' to create the information that Eclipse needs
3: Import an existing git project into Eclipse

## Making the git repository

This is needed so that Eclipse will know that it is dealing with a Git project. The command 'git init' executed in /x/git/notary is enough for now

## Importing into Eclipse

I use 'File/Import/Git/projects from git' and then navigate to /x/git/notary through the wizard. Having done this I do a manual refresh and clean on it. 

# Setting up Git properly

All I have done so far is 'init' the git repository. I need to commit everything. I use Github as my online Git Repository. The following get's me going

{% highlight scala %}
1: Create a README file
2: git add --all
3: git status
{% endhighlight %}
I take a look at the files that are going to be commited.

And there are a ton I don't want! 

{% highlight scala %}
.cache-main
.cache-tests
.sbtserver/ (a load of files u
a load of files starting with the word activator
{% endhighlight %}

Time to add the following to the end of .gitIgnore
{% highlight scala %}
.cache-main
.cache-tests
.sbtserver/
{% endhighlight %}
And now I can commit

{% highlight scala %}
1: git commit -a -m "Initial Commit"
2: git remote add origin git@github.com:phil-rice/Notary.git
3; git push -u origin master  
{% endhighlight %}

#Heroku

In order to deploy my application I need to get it onto the internet. I already have Heroku set up on my machine, as should you if you followed the guide

The steps though are

|get a Heroku account|[Free from here](http://www.heroku.com) 
|download the Heroku tool belt||
|create a heroku application|heroku create from the command line|
|Check it's set up|git remote --verbose (and see that there is a remote repository called heroku)|
|git push heroku|Pushes to heroku and says make the application live on the internet

Only when I tried it, it didn't work! I found that actually I needed to add back in the line I had edited away from the build.sbt

>resolvers += "Scalaz Bintray Repo" at "https://dl.bintray.com/scalaz/releases"

That done though, the application was live and running on the internet.


#Review of where we are

We have a 'Hello World' Play 2.4 application with minimal extra stuff. We are confident that all the version numbers are correct (that was the main point of using the activator) and
we have a working environment. "sbt run" allows us to  run locally. We can edit the files in our favourite IDE, and deploy easily to Heroku. We have a github repository with the source code
in it. All in all a healthy place to be.
