---
layout: post
title: "Real World Play - Testing Part 2"
date:   2015-11-28 16:00:23 +0000
comments: True
categories:
- scala
- play
- testing
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

The business logic is mostly held in the Digester and the ToHex object. As they are basically just implementing simple functions, with no sideeffects, they are straightforwards t
to test.  For the digester tests, I used this site http://www.fileformat.info/tool/hash.htm to find the digest for "Some string" 

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
       ToHex(Array[Byte](0xFF,0x30,0x5,0xc,0x80,0xFF)) mustEqual ("ff30050c80ff")
    }
  }
}
{% endhighlight %}

Unfortunately the Digester test requires the ToHex code to work, which is often a bad thing for a Unit test. I'm happy with it though: ToHex mostly only exists so that 
Digester can do it's job, and is very simple

Running the tests shows us an error in the HelloWorldSpec. There is a line that requires the page to return the exact text 'Hello World', rather than contain it. Since we 
have totally changed this class, and that test has no business logic captured in it, I delete it.

##View tests

It actually takes a little bit of work to make the view tests. The big problem is the internationalisation. I don't want the view tests to depend on the values in the 
Messages file, as that is very likely to change, thus I will be using my own Messages for the test rather than anything provided automagically by the Play framework.
 I also want to be able to rip content out of HTML, and will need some tools to do this. The view tests are going to look like this when we finish. 
 The xml ripping code is in 'ViewSpec' which we will see shortly

{% highlight scala %}
class IndexViewSpec extends ViewSpec {

  implicit val messages = MessagesForTest(
     "notary.text" -> "TextToBeNotarised", 
     "digest.title" -> "DigestTitle")

  def getContent(digest: Option[String], form: Form[String] = 
    NotaryController.NotaryForm.inputForm): Elem =
       XML.loadString(views.html.page.notary.index.render(
                         digest, form, messages, NotaryController.notaryCss).body)

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
  val contentWithSomeText = 
     getContent(None, NotaryController.NotaryForm.inputForm.fill("someText"))

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
The tests are pretty simple. 'getContent' is used to get an XML block representing the result of the view. It is worth noting that this will explode with an exception 
if the view isn't returning proper XML, but the fact these tests enforce that is a good thing anyway. Let's take a look at ViewSpec. We will define this in the test sub directory
of the common project, as it is clearly test code. 

{% highlight scala %}
package org.validoc.notary.common

trait XmlSpec extends PlaySpec {
 def textIn(elemTag: String, trim: Boolean)(xml: Node, filter: Node => Boolean) = {
    (xml \\ elemTag).filter(filter) match {
      case x if x.size == 0 => None
      case x if x.size == 1 => Some(if (trim) x.text.trim else x.text)
      case x                => throw new RuntimeException(
                                 s"Cannot find $elemTag in \n" + xml)
    }
  }
  def textInTagWithClass(elemTag: String, trim: Boolean = true)(
                         xml: Node, elementClass: String) =
    textIn(elemTag, trim)(xml, t => (t \ "@class").text == elementClass)

  val divContents = textInTagWithClass("div")_
  val spanContents = textInTagWithClass("span")_
  val textAreaContents = textIn("textarea", trim = true)_

  def onlyTagIn(elemTag: String)(xml: Node) = (xml \\ elemTag).toList match {
    case x :: Nil => x
    case x        => throw new RuntimeException(
               s"Cannot find $elemTag in \n" + xml)
  }
  val onlyTextArea = onlyTagIn("textarea")_
  val onlyForm = onlyTagIn("form")_
  val onlyButton = onlyTagIn("button")_
}

trait ViewSpec extends XmlSpec
{% endhighlight %}
This code is fairly straightforward. It does show a nice use for currying I think. The 'textInTagWithClass', although the inability to have default values through the currying
was a little annoying. I don't want to specify the trim value all the time, and without doubt it would be better in the second parameter block. However once curried it
becomes a mandatory parameter ignoring the default values, so I settled for this.

As mentioned above the messages needs sorting out for testing:

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

#Modifying build.sbt

It's pretty clear that these test helpers belong on the common module. Unfortunately when we try and use them, they are 'not detected'. We get error messages telling us that the 
org.validoc.notary.common package doesn't exist. This is because the code in the 'test' section. To allow this to be picked up, we need to modify build.sbt

{% highlight scala %} 
lazy val notary = (project in file("module/notary")).
                settings(commonSettings: _*).
                settings(
                   libraryDependencies ++= notaryDependencies
                ).enablePlugins(PlayScala).
                dependsOn(common  % "test->test;compile->compile", utilities)
{% endhighlight %}

# Unit tests of the controller

This required a little more refactoring than I expected. Again like the view test I want to inject the Messages. I'm still using objects for the controller, but it's challenging to inject anything 
a companion object safely, so I split the behaviour into a trait, and had the controller extend the trait

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
      val request = FakeRequest(POST, "/").
                 withJsonBody(Json.parse(s"""{ "notary.text": "Some string" }"""))
      val result = controller.postText()(request)

      val xml = xmlOfResult(result)
      spanContents(xml, "digestValue") mustEqual 
           (Some("3febe4d69db2a2d620fa73388dbd3aed38be5575"))

      checkResults(result)
    }

    "not display the digest div, and say 'This field is required' if there was no text in the text area" in {
      val request = FakeRequest(POST, "/").
                 withJsonBody(Json.parse(s"""{ "notary.text": "" }"""))
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

# Browser Tests
At the moment we have tested the business logic with the unit tests, we've tested that the view displays the digest if and only if it is needed. We've checked that the controller 
puts correct digest for the test that was posted to it.

I am a big fan of the browser tests though. This checks the actual workflow against an actual server. As much as anything it is testing 'the plumbing all works'. I don't need to 
test everything I did in the view or controller tests, but doing the basic workflow of 'entering some test, clicking submit and getting the digest' is easy to write and gives
us a lot of confidence that the web site is working. I call these weak tests a 'smoke test' after the days when I was working on electronic equipment, and we would turn the power on
and smell for smoke before actually doing any proper tests.

{% highlight scala %}
package org.validoc.notary

import org.scalatestplus.play._
import org.validoc.notary.common.BrowserSpec

class IndexPageSpec extends BrowserSpec with OneServerPerSuite with OneBrowserPerTest with ChromeFactory {
  "The index page" must {
    "initially not show the digest result" in {
      go to (s"http://localhost:$port/")
      divContents(xmlSource, "digestText") mustEqual (None
    }
    "allow text to be posted and display its digest" in {
      go to (s"http://localhost:$port/")
      textArea("notary.text").value = "Some string"
      submit
      spanContents(xmlSource, "digestValue") mustEqual (Some("3febe4d69db2a2d620fa73388dbd3aed38be5575"))
    }
  }
}
{% endhighlight %}

This used the trait BrowserSpec. I have this so that I can have all my browser integration tests extending one trait, letting the IDE and reflection help me later. I modified
the LoginIntegrationTest to use it as well. 

{% highlight scala %}
trait BrowserSpec extends PlaySpec with XmlSpec {
  def xmlSource(implicit driver: WebDriver) = XML.loadString(driver.getPageSource)
}
{% endhighlight %}

#Summary
At this point we are quite confident that the web site works. Confident enough that if the automated tests the behaviour is good enough to go live. There are no tests yet for 
the 'beautification' or 'performance', but we'll add those as we go through the project.

