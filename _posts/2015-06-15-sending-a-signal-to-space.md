---
tags: [swift, monads, errorhandling]
series: Functional Reactive Programming in Swift
series-index: 4
abstract: |
  We've already seen how to handle a chain of failable, synchronous function
  calls. Next up we explore the missing piece: Async callbacks.
---

# Sending a _Signal_ to _Space_

We've already seen [how to handle a chain of failable, synchronous function calls.]({% post_url 2015-06-09-how-to-train-your-monad %})
Next up we explore the missing piece: Async callbacks.

Our previous example had a major drawback: All chaining magic via bind only
works if the function immediately returns with a result. If it takes some time
and returns later via a callback block `Result<T>` is not a big help.

## Signals as async results
A signal is nothing more than a `Result<T>` that will return the value at a later
point in time and that can update it's value later on. Just see it an object
oriented bag of callback methods.

```swift
class Signal<T> {

  private var value: Result<T>?
  private var callbacks: [Result<T> -> Void] = []

  public func subscribe(f: Result<T> -> Void) {
    if let value = value {
      f(value)
    }
    callbacks.append(f)
  }

  public func update(value: Result<T>) {
    self.value = value
    self.callbacks.map{$0(value)}
  }
}
```
`Signal<T>` is a reference type (in contrast to `Result<T>` which was a value
type) because we will need a reference to it so we can update the value later.
Via `subscribe` you can add a new function that receives a result as an argument
(this is actually just the usual callback blocks we're all used to). Every object
that has a reference to the signal can update it at any given time (let's talk
about multithreading in the next blog post). Also the signal keeps it's last
value (either successfull or unsuccessfull) so each subscriber get's the value
immediately on subscription.

## Transforming signals
Let's implement the methods we already know from `Result<T>` on signal to handle
[transform functions]({% post_url 2015-05-24-transforming-the-world-into-a-better-place %}):

```swift
func map<U>(f: T -> U) -> Signal<U> {
  let signal = Signal<U>()
  subscribe { result in
    signal.update(result.map(f))
  }
  return signal
}
```
Mapping a signal returns a new signal that contains the transformed value. If
the current signal changes the mapped signal will be notified and changed as well.
This also introduces the memory management for signals: Each signal keeps a
reference to the child signals that depend on it via the subscribe block. If you
let go of a signal all transformed signals will dealloc as well (unless you keep
a reference to a transformed signal explicily).

Next up are failable transforms:

```swift
func bind<U>(f: T -> Result<U>) -> Signal<U> {
  let signal = Signal<U>()
  subscribe { result in
    signal.update(result.bind(f))
  }
  return signal
}

func bind<U>(f: (T, (Result<U>->Void))->Void) -> Signal<U> {
  let signal = Signal<U>()
  subscribe { value in
    value.bind(f)(signal.update)
  }
  return signal
}
```

The first one is for synchronous transforms (you may remember it from the Result
implementation). The second one finally explains why we needed the async bind
method for results. Chaining signals is now pretty easy: You can bind an async
function that may fail to a signal and immediately get a new signal back. You
then subscribe to that new signal and get notified once the operation is complete.

## Ensuring success
Sometimes you want to transform a signal even if it fails (we'll see a usefull
application in the next post about threading). For that we'll define the `ensure`
function that takes a result and returns a result (it's event allowed for it to
turn an error into a success which is pretty handy for retry logic):

```swift
public func ensure<U>(f: (Result<T>, (Result<U>->Void))->Void) -> Signal<U> {
  let signal = Signal<U>()
  subscribe { value in
    f(value) { signal.update($0) }
  }
  return signal
}
```
This is the most straight forward implementation so far (create a new signal and
notify it as soon as content is available).

Believe it or not: We're finally done. You can find the full source code as a
framework on Github called [Interstellar](https://github.com/JensRavens/Interstellar).
You can also easily add it to your project via
[Carthage](https://github.com/Carthage/Carthage), just add

```bash
github "jensravens/interstellar" "master"
```
to your Cartfile.

## Why are signals so usefull?
Signals actually replace a whole lot of communication patterns in an easy, object
oriented way (who said FP and OOP can't live together?). Signals can replace KVO,
target/action, delegates, multi-delegates and notifications in a safe way (no need
to unsubscribe!).

Let's see a UIKit based example:

```swift
var TextSignalHandle: UInt8 = 0
extension UISearchBar: UISearchBarDelegate {
  public var textSignal: Signal<String> {
    let signal: Signal<String>
    if let handle = objc_getAssociatedObject(self, &TextSignalHandle) as? Signal<String> {
      signal = handle
    } else {
      signal = Signal()
      delegate = self
      objc_setAssociatedObject(self, &TextSignalHandle, signal, objc_AssociationPolicy(OBJC_ASSOCIATION_RETAIN_NONATOMIC))
    }
    return signal
  }

  public func searchBar(searchBar: UISearchBar, textDidChange searchText: String) {
    textSignal.update(.Success(Box(self.text)))
  }
}
```
This extends `UISearchBar` to have an additional `textSignal` property in addition
to `text`. It uses the objc runtime to associate the signal to the object (so we can
fake a property in an extension). Every time the search bar updates it's text all
subscribers to the signal will be called.

Now it's pretty easy to execute a function every time the user types on the
keyboard (`next` is a wrapper around subscribe that only executes if the result is
a `.Success`):

```swift
let searchBar = UISearchBar()
searchBar.textSignal.next { text in
  println("User has typed \(text)")
}
```
`text` is already of type `String` and can be used in the block right away. Be
carefull if you're using self within the block as this can cause a memory leak
(just make sure to mark it as weak before).

## This is not all or nothing.
You don't have to decide for the whole project if you want to do a signal based
architecture. Just drop in a signal in some places that make sense to you and see
how easy it gets over time. Signals are a really great way to cut your way
through dependency jungle (as the subscriber doesn't have to know about the
observable and can get rid of it at any time).

## Where to go from here
Interstellar is a very simple and lightweight implementation of Functional
Reactive Programming. It's open source on GitHub (please help to write
documentation and tests!).

If you need more there are bigger projects out there like
[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) and
[RxSwift](https://github.com/kzaher/RxSwift). You should also take a look at
[The introduction to Reactive Programming you've been missing](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754).

In the next post I'll will talk about threading in the context of signals and
how to make the main/background dance ridiculously simple. After that we're
ready to implement an example application using Interstellar.
