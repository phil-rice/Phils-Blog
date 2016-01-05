---
layout: post
title: "Real World Play - Uploading files"
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
* [First Views and CSS added here](/scala/play/refactoring/2015/11/23/real-world-play-views-and-css.html)
* [tested again here](/scala/play/refactoring/2015/11/28/real-world-play-testing-part-2.html) 


The code for at the start of this can be found here [https://github.com/phil-rice/notary3](https://github.com/phil-rice/notary4) and at the 
final code can be found [https://github.com/phil-rice/notary4](https://github.com/phil-rice/notary5) 

#Review

At the moment we have a small Scala Play application with a few modules. There are two controllers: one does a very simple login story, the other
displays an index page with a text area. When the form containing that text area is submitted the digest for that text is displayed.

#Goals

While this isn't the world's most exciting website, it does actually perform a useful function. It would though be much more useful in general if files could be uploaded and
digested. It would be even better if 'have I seen this digest before' functionality was added. This allows us to actually add as a digital notary. As the notary we don't need 
to actually keep a copy of the actual file, which reduces our storage cost dramatically. Our business people however tell us that some customers will pay extra if we do keep a 
copy of that file, so we will add a property to the user, and if that is set we will save the file as well as the metadata about it.

One thing to bear in mind is the cost of resources. The more memory and harddrive space we consume, the more expensive the server we need to pay for. Some files can be very big: tens of gigabytes are 
the largest our customer is expecting people to use. We will thus want to make sure that we don't have to load 'all the file' into memory before we perform operations on it. We will
also want to put caps on the size of the file that we can store for any customer, otherwise we might accidentally find ourselves being asked to process a petabyte file or something

#Required reading
I found these this very helpful [ScalaFileUpload](https://www.playframework.com/documentation/2.4.x/ScalaFileUpload). There is a rather scary statement at the end of it 

> If you want to handle the file upload directly without buffering it in a temporary file, 
> you can just write your own BodyParser. In this case, you will receive chunks of data that 
> you are free to push anywhere you want.

This for me is like reading a statement in a mathematical proof 'clear it can be seen that'. This is however what we want: there is no real upside in us using temporary
files, and plenty of downsides (disc access, resources consumed...) 

## Iteratees
Understanding these turned out to be important. They are actually (like most things when you get them) quite simple. [This blog covers why you want them, and why they are a good idea, in some detail](https://jazzy.id.au/2012/11/06/iteratees_for_imperative_programmers.html)
[The Scala play documentation is quite helpful if you just want to work with them](https://www.playframework.com/documentation/2.4.x/Iteratees).

##Body Parser
So this documentation on [Body
N1Parsers](https://www.playframework.com/documentation/2.4.x/ScalaBodyParsers) was 
useful to explain what was meant by a body parser, but not great in expressing how to write one. [That task was met for me by this](http://manuel.bernhardt.io/2013/10/21/reactive-golf-or-iteratees-and-all-that-stuff-in-practice-part-2/)



#Getting going using temporary files
Although I said we aren't going to use temporary files, and are going to make our 
    /**
     * Store the body content into a temporary file.
     */
    def temporaryFile: BodyParser[TemporaryFile] = BodyParser("temporaryFile") { request =>
      Iteratee.flatten(Future {
        val tempFile = TemporaryFile("requestBody", "asTemporaryFile")
        file(tempFile.file)(request).map(_ => Right(tempFile))(play.api.libs.iteratee.Execution.trampoline)
      }(play.core.Execution.internalContext))
    }