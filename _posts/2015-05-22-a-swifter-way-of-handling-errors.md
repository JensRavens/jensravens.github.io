---
tags: swift, monads, errorhandling
abstract: >
  Handling errors is something most developers try to avoid. But error handling
  got way more important on mobile devices where the user constantly switches
  apps, the network connection drops in the middle of a transfer or the
  application is terminated by memory pressure.
---

# A _Swifter_ Way of _Handling Errors_

Handling errors is something most developers try to avoid. Handling errors is no
fun. Coding _the happy path_ is way more entertaining. Who want's to handle an
error if he could implement new and shiny features instead? But error handling
got way more important on mobile devices where the user constantly switches
apps, the network connection drops in the middle of a transfer or the
application is terminated by memory pressure.

Every language and framework has it's own special best practices of error
handling and Swift is no exception to that rule. It's still supporting the old
ObjC way but also is kind of a blank slate to experiment with new ideas.

## Handling Errors the ObjC Way

Remember the good old days of pointers to errors in ObjC?

``` objective-c
NSError* error;
id json = [NSJSONSerialization JSONObjectWithData: data, options: 0, error: &error];
if(json){
  // success!
} else {
  // handle the error
}
```

Create an error, give your method call the pointer address and look if the error
if filled after the call. Most of the times. Sometimes the error is filled but
already has been handled by the system (quite common when working with Core
Data), so check the response value and if the method hasn't responed check the
error next (if that's new to you I strongly suggest to read the [Cocoa Error
Handling Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ErrorHandlingCocoa/ErrorHandling/ErrorHandling.html)).

Of course all that stuff is still available:

``` swift
var error: NSError?
if let json = NSJSONSerialization.JSONObjectWithData(data, options: nil, error: &error) as? NSDictionary {
    //handle data
} else {
    // handle error
}
```

This is one of the few cases you'll ever see an inout-parameter (the `&` next to
the error) to pass in a variable as a reference instead of a value. Compared to
other Swift APIs this feels clunky and old fashioned. So what are our
alternatives?

## This might work. Or not.

Swift introduced a special construct for values that might be there or not. This
construct is called `Optional<T>` and either contains a value or nil. So take a look
this json parsing function:

``` swift
if let json = NSJSONSerialization.JSONObjectWithData(data, options: nil, error: nil) as? NSDictionary {
    // handle json
} else {
    // handle error
}
```

This is just skipping the error parameter and relies on the method to return either
the parsed json or nothing at all. This is usefull for parts of the application
where you actually don't care why it fails, just if the operation was successfull.

## Two values are better than one

But what if you actually care about the reason for failure? Maybe there's a way
to recover from certain errors. Consider this enhanced version that uses tuples:

``` swift
func parseJson(data: NSData) -> (NSDictionary?, NSError!) {
    var error: NSError!
    if let json = NSJSONSerialization.JSONObjectWithData(data, options: nil, error: nil) as? NSDictionary {
        return (json, nil)
    } else {
        return (nil, error)
    }
}

let (json, error) = parseJson(data)
if let json = json {
    //handle data
} else {
    // handle error
}
```

This version uses tuples to return multiple values from a function (something
that wasn't possible before in ObjC) and a destructuring assignment to unpack it
into two variables. So we finally got rid of the inout parameter and still know
about the error. But there's an even better way.

## Defining a custom result type

The destructuring is still a bit implicit. What if we access the error before
checking for the value? This actually leads to a crash. Logically there's no
way for both values to be set. It's _either_ value or error. So if there are two
states this sounds like a nice use case for an enum, doesn't it?

``` swift
enum Result<T> {
  case Success(T)
  case Error(NSError)
}
```

But there is currently an issue (as of Swift 1.2) that forbids enums to hold a
generic value. So let's box things up:

``` swift
final class Box<A> {
  public let value: A

  public init(_ value: A) {
    self.value = value
  }
}

public enum Result<T> {
  case Success(Box<T>)
  case Error(NSError)
}
```

Now we can define an elegant solution to our error handling problem:

``` swift
func parseJson(data: NSData) -> Result<NSDictionary> {
    var error: NSError!
    if let json = NSJSONSerialization.JSONObjectWithData(data, options: nil, error: nil) as? NSDictionary {
        return .Success(Box(json))
    } else {
        return .Error(error)
    }
}

switch parseJson(data) {
case let .Success(box): //handle the dictionary (box.value)
case let .Error(error): //hande the error
}
```

Think of the result type as something similar to the `Optional`, just with an
attached error for the nil case. In my opinion this should actually be part of
the Swift standard library.

## Are we functional yet?

Although this post should be the first post in a series of posts about functional
programming in Swift, all mentioned techniques so far are not functional per se.
They just add a more explicit way to handle errors without introducing side
effects. Are they helpfull for functional programming? Sure. Can you use them if
you want to stay completely object oriented? Of course! Just know all the tools
out there so you can always pick the right tool for the job.

Next up we'll take a look at transforming values and how to chain transforms that
produce `Result`s.
