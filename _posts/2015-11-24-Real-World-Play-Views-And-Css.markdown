---
layout: post
title: "Real World Play - Views And CSS"
date:   2015-11-23 16:00:23 +0000
comments: True
categories:
- scala
- play
- refactoring
---

This follows the work 

* [started here](/scala/play/2015/11/16/getting-started-with-play.html)
* [tested here](/scala/play/testing/2015/11/22/getting-going-with-play-and-testing.html) 
* [Split into modules here](/scala/play/refactoring/2015/11/22/getting-going-with-play-splitting-into-modules.html)

The code for at the start of this can be found here [https://github.com/phil-rice/notary2](https://github.com/phil-rice/notary2) and at the 
final code can be found [https://github.com/phil-rice/notary3](https://github.com/phil-rice/notary3) 

#Review

At the moment we have a small Scala Play application with a few modules. There are two controllers: One that handles the index page and displays hello world, and another that does a very simple login
story. Both have tests. The purpose of this play application is going to be a digital notary, although very little of the work so far has
cared at all about the actual application!

#Goals

At the end of this we want the ability to enter some text and display the 'digest' of it. This is definitely a step in the right direction for our
digital notary. It is important that we have unit and browser tests for our application. In this blog we won't worry about testing: we'll look at that in the next blog

#Getting going

We are going to start with a page that has a form on it. To have a browse button on that form that selects a file, and for that form to then post the data. Some other page is
then usually displayed, with the results of uploading visible 

At the moment we don't really have any 'pages' in our application. Since our application is a digital notary, there is nothing wrong with this form being the index page. This means
our HelloWorld is about to be replaced with a proper page.

Proper pages need Html and CSS. In most projects both of these start simple and then end up seriously complex. If we aren't careful, and don't refactor ruthlessly, then we will have a mess. It is
tempting to start with a 'plan'. In half my projects I find I have a specification, in the others I don't. In both the moment the user sees the actual working system they want it changed. 
For me it's far more important to have code, especially in this area, than it is to have a plan. Even so a rough one helps

#Rough Layout

Like most websites we will have a menu bar at the top, some links at the bottom and content in the middle. Somewhere around the menu bar will be the site icon and clicking it will take us to the
index page.  I am going to have a template 'main' that holds this template. As we are using the play framework we'll use the twirl templates. Even at this early stage I like splitting them 
into two packages: one for pages and one for templates. I tend to include the module name in the package as well, to avoid any naming issues.

#Making the first page

To get our first page I create the following files

| Root directory| package and file |purpose
| /module/notary| views.template.notary.main.scala.html|This is the 'main template' that will hold icon, menu bar and so on|
| /module/notary| views.page.notary.index.scala.html|This is the 'index' page that will be using the main template and holding the form for uploading|

The naming structure for the packages is quite important. 'views' is mandated by the play framework. I like the 'page' or 'template' next, as that helps me a lot when understanding the
purpose of the file. I tend to then put 'project.subproject' next, followed by the actual name, then the file type 'scala.html'. These tend to get to be a mouthful. Given that
the project is called notary, and the sub project is too, I compress those down to just .notary instead of .notary.notary.  

### views.page.notary.index.scala.html

At the moment I'm not adding the form, so I'll leave Hello World as the main text. This allows me to see something on the screen, and gives me confidence that all the wiring is
in place

{% highlight scala %}
@()
@import template.notary._
@main("Validoc"){
   Hello World
}
{% endhighlight %}

### views.template.notary.main.scala.html

I'm not worrying about menus at the moment, just want a 'place holder' for it. I know that this template is going to be shared by a load of files (most in fact) and that the 
css is going to be 'a big deal'. I don't want one ginormous css file, so I am putting a place holder in for that sort of code too.
It's not actually good enough though, because this just hard codes the HelloAssets controller. If I want to reuse this template, then I'm going to have to work out how to deal 
with that. We'll look at how to do that after we get the files uploading. We'll do that in a bit

{% highlight scala %}
@(title: String, css: List[String]=List())(content: Html)
<!DOCTYPE html>
<html>
  <head>
     <title>@title</title>
      @for(c<-css){
           <link rel="stylesheet" media="screen" 
              href='@controllers.notary.routes.NotaryAssets.
                    versioned(s"stylesheets/$c.css")'/>      
      }
  </head>
  <body>
	<div class='mainMenu'>Menu goes here</div>
	@content
	<div class='bottomRow'>Bottom row goes here</div>
  </body>
</html>
{% endhighlight %}

# NotaryController

This needs to be changed to 
{% highlight scala %}
object NotaryController extends Controller{
  def index = Action { Ok(views.html.page.notary.index()) }
}
{% endhighlight %}

## Adding our first CSS

Let's put something ugly in space for the bottom row. We aren't beautifying the website, so all we want to do is ensure that the Css is getting picked up. We'll create a file
main.css, that we put in the public/stylesheets sub directory of the notary module
{% highlight css %}
.bottomRow {
	background: red;
}
{% endhighlight %}

## Seeing what we have

Using the command 'run' allows us to view the website. It looks ugly, but we have separated out the template from the index page, and we have our first Css being picked up.


# Making the form work with just a text area

The following assumes [you have read this](https://www.playframework.com/documentation/2.4.x/ScalaForms). If not, I recommend reading it before continuing.

We have a number of tasks to do

* Implement the form view
* Implement a Scala Play form holding the relevant data
* Implement an action on a controller that will process the form
* Do the model work of coding up digests
* Implement the view of the 'here is your digest'

For now we will do this in the index page. We are going to need a 'proper place' to declare the forms, but the following code gets us started

On the index.html.scala we start with 
{% highlight scala %}
@(notaryForm: Form[String])
@import template.notary._
@import controllers.notary._
@main("Validoc"){
   @helper.form(action = routes.NotaryController.postText()) {
      @helper.textarea(field = notaryForm("notary.text"), args = 'rows -> 3, 'cols -> 50)
      <button type='submit' >Submit</button>
   }
}
{% endhighlight %}
There is quite a bit in this small piece of code. Firstly there is the fact that a 'Form[String]' called notaryForm will hold the data about the form. the '@helper.form' method puts
all the html in place for the form. The action is how this html is linked to the controller. To make this type safe magic work, in the notary.routes file we need to add the following
{% highlight scala %}
POST   /      controllers.notary.NotaryController.postText
{% endhighlight %}

This of course requires us to actually have that method 'postTest'. We can start with a simple one
{% highlight scala %}
object NotaryController extends Controller {
  object NotaryForm {
    val inputForm = Form(single("notary.text" -> nonEmptyText))
  }

  def index = Action { Ok(views.html.page.notary.index(NotaryForm.inputForm)) }

  def postText = Action { implicit request =>
    NotaryForm.inputForm.bindFromRequest().fold(
      withErrors => Ok(views.html.page.notary.index(withErrors)), {
        value =>
          println(s"Got value $value")
          Ok(views.html.page.notary.index(NotaryForm.inputForm))
      })
  }
}
 {% endhighlight %}
Let's examine this code. The object NotaryForm is the current place I am keeping my forms. In a little while I am likely to refactor this to its own place. 
  The inputForm is pretty simple. There is only a 'single' value which has a 'key' of 'notary.text'. I like the 'a.b.c' notation here as this makes the identifiers unique. 
  Unique identifiers eases my life when it comes to CSS: we will almost certainly be wanting to style this text when we beautify the site. 
  
  In the postText method, we can see the 'bindFromRequest' followed by fold which is the normal means for getting data from posts. Note that I had to modify the Action to let the bindRequest
  method compile. Instead of   Action{ Ok(...) } it was Action { implicit request => Ok(...)}. There are two functions passed into bindRequest. The first is executed if
 there are any errors, and just redisplays the form with the data about errors in the 'withErrors' form. The second is executed when things go well, prints the value that was in
 the text field to the screen and then displays a blank form.
 
## Internationalisation
  
Unfortunately the above doesn't work. The error message when when we compile it is
{% highlight scala %}
[info] Compiling 7 Scala sources and 1 Java source to ....
[error] ....\notary\module\hello\app\views\page\notary\index.scala.html:6: 
        could not find implicit value for parameter messages: play.api.i18n.Messages
[error]       @helper.textarea(field = notaryForm("notary.text"),
                               args = 'rows -> 3, 'cols -> 50)
 {% endhighlight %}
 The implicit messages object it is referring too is the one that is needed for internationalistion. [It is worth reading this for an overview](https://www.playframework.com/documentation/2.4.x/ScalaI18N).
 In essence though the Messages file in the conf subdirectory of the current project is used to turn 'notary.text' into a format suitable for the current browser language. To do 
 this there has to be an implicit Messages object in scope for the 'helper.textArea' method. It is 'passed in' rather than defaulting for all the good reasons that we don't like
 Global parameters. And just like not using Global parameters it causes us a little bit of pain without them, in order to fix a much bigger pain later in the project  
  
Let's do this 'little bit of pain' npw. We start by adding an implicit messages to index.html.scala 

{% highlight scala %}
@(notaryForm: Form[String])(implicit messages: Messages)
@import template.notary.hello._
@import controllers.hello._
@main("Validoc"){
   @helper.form(action = routes.NotaryController.postText()) {
      @helper.textarea(field = notaryForm("notary.text"), 
                       args = 'rows -> 3, 'cols -> 50)
     <button type="submit" >Submit </button>
   }
}
{% endhighlight %}

Now the pain has moved. We need an implicit Messages anywhere this is used. At the moment this is in the notary controller

{% highlight scala %}
object NotaryController extends Controller {
  implicit val messages: Messages = ???
{% endhighlight %}

This compiles, but doesn't run. How do we create this messages object?
The following code works. 
{% highlight scala %}
  def index = Action {
	  import play.api.Play.current
    import play.api.i18n.Messages.Implicits._
    Ok(views.html.page.notary.hello.index(NotaryForm.inputForm))
  }

  def postText = Action { implicit request =>
  import play.api.Play.current
    import play.api.i18n.Messages.Implicits._
    NotaryForm.inputForm.bindFromRequest().fold(
      withErrors => Ok(views.html.page.notary.hello.index(withErrors)),
      { value =>
        println(s"Got value $value")
        Ok(views.html.page.notary.hello.index(NotaryForm.inputForm))
      })
  }
{% endhighlight %}
It is though violating the 'Don't Repeat Yourself' principle quite a lot. We can move the import play.api.Play.current outside the method, but not the 'Messages.Implicits': they require an implicit request. 
The following is slightly cleaner code, removing this plumbing from the method. 

{% highlight scala %}
  implicit def messages(implicit request: Request[_]) = {
    import play.api.Play.current
    import play.api.i18n.Messages.Implicits._
    implicitly[Messages]
  }

  def index = Action {implicit request =>
    Ok(views.html.page.notary.index(NotaryForm.inputForm))
  }

  def postText = Action { implicit request =>
    NotaryForm.inputForm.bindFromRequest().fold( 
      withErrors => Ok(views.html.page.notary.index(withErrors)),
      { value =>
        println(s"Got value $value")
        Ok(views.html.page.notary.index(NotaryForm.inputForm))
      })
  }
{% endhighlight %}

#Seeing it work

We can use the run command in SBT and navigate to localhost:9000 to see it working. When we just click submit, the error message 'This field is required' is displayed, which tells us that the 
postText action is being displayed. If we put some text in the box then click submit, that text appears in the SBT console and the page returns with the box blank: just as we expect.

#Adding the business logic

This is pretty straightforwards. We want to display the digest if it has been calculated. The classical Scala way to represent this digest is with an option, and here is some code to do it. We will
refactor it into a different place soon, but it speeds up writing if we put it close to where it is used. We change index.scala.html to 

{% highlight scala %}
@(digest: Option[String], notaryForm: Form[String])(implicit messages: Messages)
@import template.notary._
@import controllers.notary._
@main("Validoc"){
   @if(digest.isDefined){
   	  <div class='digestText'><span class='digestTitle'>@messages("digest.title") 
   	        <span class='digestValue'>@digest</span>
   	  </div>
   }
   @helper.form(action = routes.NotaryController.postText()) {
      @helper.textarea(field = notaryForm("notary.text"), 
                       args = 'rows -> 3, 'cols -> 50)
      <button type='submit' >Submit</button>
   }
}
{% endhighlight %}
The reason for the div and span is two fold. Firstly it's obviously nice to be able to format the data differently. But the main reason is probably this second one: it makes testing much easier. When we
get to the test for 'did the digestText appear', we can simply look for the presence or absence of the class digestText. We can also get the exact digest from the 'digestValue'. The 'messages("digest.title") allows us some
amount of internationalisation. The text can be changed, but not the left/right nature, and it's not possible to easily embed the digest into the text (such as 'Your digest is [3f17...], don't forget to email it"). I don't like putting
HTML into messages if I can avoid it, and I like the ability to make the digest formated differently and easy to test more than I am worried about the left right problem. If I come to add a language with right/left dominance, then I'll probably put
a 'notary.text.left' and a 'notary.text.right' text message in
  
 Now we need to modify NotaryController. At the moment we are putting the digest calculation in there, but once we have it roughly working we'll refactor it to it's proper destination 
{% highlight scala %}
  def toHex(b: Byte) = {val s = "0" + b.toInt.toHexString; s.substring(s.length - 2)}
  
  def digest(s: String) = {
    val md = MessageDigest.getInstance("SHA");
    md.update(s.getBytes);
    md.digest().map(toHex).mkString("")
  }

 def index = Action { implicit request => 
    Ok(views.html.page.notary.index(None, NotaryForm.inputForm)) }

  def postText = Action { implicit request =>
    NotaryForm.inputForm.bindFromRequest().fold(
      withErrors => Ok(views.html.page.notary.index(None, withErrors)),
      { value =>
        println(s"Got value $value")
        Ok(views.html.page.notary.index(Some(digest(value)), NotaryForm.inputForm))
      })
  }
{% endhighlight %}

#Refactoring

We have a working system now. My philosophy on 'I've got myself a working system now' is to commit it to a branch, then review it looking for issues and opportunities to refactor.
Very often this if better done 'the next day'!. As an example here on review it becomes obvious that the template 'main' is absolutely going to be shared by multiple modules and 
doesn't belong in the notary project. The login pages are almost certainly going to want either main or a similar template. Even more interestingly is 'how is that menu bar going to work'. 
The template main is certainly going to link to the index page, but is also going to link to login in. In fact given modern styles there is every chance that there is going to be a 
login area inside the menu bar. How are we going to make all that work? 

We'll start by moving the template to 'common'. That needs us to address how we will deal with CSS. We want to define  the CSS in the sub module that the page is created in, 
not the sub module that defines the template. We'll then move the template to common. Issues about hyperlinks between sub modules we'll address in a later blog. 

## CSS

Let's look at the constraints on us. Firstly we want the css to be specificed in the 'head' of the html. That means it has to be generated by the main template. Secondly the css files 
themselves want to be stored in the same module as the page that specifies them. For example the login css wants to be specified in the login module. 

A simple solution is make Css a class. It is just going to wrap a 'Call', but I am a fan of strong typing so I think it's worth the slight extra work

{% highlight scala %}
package org.validoc.notary.hello
case class Css(call: Call)
{% endhighlight %}

It's very likely (and we can demand it) that there will be a 'primary' css that every page served by a controller will use. We can force this by changing our template and our sample page
{% highlight scala %}
@(title: String, cssList: List[org.validoc.notary.hello.Css]=List())(content: Html)(implicit css: org.validoc.notary.hello.Css)
@import org.validoc.notary.Css
<!DOCTYPE html>
      ....
      @for(Css(assetts, file) <-cssList :+ css){
          <link rel="stylesheet" media="screen" href='@c.call' />
      }
      ....
{% endhighlight %}
And in the page that uses this
{% highlight scala %}
@(notaryForm: Form[String])(implicit messages: Messages, css: org.validoc.notary.Css)
...
@main("Validoc"){
...
{% endhighlight %}
To make this work for the NotaryController:
{% highlight scala %}
object NotaryController extends Controller {
   implicit val notaryCss = Css(routes.HelloAssets.versioned("hello.css"))
  ...
{% endhighlight %}

At this point we can run the tests, and run the application and see everything still works.

## Moving the 'main' template
Now the 'main' template is less coupled to the module it is defined in we can move the main template into 'common'. I need to create an app directory in common, and just move the package 
containing main.scala.html. The package name is wrong now, so I have to change it to 'views.template.notary.common' and chase down and fix all the compilation errors

#Review

We have added some views using a simple templating story. We put some business logic in our main controller that used those views. We wrote some tests for that main controller, and some integration tests for the browser.
While that took a lot of code, quite a lot of the code is 'infrastructure' that will help us with future tests.