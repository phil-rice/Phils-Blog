---
layout: post
title: "Getting going with Scala Macros"
date:   2015-11-14 12:00:23 +0000
comments: True
categories:
- scala
- macros
---
I've used the macros introduced in Scala 2.10 extensively. Every time I have used them I have ended up with serious pains: they are remarkably hard to reason about, to manipulate and construct. One of the 
biggest problems for me is the amount of information you need to know about how the compiler works.

Fortunately there is a new approach to macros, and I thought I would try it out. I started by refreshing my memory on how macros work with [This overview](http://docs.scala-lang.org/overviews/macros/overview.html), 
and then took a look at the new approach while [heavily depends on Quasiquotes](http://docs.scala-lang.org/overviews/quasiquotes/intro.html). I found 
[this blog particularly interesting](http://imranrashid.com/posts/learning-scala-macros/)

[The code for this can be found here](https://github.com/phil-rice/ScalaMacroBlog)

#Scala string interpolation

Before we go into quasi quotes it helps if [we understand string interpolation](http://docs.scala-lang.org/overviews/core/string-interpolation.html). I have used similar things in ruby in the past and have
missed them in Scala and Java. Scala 2.10 introduced them, which made me a lot happier. Let's compare the following two printlns
{% highlight scala %} 
 object StringInterpolation {

  def main(args: Array[String]): Unit = {
    val a = 1
    val b =2 
    print("The value of a is " + a + ", while the value of b is " + b)
    print(s"The value of a is $a  while the value of b is ${b}")
  }
}
{% endhighlight %}
The $a in the string prefixed with an s means 'put the toString of the variable a in the string. The ${<some code>} means evaluate the code between the {} and do a toString on the result, and stick that in the string

These are very useful, and I use them a lot. Quasiquotes are an extension of the same code base

#Quasiquotes

I started by following the [guide]( http://docs.scala-lang.org/overviews/quasiquotes/intro.html)
{% highlight scala %} 
object QuasiQuotes {
  def main(args: Array[String]): Unit = {
    val tree = q"i am { a quasiquote }"
    println(tree)
  }
}
{% endhighlight %}
And... it didn't compile. 'value q is not a member of StringContext'. A little google fu lead me to adding the following to build.sbt

>libraryDependencies += "org.scala-lang" %% "scala-reflect" % "2.11.7"

About now I have two things in the build.sbt that specify the scala version. I am tempted to move to a better way of doing the build file so that I can express this with a variable. This
code is just 'playing around though', so I'll leave it for now. In any event it wasn't enough. But this worked:
{% highlight scala %} 
object QuasiQuotes {

  val universe: scala.reflect.runtime.universe.type = scala.reflect.runtime.universe
  import universe._

  def main(args: Array[String]): Unit = {
    val tree = q"i am { a quasiquote }"
    println(tree)
  }
}
{% endhighlight %}
On my console I ended up with   
{% highlight scala %} 
i.am(a.quasiquote)
{% endhighlight %}
What this is, is a pretty printed abstract syntax tree of the code i.am(a.quasiquote). 'i' represents some variable or method or object called i. Similarly with 'a'. The words 'am' and 'quasiquote' are method calls.
Of course none of these things exist. There are no variables called 'i' and 'a', and they certainly don't have these methods! 

#What is an Abstract Syntax Tree (AST)?
These words are used a lot in the documentation. In essence they are the domain model for code. If you have a line of code 'if(a) b else c' then this would be modeled as a tree with an if statement
at the top. That if statement would have a condition, a then and an optional else. The condition in this case is an identifier with the value of the string 'a'. A good summary (as every) 
[is on wikipedia](https://en.wikipedia.org/wiki/Abstract_syntax_tree) 

One way of thinking of code is as 'a way of describing an abstract syntax tree so the compiler can turn it into byte codes'. What macros let us do is replace one node of the AST with the contents of the macro.
The macro 'executes' during compile time, and returns a (suitably typed)  AST that is spliced into the rest of the compiled code. One really nice thing about it (perhaps the most useful singlet thing) is
that the parameter to it are NOT evaluated and passed in. Instead the AST that represents them is passed in. A use for this can be seen in the FunctionalWrapper example below  

#Hello World Macro
{% highlight scala %} 
import language.experimental.macros
import reflect.macros.Context

object LogMacro {
  def hello(): Unit = macro helloImpl

  def helloImpl(c: Context)(): c.Expr[Unit] = {
    import c.universe._
    reify { println("Hello World!") }
  }

  def main(args: Array[String]): Unit = {
    hello() //doesn't work
  }
}
{% endhighlight %}
This is a pretty simple macro that [I coped from here](http://www.warski.org/blog/2012/12/starting-with-scala-macros-a-short-tutorial/) I first coded it up in about 2012. When we compile it 
we get a deprecation warning, which I wasn't expecting! Chasing that deprecation warning leads us to 

>In Scala 2.11, macros that were once the one are split into blackbox and whitebox macros,
>with the former being better supported and the latter being more powerful. You can read about
>the details of the split and the associated trade-offs in the [[http://docs.scala-lang.org/overviews/macros.html Macros Guide]].
>
>`scala.reflect.macros.Context` follows this tendency and turns into `scala.reflect.macros.blackbox.Context`
> and `scala.reflect.macros.whitebox.Context`. The original `Context` is left in place for compatibility reasons,
>but it is now deprecated, nudging the users to choose between blackbox and whitebox macros.

There is a [nice discussion on this point here](http://docs.scala-lang.org/overviews/macros/blackbox-whitebox.html). The essence of the discussion is that if you want to do
powerful things whitebox macros are really good, but if you can get away with a blackbox context that's a good idea.  We aren't going to be messing with whitebox macros for a while (if ever), 
so we change the code to import reflect.macros.blackbox.Context.

We still have a compilation issue: Macros can't be executed in the  same compilation unit that defines them

#Splitting the project into two projects

SBT allows multiple projects in the same build unit. While I was doing the plumbing for that, I sorted out the 'scala version number' so that I only have to define it in one place. build.sbt is now
{% highlight scala%} 
import Dependencies._

lazy val commonSettings = Seq(
  version := "0.1.0",
  scalaVersion := "2.11.7",
  javacOptions ++= Seq("-source", "1.8", "-target", "1.8"),
  libraryDependencies ++= commonDependencies
)
  
lazy val macroDefn = (project in file("macroDefn")).
                     settings(commonSettings: _*)

lazy val macroUser = (project in file("macroUser")).
                     settings(commonSettings: _*).
                     dependsOn(macroDefn)

lazy val root = project.aggregate(macroDefn, macroUser)
{% endhighlight %}
The object Dependencies is defined in a file Dependancies in the project subdirectory
{% highlight scala%} 
import sbt._

object Dependencies {
  lazy val scalaVersionNo="2.11.7"
  
  lazy val commonDependencies = Seq(
     "org.scala-lang" % "scala-reflect" % scalaVersionNo,
     "org.scalatest" %% "scalatest" % "2.2.4" % "test"
  )
}
{% endhighlight %}
I then needed to move scala files around. I ended up with the following file structure:
{% highlight scala%} 
scalamacroblog
    macroDefn
       src/main/scala/org/validoc/scalaMacroBlog
          HelloWorldOldMacro.scala
    macroUser
       src/main/scala/org/validoc/scalaMacroBlog
          HelloWorldOldMacroUser.scala
{% endhighlight %}
File HelloWorldOldMacro looks like
{% highlight scala%} 
import language.experimental.macros
import reflect.macros.blackbox.Context

object HelloWorldOldMacro {
  def hello(): Unit = macro helloImpl

  def helloImpl(c: Context)(): c.Expr[Unit] = {
    import c.universe._
    reify { println("Hello World!") }
  }
}
{% endhighlight %}
File HelloWorldOldMacroUser.scala looks like this. Importantly there is no perceptible way of seeing that 'hello' is a macro rather than a method
{% highlight scala%} 
object HelloWorldOldMacroUser {
  def main(args: Array[String]): Unit = {
    HelloWorldOldMacro.hello() 
  }
}
{% endhighlight %}
We run HelloWorldOldMacroUser and voila! We get Hello printed on the console

#What's wrong with the old macros? 

Well here is an example from something I've done before
{% highlight scala%}
  def becauseImpl[P1: c.WeakTypeTag, P2: c.WeakTypeTag, P3: c.WeakTypeTag, 
       R: c.WeakTypeTag, FullR: c.WeakTypeTag](c: Context)(
       because: c.Expr[(P1, P2, P3) => Boolean]): 
                   c.Expr[Builder3[P1, P2, P3, R, FullR]] = {
    import c.universe._
    reify {
      val l = bl[P1, P2, P3, R, FullR]()
      val ch = CodeHolder[((P1, P2, P3)) => Boolean]((p123) => 
                 because.splice(p123._1, p123._2, p123._3), 
                 c.literal(show(because.tree)).splice)
      val thisObject: Builder3[P1, P2, P3, R, FullR] = 
                 (c.Expr[Builder3[P1, P2, P3, R, FullR]](c.prefix.tree)).splice
      thisObject.becauseHolder(ch)
    }
  }
{% endhighlight %}
And... what is going on? There is so much code in that reify! Even worse it took me about two weeks to write it: I had to ask questions on Stack overflow, chase down source code
and work out some unpleasant compiler details. And... it's quite verbose! There are also horrible and complicated issues with types that are 'unnecessary complixity'. The new approach
using quasiquotes seems to remove a lot of that pain

#Hello World with Quasiquotes
In the macroDefn project we have
{% highlight scala%}
import scala.reflect.macros.blackbox.Context
import scala.language.experimental.macros

object QuasiQuotes {

  val universe: scala.reflect.runtime.universe.type = scala.reflect.runtime.universe
  import universe._

  def helloWithQuasiquotesImpl(c: Context): c.Expr[Unit] = {
    import c.universe._
    val value = q"""println("Hello")"""
    c.Expr(value)
  }

  def hello:Unit  = macro helloWithQuasiquotesImpl
}
{% endhighlight %}
In the macroUser project we have 
{% highlight scala%}
object QuasiQuotesUser {
  def main(args: Array[String]): Unit = {
    QuasiQuotes.hello
  }
}
{% endhighlight %}
If we run the QuasiQuotesUser main method 'Hello' is printed to the console

#What's going on

Let's break the code down a little. We start with the place where we declare 'there is a macro called hello'.

{% highlight scala%}
  def hello: Unit  = macro helloWithQuasiquotesImpl
{% endhighlight %}
 This macro has no parameters, and returns unit. We'll see shortly how we deal with parameters. As well as declaring the signature (no parameters and returning Unit) this line of code also states 
 where you go to find the macro implementation. In this case the method helloWithQuasiquotesImpl

Now let's look at the implementation of the macro. 
{% highlight scala%}
  def helloWithQuasiquotesImpl(c: Context): c.Expr[Unit] = {
    import c.universe._
    val value = q"""println("Hello")"""
    c.Expr(value)
  }
{% endhighlight %}
First we have 'stuff we have to do'. The 'c: Context' needs to be passed into the macro code. This contains all sorts of compiler goodness. Fortunately 
we don't need to worry too much about the compiler stuff. The result of the macro is going to be 'Unit', so the macro implementation has to have a return type 
of "c.Expr[Unit]": compilers errors will ensue if we don't get this correct. 

Once we have the declaration sorted, we need to provide all the plumbing to allow the quasiquotes to produce things in the 'right context'. That's done by the "import c.universe._". 

The quasiquotes then allow us to make an abstract syntax tree representing the code we want to replace the macro call, and c.Expr turns that AST into the result type expected by the macro

#Nice function toString with Macros

The reason I turned to macros was that I want to have pretty toStrings for my closures. Which is nicer? 

* &lt;function1&gt;
* (i: Int) => i.*(2) 

as the toString of (x: Int)=>i*2? I think the answer is obvious

It turns out to be fairly easy to do this. As I mentioned before the parameter passed to the macro is not 'evaluated' but instead the AST of it is passed in. Let's look at the following code and see how we can use this
{% highlight scala%}
import scala.reflect.macros.blackbox.Context
import scala.language.experimental.macros
object FunctionWrapper {

  implicit def apply[P, R](fn: P => R): FunctionWrapper[P, R] = macro apply_impl[P, R]

  def apply_impl[P: c.WeakTypeTag, R: c.WeakTypeTag](c: Context)(fn: c.Expr[P => R]): 
                  c.Expr[FunctionWrapper[P, R]] = {
    import c.universe._
     c.Expr(q" new FunctionWrapper($fn, ${show(fn.tree))})")
  }
}
class FunctionWrapper[P, R](val fn: P => R, description: String) 
      extends Function1[P, R] {
  def apply(p: P) = fn(p)
  override def toString = description
}
{% endhighlight %}
An example of it in use is:
{% highlight scala%}
object FunctionWrapperUser {
  def main(args: Array[String]): Unit = {
    val double = FunctionWrapper((i: Int) => i * 2)
    println(s"double to String is $double")
    println(double(4))
  }
}
{% endhighlight %}
This prints out the "(x: Int)=>i*2" that we want to see, and as well as that the double(4) prints out 8.

In the same way we looked at the hello macro, let's look at FunctionWrapper.apply. The macro definition is clear.
{% highlight scala%}
 implicit def apply[P, R](fn: P => R): FunctionWrapper[P, R] = macro apply_impl[P, R]
{% endhighlight %}
 We pass in a Function from P to R, and implementation is done by apply_impl
 
{% highlight scala%}
def apply_impl[P: c.WeakTypeTag, R: c.WeakTypeTag](c: Context)(fn: c.Expr[P => R]): 
                                c.Expr[FunctionWrapper[P, R]] = {
    import c.universe._
    c.Expr(q" new FunctionWrapper($fn, ${show(fn.tree)})")
}
{% endhighlight %}
Here we see how to specifiy generics. In the 'apply' we have P as the parameter and R as the result of the function. In the macro implementation we need to define those. "[P: c.WeakTypeTag, R: c.WeakTypeTag]" is the 
magic that does that. Again we need to "import c.universe._" to ensure that the quasiquotes are in the correct context. The quasiquotes make a new FunctionWrapper passing the fn into it. As well as the fn
they pass in a string that is the 'tostring' of the fn. The code 'show(fn.tree)' turns fn's AST into a pretty string.

#Summary

Macro's are a useful tool. They are a way to do things that you can't otherwise do in a language. I don't know anyother every half reliable way of getting the toString of a function to be realibly
made from it's abstract syntax tree. I hope this example showed that useful macros don't have to be horribly complicated to write. I use the FunctionalWrapper
approach quite a lot when trying to produce nice error messages and reports from deep inside code. 

