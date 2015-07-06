---
tags: [swift, monads, errorhandling]
series: Functional Reactive Programming in Swift
series-index: 6
title: Wrapping View Controllers in Signals
abstract: Signals aren't just only great for handling model state - you can use them as well to plan your applications' flow.
---

# Wrapping _View Controllers_ in _Signals_

Signals aren't just only great for handling model state - you can use them as
well to plan your applications' flow.

Most people think that functional reactive programming is only useful to control
your applications' model. But it's also a pretty handy concept to layout
your applications' flow as well. You can achieve this by wrapping your view
controllers in an async transform.

For this example we'll build a transform that let's the user take a picture and
returns a `Result<UIImage>`. We'll use `UIImagePickerController` and implement it's
delegate in a class (this example uses Swift 2):

```swift
class ImageCapture: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
    private var completion: (Result<UIImage> -> Void)?
    let vc: UIViewController

    init(viewController: UIViewController) {
        vc = viewController
    }

    func takePicture(source: UIImagePickerControllerSourceType, completion: (Result<UIImage>->Void)){
        self.completion = completion
        let picker = UIImagePickerController()
        picker.delegate = self
        picker.sourceType = source
        vc.presentViewController(picker, animated: true, completion: nil)
    }

    func imagePickerControllerDidCancel(picker: UIImagePickerController) {
        let error = NSError(domain: "User did cancel", code: 401, userInfo: nil)
        completion?(Result.Error(error))
        picker.presentingViewController?.dismissViewControllerAnimated(true, completion: nil)
    }

    func imagePickerController(picker: UIImagePickerController, didFinishPickingImage image: UIImage!, editingInfo: [NSObject : AnyObject]!) {
        completion?(Result.Success(image))
        picker.presentingViewController?.dismissViewControllerAnimated(true, completion: nil)
    }
}
```

Let's explore this code step by step:

1. `ImageCapture` is an `NSObject` so it can conform to the
  `UIImagePickerControllerDelegate`. It's also a `UINavigationControllerDelegate`
  for the same reason.
2. It's inited with a `UIViewController` that's later used to present the picker.
3. `takePicture` is the actual transform. It saves the completion handler for
  later use and presents the picker.
4. If the picker is canceled by the user the completion is called with an error
  and the picker is dismissed.
5. If the user picks an image, the completion handler is called with the image object.

Now it's pretty easy to take a picture. If you have a signal that executes because
of a button, you can attach the picker like this:

```swift
let buttonSignal = Signal<Event>()
buttonSignal
.map {event in
  .Campera
}
.bind(ImageCapture(viewController: self).takePicture)
.next { image in
  // here's the picture!
}
.error { error in
  // the user is not really into taking pictures.
}
```

There we are: A selection from the user can be seen as a transform (just like
picking stuff from a list, selecting a picture, select a receiver of a message
or edit details of an object). The moment your signal is triggered, the picker
will be presented on screen and automatically dismissed once the user has
decided. Your application logic doesn't need to know about the details: It just
cares about the result of picking an image (either an image or an error).

Laying out the flow of your application in
transforms can greatly simplify view controller presentation logic because all
dependencies are explicit and there's no need to use segues (as always: Functional
programming is not a silver bullet. Segues can be extremly handy).
