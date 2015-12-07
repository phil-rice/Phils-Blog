---
layout: post
title: "Real World Play - Testing Part 2"
date:   2015-11-28 16:00:23 +0000
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

The code for at the start of this can be found here [https://github.com/phil-rice/notary3](https://github.com/phil-rice/notary3) and at the 
final code can be found [https://github.com/phil-rice/notary4](https://github.com/phil-rice/notary4) 

#Review

At the moment we have a small Scala Play application with a few modules. There are two controllers: One that handles the index page and displays hello world, and another that does a very simple login
story (and doesn't even have a web view). We want to test the code we added in the last blog: the code that takes a block of test, digests it, and displays it in the index page.

#Goals

At the end of this we will have the the ability to enter some text and display the 'digest' of it. We 'sort of' have that in that the code (probably) works, but we don't know
we have it until we have written tests for it, and have those tests pass.

#Unit tests

In order test the Digest and toHex code we should move that code. I'm moving the code to the utilities module, and while I'm doing it making it a little nicer

{% highlight scala %}
package org.validoc.utilities.digest
import java.security.MessageDigest

object ToHex {
  def apply(b: Byte): String = { val s = "0" + b.toInt.toHexString; s.substring(s.length - 2) }
  def apply(b: Array[Byte]): String = b.map(ToHex(_)).mkString("")
}

object Digester {
  def asByteArray(s: String) = {
    val md = MessageDigest.getInstance("SHA");
    md.update(s.getBytes);
    md.digest()
  }
  def apply(s: String) = ToHex(asByteArray(s))
}
{% endhighlight %}
This needs a very minor change in the NotaryController: deleting the Digester code

{% highlight scala %}
  def postText = Action { implicit request =>
    NotaryForm.inputForm.bindFromRequest().fold(
      withErrors => Ok(views.html.page.notary.index(None, withErrors)),
      { value =>
        println(s"Got value $value")
        Ok(views.html.page.notary.index(Some(Digester(value)), NotaryForm.inputForm))
      })
  }
}
{% endhighlight %}

##The tests

For the digester tests, I used this site http://www.fileformat.info/tool/hash.htm to find the digest for "Some string" 

{% highlight scala %}
class DigestSpec extends PlaySpec{
  
  "The digestor" should {
    "turn find the bytes in the digest, then turn them into hex" in {
      val bytes  = Digester.asByteArray("Some string")
      Digester("Some string") mustEqual ToHex(bytes)
    }
    "return a hex string which is the digest of the byte array it is passed" in {
      Digester("Some string") mustEqual("3febe4d69db2a2d620fa73388dbd3aed38be5575")
    }
  }
}

class ToHexSpec extends PlaySpec {
  implicit def toByte(i: Int) = i.toByte
  "ToHex" should {
    "turn a single byte to hex " in {
      ToHex(0) mustEqual ("00")
      ToHex(0x5) mustEqual ("05")
      ToHex(0xc) mustEqual ("0c")
      ToHex(0x30) mustEqual ("30")
      ToHex(0x80) mustEqual ("80")
      ToHex(0xFF) mustEqual ("ff")
      ToHex(-1) mustEqual ("ff")
    }
    "return a hex string which is the digest of the byte array it is passed" in {
    	ToHex(Array[Byte](0xFF, 0x30, 0x5, 0xc, 0x80, 0xFF)) mustEqual ("ff30050c80ff")
    }
  }
}
{% endhighlight %}

Well that's the caculations tested. To be fair the Digester test requires the ToHex code to work, which is often a bad thing for a Unit test. I'm happy with though: ToHex mostly only exists so that Digester can do it's job

Running the tests shows us an error in the HelloWorldSpec. There is a line that requires the page to return the exact text 'Hello World', rather than contain it. Since we want to totally change this class, I'm going to delete it!

##View tests

It actually takes a little bit of work to make the view tests. The big problem is the internationalisation. I don't want the view tests to depend on the values in the Messages file,
as that is very likely to change. The view tests are going to look like this when we finish

{% highlight scala %}
class IndexViewSpec extends ViewSpec {

  implicit val messages = MessagesForTest("notary.text" -> "TextToBeNotarised", "digest.title" -> "DigestTitle")

  def getContent(digest: Option[String], form: Form[String] = NotaryController.NotaryForm.inputForm): Elem =
    XML.loadString(views.html.page.notary.index.render(digest, form, messages, NotaryController.notaryCss).body)

  "The index view with None as digest" should {
    "hide the digest class" in {
      val content = getContent(None)
      divContents(content, "digestText") mustEqual (None)
      spanContents(content, "digestTitle") mustEqual (None)
      spanContents(content, "digestValue") mustEqual (None)
    }
  }
  "The index view with Some(digest)" should {
    "show the digest class and digest itself" in {
      val content = getContent(Some("actualDigest"))
      divContents(content, "digestText").isDefined mustBe (true)
      spanContents(content, "digestTitle") mustEqual (Some("DigestTitle")) 
      spanContents(content, "digestValue") mustEqual (Some("actualDigest"))
    }
  }
  val contentWithSomeText =  getContent(None, NotaryController.NotaryForm.inputForm.fill("someText"))

  "The index view" should {
    "have the text in the 'form' displayed in a testarea" in {
      onlyTextArea(contentWithSomeText).text mustEqual ("someText")
    }
    "have the only textarea inside a form" in {
      val form = onlyForm(contentWithSomeText) 
      onlyTextArea(form).text mustEqual ("someText")
    }
    "have a submit button inside a form" in {
      (onlyButton(contentWithSomeText) \ "@type").text mustEqual ("submit")
    }
  }
}
{% endhighlight %}
The tests are pretty simple. They use a number of helper methods and MessagesForTest. The following shows how the methods like 'divContents' work. The standard Scala XML library
is used. This does require the view to return xml, but the fact these tests enforce that is a good thing anyway.

{% highlight scala %}
trait XmlSpec extends PlaySpec {
 def textIn(elemTag: String, trim: Boolean)(xml: Node, filter: Node => Boolean) = {
    (xml \\ elemTag).filter(filter) match {
      case x if x.size == 0 => None
      case x if x.size == 1 => Some(if (trim) x.text.trim else x.text)
      case x                => throw new RuntimeException(s"Cannot find $elemTag in \n" + xml)
    }
  }
  def textInTagWithClass(elemTag: String, trim: Boolean = true)(xml: Node, elementClass: String) =
    textIn(elemTag, trim)(xml, t => (t \ "@class").text == elementClass)

  val divContents = textInTagWithClass("div")_
  val spanContents = textInTagWithClass("span")_
  val textAreaContents = textIn("textarea", trim = true)_

  def onlyTagIn(elemTag: String)(xml: Node) = (xml \\ elemTag).toList match {
    case x :: Nil => x
    case x        => throw new RuntimeException(s"Cannot find $elemTag in \n" + x.mkString("\n") + "\nWhole xml is\n" + xml)
  }
  val onlyTextArea = onlyTagIn("textarea")_
  val onlyForm = onlyTagIn("form")_
  val onlyButton = onlyTagIn("button")_
}

trait ViewSpec extends XmlSpec
{% endhighlight %}
The messages need to be mocked up as well. This turned out to be quite a bit harder than I expected!

{% highlight scala %} 
object MessagesForTest {
  def apply(kvs: (String, String)*) = {
    val map = Map(kvs: _*)
    Messages(Lang.defaultLang, new MessagesApi {
      def apply(key: String, args: Any*)(implicit lang: Lang): String =
        map.get(key) match {
          case Some(value)                        => MessageFormat.format(value, args)
          case _ if key.startsWith("constraint.") => key
          case _                                  => throw new RuntimeException(key)
        }

      def apply(keys: Seq[String], args: Any*)(implicit lang: Lang): String = ???
      def messages = Map()
      def preferred(candidates: Seq[Lang]): Messages = ???
      def preferred(request: RequestHeader): Messages = ???
      def preferred(request: play.mvc.Http.RequestHeader) = ???
      def setLang(result: Result, lang: Lang) = ???
      def clearLang(result: Result) = ???
      def translate(key: String, args: Seq[Any])(implicit lang: Lang) = ???
      def isDefinedAt(key: String)(implicit lang: Lang): Boolean = ???
      def langCookieName: String = ???
      def langCookieSecure: Boolean = ???
      def langCookieHttpOnly: Boolean = ???
    })
  }
}
{% endhighlight %}

# Unit tests of the controller

This required a little more refactoring than I expected. Again like the view test I want to inject the Messages. I'm still using objects for the controller, but I need to inject messages into it.
To do this I put the behaviour into a trait, and had the controller extend the trait

{% highlight scala %} 
trait NotaryController extends Controller {
  implicit val notaryCss = Css(HelloAssets, "hello")

  object NotaryForm {
    val inputForm = Form(single("notary.text" -> nonEmptyText))
  }

  implicit def messages(implicit request: Request[_]): Messages

  def index = Action { implicit request =>
    Ok(views.html.page.notary.index(None, NotaryForm.inputForm))
  }

  def postText = Action { implicit request =>
    NotaryForm.inputForm.bindFromRequest().fold(
      withErrors => Ok(views.html.page.notary.index(None, withErrors)),
      { value =>
        println(s"Got value $value")
        Ok(views.html.page.notary.index(Some(Digester(value)), NotaryForm.inputForm))
      })
  }

}

object NotaryController extends NotaryController {
...
{% endhighlight %}

The tests are pretty simple. We are confident the view works. We have only a few things we want to check. In some ways it would be nice to check 'this view was called with these 
parameters', but I don't actually know how to do that neatly. The approach used here is to use some aspect of the view. For example 'is the digest div visible', 'has the digest value'
been set, and 'has the normal Play framework error code been put in place when there was an error. One interesting thing about this sort of testing is that sometimes
'less is more'. The more thoroughly we test things, the more we lock this test down to just duplicating the view test, and as the project grows other tests as well. The purpose of
this test is to exercise the paths through the controller, and check that the right data has been put in the view, and that's about it. 
{% highlight scala %} 
class NotaryControllerSpec extends ContollerSpec with Results {

  val controller = new NotaryController {
    implicit def messages(implicit request: Request[_]) =
      MessagesForTest(
        "notary.text" -> "TextToBeNotarised",
        "digest.title" -> "DigestTitle")
  }

  def checkResults(result: Future[Result]) {
    status(result) mustEqual OK
    contentType(result) mustBe Some("text/html")
    charset(result) mustBe Some("utf-8")
  }

  "The NotaryController when index is 'GET'" should {
    "not have the digest div" in {
      val result = controller.index()(FakeRequest())

      val xml = xmlOfResult(result)
      divContents(xml, "digestText") mustEqual (None)
      checkResults(result)
    }
  }

  "The NotaryController when index is 'POST' with data from text Area" should {
    "display the digest div if there was text" in {
      val request = FakeRequest(POST, "/").withJsonBody(Json.parse(s"""{ "notary.text": "Some string" }"""))
      val result = controller.postText()(request)

      val xml = xmlOfResult(result)
      spanContents(xml, "digestValue") mustEqual (Some("3febe4d69db2a2d620fa73388dbd3aed38be5575"))

      checkResults(result)
    }

    "not display the digest div, and say 'This field is required' if there was no text in the text area" in {
      val request = FakeRequest(POST, "/").withJsonBody(Json.parse(s"""{ "notary.text": "" }"""))
      val result = controller.postText()(request)

      val xml = xmlOfResult(result)
      spanContents(xml, "digestValue") mustEqual (None)
      val errorDl = onlyTagWithId("dl")(xml, "notary_text_field")
      (errorDl \ "@class").text mustEqual(" error")
      checkResults(result)
    }
  }
}
{% endhighlight %}
This required me to add 'onlyTagWithId' to the XmlSpec train that ControllerSpec inherits from:

{% highlight scala %}
  def onlyTagWithId(elemTag: String)(xml: Node, id: String) =
    (xml \\ elemTag).filter(x => (x \ "@id").text == id).toList match {
      case x :: Nil => x
      case x        => throw new RuntimeException(s"Cannot find $elemTag in\n" + xml)
    }
{% endhighlight %}


#Refactoring

This is quite nice, but the template 'main' is absolutely going to be shared by multiple modules and doesn't belong in hello. The login pages are almost certainly going to want either 
main or a similar template. Even more interestingly is 'how is that menu bar going to work'. The template main is certainly going to link to the index page, but is also going to link to 
login in. In fact given modern styles there is every chance that there is going to be a login area inside the menu bar. How are we going to make all that work?

## CSS

The approach taken with CSS in the main template can't work the way we have it at the moment. Let's look at the constraints on us. Firstly we want the css to be specificed in the 'head' of the 
page. That means it has to be generated by the main template. Secondly the css files themselves want to be stored in the same module as the page that specifies them. For example
the login css wants to be specified in the login module. 

A simple solution is make Css a class. 
{% highlight scala %}
package org.validoc.notary
case class Css(assetts: controllers.AssetsBuilder, file: String)
{% endhighlight %}
It's very likely (and we can demand it) that there will be a 'main' css that every page served by a controller will use. We can force this by changing our template and our sample page
{% highlight scala %}
@(title: String, cssList: List[org.validoc.notary.Css]=List())(content: Html)(implicit css: org.validoc.notary.Css)
@import org.validoc.notary.Css
<!DOCTYPE html>
      ....
      @for(Css(assetts, file) <-cssList :+ css){
           <link rel="stylesheet" media="screen" href='@assetts.versioned("assets/stylesheets", s"${file}.css")'>
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
  implicit val notaryCss = Css(HelloAssets, "hello")
  ...
{% endhighlight %}

## Moving the 'main' template
Now it works here, we can move the main template into a shared project. Often I make a project called 'common' just for that purpose, and that's what we will do here.

* Modify build.sbt and Dependencies.scala
* Update my IDE
* Move the code
* Test it works

### Build.sbt

Cut and paste (and editing) is our friend here. It would of course be possible to simplify this and generalise it, but as this is the main build file I'm happy with the clarity of 
repeating: especially as all the places that would need to be changed if we change our mind are in one place. As well as this code I added 'common' to the 'dependsOn' for the root, hello
and user modules, and to the 'aggregate' for the root.

{% highlight scala %}
lazy val common = (project in file("module/common")).
                settings(commonSettings: _*).
                settings(
                   libraryDependencies ++= commonDependencies
                ).enablePlugins(PlayScala).
                dependsOn(  utilities % "test->test;compile->compile")
{% endhighlight %}				  
### Dependencies 
				  
{% highlight scala %}
  val commonDependencies =  Seq()
{% endhighlight %}				  


The really nice thing is that with Play's strong typing if it runs at all after an activity like this we can be reasonably sure the application works, and with our tests we can be 
even more confident.

### Update my IDE

I use the SBT command 'eclipse' to do this, import the new 'commons' project and refresh all projects. 
This has the nice sideeffect of creating the module 'common', but you could do this manually if you want.

### Move the code

I create an app directory in common, and just move the folder containing main.scala.html. The package name is wrong now, so I have to change it to 'views.template.notary.common' and chase
and fix all the compilation errors


#Review

We have added some views using a simple templating story. We put some business logic in our main controller that used those views. We wrote some tests for that main controller, and some integration tests for the browser.
While that took a lot of code, quite a lot of the code is 'infrastructure' that will help us with future tests.