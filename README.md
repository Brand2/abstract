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
So, let's say we're designing some API with a number of endpoints, which are handled by the functions (attached to some generic `Server` type) below (***NOTE:*** I have elected to follow the [gRPC](https://grpc.io/)-style method of defining services, as this could also be functions in some greater application called internally passing a pointer to an object):

```go
func (s *Server) getCatByRequest(ctx context.Context, r *SomeRequest) (*Cat, error) {
  // Assume we do nothing with ctx, SomeRequest is defined elsewhere, and we aim to
  // return either a ptr to a Cat instance or an error
  // 
  // Let's start by assuming we're using a MongoDB server, accessed through the mgo
  // package - Establish a session, access a database and a collection in there
  session, err := mgo.Dial("someurl")
  if err != nil {
    // Handle error if you've been given one
  }
  // Let's set our output var cat AND our collection
  cat := &Cat{}
  coll := session.DB("somedb").C("cats")
  // Here's where we would construct our query using parameters from r, but for brevity
  // let's skip that... and define "query" as some constructed query using bson.M
  if err = coll.Find(query).One(cat); err != nil {
    // Handle any errors
  }
  return cat, nil
}
```
And we would have similar for say `getCatsByRequest` (returning an array of `Cat` objects), `getDogByRequest` and `getDogsByRequest` and so on. Now, I skipped a lot of the code above, but including the construction of queries, this becomes a lot of code to put in a single function. For starters we might want to move our MongoDB session to live as a persistent object in a field attached to our `Server` type. Further, we could wrap our query construction, results retrieval etc. in a function which is called in every instance (taking our request object, `r` as a parameter), though this may not be straight forward and might contain a fair number of conditional statements (either as a `switch` or `if-else` block, depending on how things branch depending on query construction etc.). Whilst this is obviously not perfect, it does make it slightly easier to maintain! Now instead of having to change 4 places if we want to change logic which is common, we change one. Let's consider now if we add mapreduce jobs, this is even greater complexity! Finally, consider this: We have an awful lot of platform/tech. specific code in the main libraries of our application. What happens if we want (or more likely, need) to change this? We suddenly have a major rewrite on our hands! Further, if we re-write this code our new version is no longer compatible with our old version (unless we add more code... which quickly becomes unmaintainable).

### So, let's abstract
Now, we might find ourselves in a bit of a proverbial pickle here, as we find ourselves with code permeated with lines specific to a particualr kind of backend, when suddenly we realise we're going to have to change it! Or perhaps the data structure/types! So, let's consider a few cool tools we have up our sleeves as part of programming languages, in Go we can define an `interface`, so how about we define an `interface` which does a few things; let's call it `Backend` living within `package backend`.

```go
package backend

// Backend represents a connection to a non-specific backend, any backend which
// is to be used with our system must implement this interface
type Backend interface {
  // Connect initiates a "connection" to our backend
  // - This may be a server connection, opening a file
  // or any number of other options, it will return
  // nil if successful or an error otherwise
  Connect() error
  
  // Disconnect closes the connection previously established
  Disconnect()
}
```

So, we have a very basic interface defined, which let's us define multiple *types* of backend in an abstract way. As an example, let's assume we have a `Kennel` type which is an implementation of `Backend` (whoever wrote it liked dogs!) and a `Cattery` which is also an implementation of `Backend` (because, why not?) Now, the nature of interfaces means that if we have a function like

```go
func DoSomethingWithBackend(b *Backend) error {
  // We do something here
}
```

Then both a `*Kennel` **and** a `*Cattery` could equally be passed to this function! Equally, if we have a struct containing a number of fields, we might want to declare one a `Backend` like the following

```go
type Server struct {
  /**
   * Extra fields go here
   */
  BackendField backend.Backend
}
```

Which means we can slot whatever backend we like into our front-facing server! Of course, this doesn't *quite* solve our problem yet, as we have no way of initialising or querying our backend meaningfully! So, let's start by expanding our earlier definition of the `Backend` interface a little to include a few methods to do our queries.

```go
type Backend interface {
  // Functions as before up to this point...
  
  // GetObjectByField lets us pass some kind of field
  // to our backend, and encapsulate the logic to do
  // the query
  GetObjectByField(interface{}) (ReturnType, error)
  
  // Do similar to get multiple objects, store them
  // and so on
}
```

However, we still run into the issue that realistically, `GetObjectByField` (and similar functions) only return a single type.

```
Well, this technically isn't true in Go, as everything technically implements the empty interface (interface{}) so this allows
us to return (or input) arbitrary types.
However I don't really like that approach as it doesn't let us check that our return types are allowed within our system
```

So, to get around this I propose we create a `RegisteredType` interface which is implemented by every data type we want returned through our API. So our `backend` package begins to look something like.

```go
type Backend interface {
  // Functions as before up to this point...
  
  // GetObjectByField lets us pass some kind of field
  // to our backend, and encapsulate the logic to do
  // the query - Logic to check that inputs are allowed
  // also captured here
  GetObjectByField(interface{}) (RegisteredType, error)
  
  // Do similar to get multiple objects, store them
  // and so on
}

type RegisteredType interface {
  // Put functions here we would like to
  // implement on all data types we would
  // like to return from the backend
}
```

***NOTE:*** I basically took this approach from the implementation of [`protobuf`](https://godoc.org/github.com/golang/protobuf/proto) (i.e. the `proto` package) for Go.

However, we still don't have a way of registering these abstractions, or even initialising/creating them when we start our application, so let's rectify that. Adding again to our `backend` package.

```go
// InitialiseBackendSystem is implemented by each backend type seperately, and represents
// a way of creating a valid backend, it should panic if this is not possible (i.e. the app
// will not start)
type InitialiseBackendSystem func(map[string]interface{}) (backend Backend, err error)

// gBackends is a global map containing all of the possible backends currently registered
// and thus available for use from the frontend
var gBackends = make(map[string]InitialiseBackendSystem)

// These two vars represent a registry for tracking all data types which our application may
// deal with - Note the use of the reflect package to store the exact data type at runtime
var (
  RegTypes = make(map[string]reflect.Type)
  RevRegTypes = make(map[reflect.Type]string)
)

// Finally a R/W mutex variable to ensure our maps are not written to concurrently
var gBackendMutex sync.RWMutex

// RegisterBackend registers our backend with a given name, in this case if a name is already
// taken at the time of registration, that registration will be overwritten
func RegisterBackend(name string, fn InitialiseBackendSystem) {
  if fn == nil {
    // No implementation of the initialising function, return
    return
  }
  
  gBackendMutex.Lock()
  defer gBackendMutex.Unlock()
  
  gBackends[name] = fn
}

// CreateBackend creates a backend using an input name, generally used by code higher in our
// application
func CreateBackend(name string, args map[string]interface{}) (backend Backend, err error) {
  gBackendMutex.RLock()
  defer gBackendMutext.RUnlock()
  
  // Retrieve our initialisation function
  fn := gBackends[name]
  if fn == nil {
    // No func found, raise an error
    return nil, errors.New("Unable to create new backend; backend type unknown")
  }
  
  // Let's run the initialisation function and return the results
  return fn(args)
}

// RegisterType stores the data type as part of the system, it also serves as
// a form of check to ensure the type implements RegisteredType and that no two
// types have the same name
func RegisterType(r RegisteredType, name string) {
  if _, ok := RegTypes[name]; ok {
    // You've already got something registered under this name, print that and return
    log.Printf("A type is already registered with the name %s\n", name)
    return
  }
  t := reflect.TypeOf(r)
  RegTypes[name] = t
  RevRegTypes[t] = name
}
```

Now we have all of the abstract tools we need to refer to this code from a higher level struct or function! Even better, this is now backend-agnostic, as we can refer to a `Backend` implementation, which may be for *MongoDB*, *Redis*, *Bolt* or any other storage/indexing system available! And because our higher level code will only ever refer to `Backend` and its attached functions/methods, we can extend, replace, and plug different backends (and data types) in and out as we need! This is a huge boon as now we could equally test things small scale with a simple databasing system (i.e. during **development**, and have the same higher level logic for our API apply when we implement our **production**-level backend systems, it's all hidden behind our abstraction (which can be slotted nicely into whatever variables and/or fields are appropriate at the higher level of the application)! Further, if we hand this code off to someone else, and they feel more comfortable maintaining a different backend system, they just have to write a new plugin! Finally, we've also reduced them amount of code we have to maintain significantly, and hopefully minimised our chances of ending up with the dreaded ***spaghetti code*** by reducing the amount of places in which our code links, and ensuring that once the high level API is finalised, the primary location for maintenance and/or changes is likely the `Backend` implementation specific to a system.

As an example of usage, a backend would be registered by implementing `InitialiseBackend` and then calling `RegisterBackend` such as

```go
// Where initialiseBackend is the internal implementation of InitialiseBackend
// for a specific Backend
backend.RegisterBackend("mongo", initialiseBackend)
```

And then from the top level of the API, let's assume we have a config file which is loaded into a `Config` object, we can have a field where we set a backend type i.e. `Config.BackendType`, now we can just change a single config file to change which type of backend we are using! Assuming we called our global config object in the main loop `gConfig`, and it stored all the arguments for the `InitialiseBackend` function as `Config.BackendArgs` or similar we would have

```go
// Top level declaration - This doesn't implement a specific backend, it's just our
// abstracted interface!
var gBackend backend.Backend

// ... Now in the main loop
gBackend, err = backend.CreateBackend(gConfig.BackendType, gConfig.BackendArgs)
```

I hope this was worth a read, and explained a little bit about the how and why of abstraction, especially in the space of databasing/data handling systems.
