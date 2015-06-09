---
tags: [swift, monads, errorhandling]
series: Functional Reactive Programming in Swift
series-index: 3
abstract: |
  Let's put all the previous stuff together to achieve our goal: Chaining
  synchronous transforms with error handling unix style.
---

# How to _Train_ your _Monad_

I've already shown you how to
[handle errors]({% post_url 2015-05-22-a-swifter-way-of-handling-errors %})
and why [transforms are great for application architecture]({% post_url 2015-05-24-transforming-the-world-into-a-better-place %}).
Let's put all that stuff together to finally achieve our goal: Chaining
transforms with error handling.

## A short recap of our goal

We wanted to achieve a way of programming that is similar to the unix pipe:

```bash
ls | grep *.jpg | sort
```

Methods in this way of programming should have the following characteristics:

1. We don't tell them how to do it, we tell it what we want instead.
2. Every command has a well defined interface (`stdin`, `stdout`, `errout`).
3. Commands are chainable.
4. Error handling is implicit.

Also we defined 4 method signatures for transforms:

```swift
// synchronous, non failing
func transform<A,B>(value: A)->B

// async, non failing
func transform<A,B>(value: A, completion: (B->Void))

// synchronous, failable
func transform<A,B>(value: A)->Result<B>

// async, failable
func transform<A,B>(value: A, completion: (Result<B>->Void))
```

## Adding the missing `map`

We've already implemented the synchronous `map` function on the `Result<T>`.
Let's add the asynchronous one:

```swift
public func map<U>(f:(T, (U->Void))->Void) -> (Result<U>->Void)->Void {
  return { g in
    switch self {
    case let .Success(v): f(v.value){ transformed in
        g(.Success(Box(transformed)))
      }
    case let .Error(error): g(.Error(error))
    }
  }
}
```

This method takes an asynchronous non failing transform (the 2nd one in our list)
and returns a function that can be invoked with a completion block to get the
result once it's completed like in this example:

```swift
func toHash(string: String, completion: Int->Void) {
  completion(count(string))
}

Result.Success(Box("Hello World")).map(toHash)(){ result in
  //result is now a .Success of Int with the value 11
}
```

This might feel a bit clunky at first, but the advantages are pretty obvious as
soon as we start to chain synchronous transforms that can fail.

## Failable Transforms

Let's consider applying a failable transform to a `Result<T>`:

```swift
func toInt(string: String)->Result<Int>{
  if let int = string.toInt() {
    return .Success(Box(int))
  } else {
    return .Error(NSError())
  }
}

Result.Success(Box("Hello World")).map(toInt)(){ result in
  // result is now of type Result<Result<Int>>
}
```

Mapping to a result of a result is actually pretty useless. In the end we'd
prefer to either have success or a failure. Most languages call this feature
`flatmap` (because it first maps and then flattens the result), `fmap` or `bind`.
I'll go with `bind` here.

```swift
public func bind<U>(f: T -> Result<U>) -> Result<U> {
  switch self {
  case let .Success(v): return f(v.value)
  case let .Error(error): return .Error(error)
  }
}
```

If the result is a success, the next function is executed, if it's a failure the
error is returned immediateley. The previous example now looks like this:

```swift
Result.Success(Box("Hello World")).bind(toInt)(){ result in
  // result is finally of type Result<Int>
}
```

This looks a lot nicer. There is still one transform missing: The async failable
transform:

```swift
public func bind<U>(f:(T, (Result<U>->Void))->Void) -> (Result<U>->Void)->Void {
  return { g in
    switch self {
    case let .Success(v): f(v.value, g)
    case let .Error(error): g(.Error(error))
    }
  }
}
```

This method returns a completion handler that can be used to grab the result once
it's completed:

```swift
func toInt(string: String, completion:Result<Int>->Void){
  if let int = string.toInt() {
    completion(.Success(Box(int)))
  } else {
    completion(.Error(NSError()))
  }
}

Result.Success(Box("Hello World")).bind(toInt)(){ result in
  // result is again of type Result<Int>
}
```

## Method Chains

Now we can finally do our unix example:

```bash
ls | grep *.jpg | sort
```

```swift
let ls = Result.Success(Box(["/home/me.jpg", "home/data.json"]))
let grep: String->([String]->Result<[String]>) = { pattern in
    return { paths in
        return .Success(Box(paths))
    }
}
let sort: [String]->Result<[String]> = { values in
    return .Success(Box(sorted(values)))
}

let chain = ls.bind(grep("*.jpg")).bind(sort) //Result<Array>
```

Let's revisit our wishlist:

### 1. We don't tell them how to do it, we tell it what we want instead.

We've written small, composable functions that can be chained together. They're
reusable and easily testable.

### 2. Every command has a well defined interface

All tansforms take a value, manipulate a copy and either returns a `.Success` or
an `.Error`.

### 3. Commands are chainable.

By using map and bind wie can chain functions and get a result (this is not yet
working for async transforms - we'll do something about that in the next post).
By using generics we can be sure that only matching functions can be concatenated
(this is actually an advancement over the unix implementation).

### 4. Error handling is implicit.

If an error happens during a transform, all further transforms are skipped and
an `.Error` is returned instead. There is no chance to forget an error handling
branch (and no "I'll do this later `//Fixme:`").


## The monad and you

What we've just implemented is also called a monad. A monad is a thing that as a
constructor and that defines bind. If you want to read more about monads, burritos
and boxes take a look at [fuckingmonads.com](http://fuckingmonads.com). Also there
is a great explanation at
[Functors, Applicatives, And Monads In Pictures](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html).

## A deeper look into space

We've seen a great application for synchronous bind methods - but handling async
stuff is still missing. In the next chapter we'll take a look at `Signal<T>` and
how to tame callback hell. In the end you will have seen the full implementation
of [Interstellar](https://github.com/JensRavens/Interstellar), the reactive
programming framework. We're almost there!
