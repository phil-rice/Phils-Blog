---
layout: post
title: "Real World Play -Testing Part 1"
date:   2015-11-22 16:00:23 +0000
comments: True
categories:
- scala
- play
- testing
---

This follows the work [started here](/scala/play/2015/11/16/getting-started-with-play.html). 

The code for this can be found here [https://github.com/phil-rice/notary1](https://github.com/phil-rice/notary1)

#Review

At the moment we have a (small) monolithic Scala Play application. I'm going to make it a little bigger before I start splitting it up. A sensible thing to add
very early on is a simple login story. After doing this simple login, we'll look at how to test it.

#Getting started

As we are using Scala Play 2.4, we need to use a suitable testing framework for it. This website https://www.playframework.com/documentation/2.5.x/ScalaFunctionalTestingWithScalaTest 
recommends using ScalaTest + Play, which can be found here http://www.scalatest.org/plus/play.

The following added to the build file seems to work, although I'd rather use an actual released version (not available at the time of writing) 

{% highlight scala %}
libraryDependencies ++= Seq(
 "org.scalatestplus" %% "play" % "1.4.0-M3" % Test
)   
 {% endhighlight %}

#Representing Users

At the moment we don't know what sort of data we will be holding on users or  how we are going to store them. These are important decisions, but we don't need to make 
them now. There is a saying in IT 'one more layer of indirection solves every problem'. The layer of indirection we will use is the Type Class User. The only thing we need
to know about for a simple login story is the username

{% highlight scala %}
package org.validoc.notary.users
trait User[A] {
  def userName(a: A): String
}

case class SimpleUser(userName: String)

object SimpleUser {
  implicit object SimpleUserAccess extends User[SimpleUser] {
    def userName(a: SimpleUser) = a.userName
  }
}
 {% endhighlight %}
 
#First Test

This test is almost so simple it's 'not worth testing'. Still it doesn't hurt to write it, and it doesn't take long. 
 
{% highlight scala %}
package org.validoc.notary.users
import org.scalatestplus.play.PlaySpec
import play.api.mvc._
class SimpleUserSpec extends PlaySpec with Results {

  "The Simple User" must {
    "Return the user name" in {
      val su = SimpleUser("someName")
      implicitly[User[SimpleUser]].userName(su) mustBe "someName"
    }
  }
}
 {% endhighlight %}
The test is simple. It's just ensuring that the implicit object is picked up, and that the Type Class method 'userName' for User[SimpleUser] returns the userName field in SimpleUser.
What is interesting is that before long we are likely to want to test 'User[A]' like behaviour for at least one implementation: 'the real one'. At that point we'll probably use
the approach in [this discussion](/scala/testing/2015/11/11/dealing-with-combinatorial-explosions-when-unit-testing-type-classes.html) 

This test is our first 'domain model' test. It has nothing to do with Play. 

#Login
 
Let's make some behaviour that is worth testing.  Play stores session state in encrypted cookies. For our first login story just storing the username in the session state is adequate. Later we will want to put timeouts in it, and perhaps
 strengthen the story in other ways. Again a level of indirection is our friend here. We can put off making these decisions too. The level of indirection helps with tests too
{% highlight scala %} 
package controllers
import play.api.mvc._
import org.validoc.notary.users.User
trait LoginControllerTrait[A] {
  self: Controller =>
  implicit val userLike: User[A]

  def userInSession: UserInSession
  def userInRequest: UserInRequest

  def index = Action { Ok("Hello World") }

  def login = Action { implicit request =>
    val userName = userInRequest.userName
    userInSession.addUserName(Ok(s"Logged in as $userName"), userName)
  }

  def who = Action { implicit request => 
                     Ok(s"You are logged in as ${userInSession.userName}") }

  def logout = Action { implicit request => userInSession.clear(Ok("Logged Out")) }
}
{% endhighlight %}

This is a controller that is abstracted away from all the details of how the login is performed or the webpage with the login details on it. The website is very ugly, 
but we don't need to worry about that at the moment. At the moment the only way to navigate to the pages is to manually type them in. This is more of a 'RESTful service' then a
website, but we'll improve it with time.  

The UserInRequest represents the ability to extract the username from the web request, and the UserInSession is how the details on the current user are saved in session state (or cookies).
It is very likely that UserInRequest will be renamed and get bigger as it is all about the authentication. The UserInSession trait might stay like this, even if there become multiple
implementations.

It would be entirely possible not to have these two abstractions: the code in them could absolutely be put in the controller itself. I like though minimising the work the control has
to do, and making the controller composed of small things: each doing one job well, rather than being a God object.

{% highlight scala %} 
 trait UserInRequest {
  def userName(implicit request: Request[_]): String
}

trait UserInSession {
  def userName(implicit request: RequestHeader): Option[String]
  def addUserName(result: Result, userName: String)(
                  implicit request: RequestHeader): Result
  def clear(result: Result)(implicit request: RequestHeader): Result
}
 {% endhighlight %}

As well as this code, we need to set up the routes file 

{% highlight scala %} 
GET   /               controllers.NotaryController.index
GET   /who            controllers.LoginController.who
GET   /login          controllers.LoginController.login
GET   /logout         controllers.LoginController.logout
...
{% endhighlight %}

#Test before Refactoring

There are a few types of tests we are going to make.

* Model tests: like the SimpleUserSpec above
* Unit tests of the major Play components (like controllers)
* Full tests using a browser to check that the website actually works

## Unit tests

Let's look at how we make a unit test for controller. As can be seen from the controller above, the way that the controller stores login state, and gets login from requests have
been parameterised out. It would be quite easy to inject some mocks to allow this code to be tested. I have had a love/hate relationship with mock frameworks for over a decade.
My experience is that they test 'too much'. They test the method calls, and require quite a lot of setting up for each test. Debugging them is something I find particularly 
challenging: hence my preference for this simple approach


{% highlight scala %}
/** This class emulates cookies or other session state*/
class UserInSessionForTest extends UserInSession {
  var name: Option[String] = None
  def userName(implicit request: RequestHeader) = { name }
  def addUserName(result: Result, userName: String)(
    implicit request: RequestHeader): Result = {
    name = Some(userName)
    result
  }
  def clear(result: Result)(implicit request: RequestHeader): Result = {
    name = None
    result
  }
}
{% endhighlight %}

This following code tests the LoginController

{% highlight scala %}
class LoginSpec extends PlaySpecification with Results {
  val controller = new LoginControllerTrait[SimpleUser] with Controller {
    implicit val userLike: User[SimpleUser] = implicitly[User[SimpleUser]]
    val userInSession: UserInSessionForTest = new UserInSessionForTest
    val userInRequest: UserInRequest = SimpleUserInRequest
  }

  def checkResult(result: Future[Result]) = {
    status(result) must equalTo(OK)
    contentType(result) must beSome("text/plain")
    charset(result) must beSome("utf-8")
  }

  "The LoginController" should {
    "should respond to 'login' with message and username in session" in {
      val request = FakeRequest(POST, "/").
                    withJsonBody(Json.
                      parse(s"""{ "user": "someName", "password": "somePassword" }"""))
      val result = controller.login()(request)

      checkResult(result)
      contentAsString(result) must be equalTo ("Logged in as someName")
      controller.userInSession.name must be equalTo (Some("someName"))
    }

    "should respond to 'who' with username" in {
      controller.userInSession.name = Some("someName")
      val result = controller.who()(FakeRequest())

      checkResult(result)
      contentAsString(result) must be equalTo ("You are logged in as Some(someName)")
      controller.userInSession.name must be equalTo (Some("someName"))
    }

    "should respond to 'who' with username and username removed from session" in {
      controller.userInSession.name = Some("someName")
      val result = controller.logout()(FakeRequest())

      checkResult(result)
      contentAsString(result) must be equalTo ("Logged Out")
      controller.userInSession.name must be equalTo (None)
    }
  }
}
{% endhighlight %}
The really nice thing about this Unit test is that it mostly doesn't care about what the actual website looks like, or the security in the actual website.
Because of the decoupling of how we get the login credentials, and how they are stored, all test is actually checking is that the appropriate methods are called (Hence
the reason that Mocks are often used)

##Browser Tests

[The sample code in here](https://www.playframework.com/documentation/2.5.x/ScalaFunctionalTestingWithScalaTest) demonstrates integration tests with HtmlUnitFactory. 

{% highlight scala %}
class HelloWorldIntegrationTest extends PlaySpec  with OneServerPerSuite 
                             with OneBrowserPerSuite with HtmlUnitFactory {

  "The index page" must {
    "display Hello World" in {
       go to (s"http://localhost:$port/")
       pageSource mustBe "Hello World"
    }
  }
}
{% endhighlight %}

I tend to only use HtmlUnitFactory when I am not using cookies or javascript. The above test worked as it was very simple, but for most uses
I prefer to use a real browser, as the goal of these tests is to check that the code actually works in a browser. The login story is all about cookie management, so
unlike most test suites that I write it has OneBrowserPerTest instead of OneBrowserPerSuite. This does add considerably to the execution time, and while for most tests I
would be happy to just call 'logout' in a beforeTest code, I'm not so keen for that in the Login code itself.

One thing that is needed with the ChromeFactory is to visit [this page](https://sites.google.com/a/chromium.org/chromedriver/downloads) and get the latest ChromeDriver. I 
put it in the root of the project.
 
{% highlight scala %}
class LoginIntegrationSpec  extends PlaySpec with OneServerPerSuite with OneBrowserPerTest with ChromeFactory {

  "The login page" must {
    "diplay 'Logged In as" in {
      go to (s"http://localhost:$port/login?user=Phil&password=SomePassWord")
      pageSource must include("Logged in as Phil")
    }
  }

  "The logout page" must {
    "diplay 'Logged out if logged in before calling" in {
      go to (s"http://localhost:$port/login?user=Phil&password=SomePassWord")
      go to (s"http://localhost:$port/logout")
      pageSource must include("Logged Out")
    }
    "diplay 'Logged out even if not logged in" in {
      go to (s"http://localhost:$port/logout")
      pageSource must include("Logged Out")
    }
  }

  "The who page" must {
    "return None before logging" in {
      go to (s"http://localhost:$port/who")
      pageSource must include("You are logged in as None")
    }

    "return who after logging in" in {
      go to (s"http://localhost:$port/login?user=Phil&password=SomePassWord")
      go to (s"http://localhost:$port/who")
      pageSource must include("You are logged in as Some(Phil)")
    }

    "display None after log in and log out" in {
      go to (s"http://localhost:$port/who")
      pageSource must include("You are logged in as None")

      go to (s"http://localhost:$port/login?user=Phil&password=SomePassWord")
      go to (s"http://localhost:$port/logout")
      go to (s"http://localhost:$port/who")
      pageSource must include("You are logged in as None")
    }
  }
}
{% endhighlight %}

#Controller Unit Tests and Browser Tests

The unit tests and browser tests are in many ways very similar. What I find in general is that at the start of the project they tend to be sufficiently similar
that I often have concerns 'am I just writing the two sets of tests because it seems a good idea'. What I tend to find over the course of a project is that they 
drift apart. As more and more security strategies are implemented, hopefully the Unit Test will not dramatically change, while the Browser tests will. The Browser tests
are likely to get quite complicated while the Unit Tests checks that the wiring is all working and won't change much

#Summary

We've seen three examples of tests for a Play application: unit tests for a model, unit tests for Play Controllers, and browser integration tests. 

