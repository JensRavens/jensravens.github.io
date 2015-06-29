---
tags: [swift, monads, errorhandling]
series: Functional Reactive Programming in Swift
series-index: 5
title: The Signal, Threading and You.
abstract: Multithreading can be hard - let's make it a bit easier by using signals.
---

# The _Signal_, _Threading_ and _You_

Multithreading can be hard - let's make it a bit easier by using signals.

So far we've seen [how to implement your own signal]({% post_url 2015-06-15-sending-a-signal-to-space %})
and [why transforms are usefull]({% post_url 2015-05-24-transforming-the-world-into-a-better-place %}).
Now we can take a look at our first real world example: Transforming signals between different threads.

Remember the ensure method from the last post? This method transforms a signal,
even if it was not successfull. Let's define a transform that returns on another
thread:

```swift
class Thread {
  static func main<T>(a: Result<T>, completion: Result<T>->Void) {
      dispatch_async(dispatch_get_main_queue()){
          completion(a)
      }
  }
}
```

This method just executes a completion handler on the main Thread. It's pretty
basic and simple to use (note we're using the Swift 2 version without a box class):

```swift
let result = Result.Success("Hello")
Thread.main(result) { result in
    // result is now on main thread
}
```

But as this function conforms to the signal's `ensure` method, we can also use
that:

```swift
Signal("Hello")
.ensure(Thread.main)
.next { text in
    // use text on main thread
}
```

Putting this all together we can do some calculation on the background thread,
go back to main and then use the calculated value:

```swift
Signal("World")
.ensure(Thread.background)
.map { name in
    return "Hello \(name)"
}
.ensure(Thread.main)
.next { text in
    print(text)
}
```

You can find the implementation for `Thread.main` and `Thread.background` on the
[Github Repository](https://github.com/jensravens/interstellar). Now we have
everything we need to do our first example application in the next post.
