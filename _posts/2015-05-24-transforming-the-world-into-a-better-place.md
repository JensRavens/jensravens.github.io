---
tags: swift, monads, errorhandling
abstract: >
  If you ask a developer what a program is he is likely to respond: A sequence of
  commands. Value oriented programming takes a step back and lays the focus on
  something different: The content of your application instead of the actual
  commands.
---

# Transforming the world into a better place

If you ask a developer what a program is he is likely to respond: A sequence of
commands. Value oriented programming takes a step back and lays the focus on
something different: The content of your application instead of the actual
commands.

Let's look at a basic unix example:

```bash
ls | grep *.jpg | sort
```

This lists the current directory, filters everything except jpgs out and then
sorts all those pictures by name. Pretty elegant, isn't it?

So what's the difference to programming as we're used to?

1. We don't tell unix how to do it, we tell it what we want instead.
2. Every command has a well defined interface (`stdin`, `stdout`, `errout`).
3. Commands are chainable.
4. Error handling is implicit.

## Transforming values

All these commands are acutally just transforming values from type `A` to `B`.
This all conforms to the basic unix principle:

> Do One Thing and Do It Well.
> <cite>Douglas McIlroy</cite>

This is how these functions would look in Swift:

```swift
func ls(path: String)->[String]

func grep(pattern: String)(lines: [String])->[String]

func sort(lines: [String])->[String]
```

These functions share a common pattern of transforming values of one type to
another one, so let's just call them a _transform function_ if they have this
kind of signature:

```swift
func transform<A,B>(value: A)->B
```

But of course this is only the first half of the story. Commands may fail. You
find some ideas on error handling [here]({% post_url 2015-05-22-a-swifter-way-of-handling-errors %}).
A more generic failable version of a transform would look like this:

```swift
func transform<A,B>(value: A)->Result<B>
```

This takes a value of `A`, transforms it into `B` and might fail while doing it's
work (e.g. network timeout, parsing error or just a bad mood). This is a basic
schema of functions that fits most use cases.

## The Power of Map

Imagine you're having a list of directorys and you want to list their contents.
The usual ObjC way would have been:

```objective-c
NSArray* dirs = @[@"/home", @"/root"];
NSMutableArray* paths = [NSMutableArray array];
for (NSString* dir in dirs) {
    [paths addObject:ls(dir)];
}
```

This code doesn't descibe what we want to be done but how we want it to be done.
As this kind of task is quite common (and you've written it hundreds of times)
there's something built into the standard library to help:

```swift
let dirs = ["/home", "/root"];
let paths = dirs.map(ls)
```

This is not only way shorted buy also easier to understand:

1. `paths` will have just as many elements as `dirs`
2. all elements in `paths` have the same type
3. this statement is actually about using the `ls` function instead of something
  deeply hidden within a loop.
4. the compile will figure out how to do this most effectively

But what does `map` actually do? It

- openes something that contains a value (or more of them)
- transforms the value by applying the function
- puts the value back in the box
- and skips the process if the box is empty

Does this sound more general to you than an array? `map` is defined for way more
than just array. Here's an example for `Optional<T>`:

```swift
let optionalString: String?

let optionalUppercase = optionalString.map { string in
    return string.uppercaseString
}
```

So map will open the optional, process the content and then repack the result
back into an optional. If the optional is empty, the result will be an empty
optional as well.

## Composing Functions

One great thing about functions is that you can compse them to build bigger
functions. This corresponds to the idea of object oriented programming of
composing small objects to bigger objects for more complex tasks. Composing
can look like this in bash:

```bash
ls | grep *.jpg | sort
```
and Swift:

```swift
sort(grep("*.jpg")(ls("/home")))
```

You'll notice the reverse notation of the functions in Swift (last executed,
first written). We'll do something about it in the next post.

## Extending Result to Support Map

We've already seen how versatile and useful map can be. So let's extend the
`Result<T>` type from the first post to support a map function:

```swift
enum Result<T> {
  case Success(Box<T>)
  case Error(NSError)

  func map<U>(f: T -> U) -> Result<U> {
    switch self {
    case let .Success(v): return .Success(Box(f(v.value)))
    case let .Error(error): return .Error(error)
    }
  }
}
```

This implementation transforms a `Result<T>` into a `Result<U` by using a
function. That function doesn't have to care about errors as it wont get called
if the result wasn't successfull. Now our `Result<T>` behaves just like
`Optional<T>`. One step closer to actually beeing useful.

## Taming asynchronous transforms

All transforms so far will return (after whatever timespan that might be)
with a result. But this blocks the execution of the thread. Let's
take a look at another kind of function:

```swift
func requestFromNetwork(url: NSURL, completion:(Result<NSData>->Void)){
    // do something async and call completion handler
}
requestFromNetwork(url){ result in
}
```

Although this kind of function looks pretty much different to our transform
functions it actually does the same thing: It transforms a value from one type
to another one and might fail while doing it. We're just adding some async magic
to it.

## The 2x2 Forms of a Transform

We end up with these 4 kinds of functions that transform a value to another one:

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

These functions transform a value and should only have dependencies on their
arguments. That's why we call them _pure transforms_.

## Wrapping it up (`Optional<Pun>` Intended)

We've seen the 4 types of value transformers in action and how to compose them
to a more complex function. We've seen how a function can be an argument for
another function (also called _higher order functions_) like in the map function.
Ever wondered about what functional programming is about? You've already seen
all the basics.

In the next post we'll take a closer look at chaining failable transforms together
so we finally can implement our unix example including error handling.
