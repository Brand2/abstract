# Abstraction makes the heart grow fonder?
*Or how I learnt to love the weirdness.*

### What is the purpose of this? What am I talking about?
Hello everyone, I'm writing this having just completed one of the weirder exercises I've had to do in coding, and
have developed what I would regard as one of the most important skills I didn't know I needed. As a bit of pre-amble,
let's describe the problem and set the stage a little.

In many problems we will be faced with the question: ***How do we handle our data?*** and equally: ***How do we store our data?***

It would be nice if there were a simple answer to these, but the fact of the matter is that there is a plethora of different
storage options and solutions for all manner of problems, scales and data types. These range from the small key-value styles,
often capable of running literally in-memory such as [Redis](https://redis.io/) or [Bolt](https://github.com/boltdb/bolt), to
moderate size such as single server [Mongo](https://www.mongodb.com/) or [Couch](http://couchdb.apache.org/) and [Postgres](https://www.postgresql.org/),
finally through to sharded versions of Mongo, Couch or Postgres, or the ever-popular [HDFS](http://hadoop.apache.org/) a.k.a Hadoop
used by many large organisations, particularly those in the business of "big data". You could ever design your own indexing system
and use standard file I/O operations!

Now, this post isn't about how to choose which system is right for your application/solution, as I think that problem is almost entirely
too large to tackle in a general manner and so that decision must be left up to the reader. Whilst there is a seemingly alarming number
of different storage solutions, it is possible to find the one most suited to your situation with a little work.

Of course, anyone who's worked with data will probably tell you that your storage backend, and especially your data format/representations
are liable to change throughout the course of a project, at least that has been the experience of the author. So, the problem I'm here to
talk to you about is: How do we handle the changes that are likely to happen throughout the lifecycle of a project to our data ***types***,
***formats*** and choice of ***backend*** storage system (as the scope of a project potentially outgrows the viable scope of the original
system(s)).

As a developer, it becomes important to be able to write your code such that it is as ***agnostic*** as possible to the specifics of
the platform or system on which you find yourself running. Not only will this save your sanity a great deal of distress, but you will
also find you have more easily *maintainable* and *extensible* code. This is where the concept of abstraction comes in (***NOTE:*** I am
aware that none of the techniques discussed here are particularly ground-breaking or new, this has been written for the benefit of myself,
colleagues, and anyone else who might be tackling similar problems).

In order to discuss this further, I will be utilising [Go](https://golang.org) a very popular programming language. These concepts however
apply in many other languages (and infact would be even more powerful in languages with [generics]( such as C/C++)

#### So, why Go?
I had to tackle this problem in Go for a number of reasons recently, so it is fresh in my mind. However, I also consider Go a great
language in terms of readability and ease-of-learning. If you are unfamiliar with the syntax, a tour of the language is available at
the link [here](https://tour.golang.org/welcome/1) and covers the majority of its specifics, though the concepts discussed here are
the important part, not the specifics of implementing them.

I will likely also make reference to Google's [Protocol Buffers](https://developers.google.com/protocol-buffers/), which are an easy way of defining quick data formats which are extensible, though this applies to any sort of data.

### Let's start with a problem
So, let's say we're designing some API 
