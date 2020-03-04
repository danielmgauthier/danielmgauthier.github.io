---
layout: post
title: "Mastering view controller transitions, part 1: Make them reusable"
twitter_subtitle: "Some thoughts on structuring your custom transition code 🏗"
twitter_image: assets/img/indie-4.png
excerpt_separator: <!--more-->
---

Last week, I wrote a primer on how custom view controller transitions actually work. If you missed it and are relatively new to this stuff, or want a quick refresher, you might want to check it out [here](https://danielgauthier.me/2020/02/19/indie-4.html).

This week, I’m doing a set of 3 short articles that explore some more advanced, concrete ideas around custom transitions. This was originally intended to be one relatively straightforward article, but apparently I’ve got a lot to say about this stuff (or maybe I’m bad at editing 🤔), and I decided that putting out a monolithic 8000 word piece probably wasn’t the most palatable choice. So, each article will revolve around one goal I had in mind as I built some nice custom transitions into my own app, and will detail the solutions I came up with in trying to achieve that goal.

<!--more-->

To get us all on the same page, let’s take a look at a couple of these transitions in action. You’ll see two different view controllers presented: the “Add game” view controller, and the “Tags” view controller. Both are presented using the same animation and presentation controllers, but they use different interaction controllers on dismissal — “Tags” can be quickly and easily swiped away, while “Add game” is a more deliberate interaction that requires the user to drag a certain distance before releasing, and that includes some decoration views as part of the interaction.

<p class="screenshot">
    <video controls muted>
      <source src="/assets/img/game-demo.mov" type="video/mp4">
    Your browser does not support the video tag.
    </video>
</p>

There are some cool things going on behind the scenes here that all add up to what I think is a pretty nifty user experience. Let’s talk about goal number one!

## Goal 1: I want a dead-simple way to use my custom transitions
This first goal actually isn’t directly relevant to the end-user experience, but is VERY relevant to me, the developer, who doesn’t want a total mess on his hands as he starts using his custom transitions in different contexts throughout the app. And, let’s be real: all technical debt seeps into the user experience eventually! So, let’s figure out how to make sure our custom transitions are easy to use and reuse. 

As I mentioned [last week](https://danielgauthier.me/2020/02/19/indie-4.html), the “transitioning delegate” responsibility often falls on the presenting view controller by default. In the most naïve case, this means that every time you want to present something using a custom transition, you need to implement a bunch of delegate methods in your presenting VC, giving it responsibilities it probably shouldn’t really have. Furthermore, if you want to support an interactive dismissal gesture, you might end up implementing a gesture handler inside your presented view controller — its view _is_ where the gesture is happening, after all — and then figure out some way to hook up your gesture handling code to the interaction controller that your presenting view controller owns.

There may be some hypothetical cases where this approach is passable — namely, if you know you’re building a very specific custom transition that’s only ever going to be used in one place. In my case, however, I knew I wanted to be able to present any view controller, from anywhere, using my custom transitions. In other words, I didn’t want to have to keep reimplementing gesture handlers and `UIViewControllerTransitioningDelegate` methods every time I wanted to use my transitions. 

My solution starts with the idea that we should have a dedicated class to serve as the transitioning delegate for our custom modal presentations — we’ll call it `ModalTransitionManager`. As the transitioning delegate, it’s simply responsible for vending presentation, animation and interaction controllers when asked. It might look something like this:

```swift
class ModalTransitionManager: NSObject {

    private var interactionController: InteractionControlling?

    init(interactionController: InteractionControlling?) {
        self.interactionController = interactionController
    }
}

extension ModalTransitionManager: UIViewControllerTransitioningDelegate {

    func presentationController(forPresented presented: UIViewController, presenting: UIViewController?, source: UIViewController) -> UIPresentationController? {
        return ModalPresentationController(presentedViewController: presented, presenting: presenting)
    }

    func animationController(forPresented presented: UIViewController, presenting: UIViewController, source: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        return ModalTransitionAnimator(presenting: true)
    }

    func animationController(forDismissed dismissed: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        return ModalTransitionAnimator(presenting: false)
    }

    func interactionControllerForDismissal(using animator: UIViewControllerAnimatedTransitioning) -> UIViewControllerInteractiveTransitioning? {
        guard let interactionController = interactionController, interactionController.interactionInProgress else {
            return nil
        }
        return interactionController
    }
}
```

The only tricky thing to notice here is the way the interaction controller is handled. Since UIKit uses the return value of `interactionControllerForDismissal(using:)` as an indication of whether or not the dismissal should be performed interactively, our modal transition manager can’t simply create and return a new interaction controller here like it does with the animation and presentation controllers; if it did, UIKit would always assume a dismissal was happening interactively, and wait for interactive updates, even if the user tapped a button to dismiss the view non-interactively. This wouldn’t be an issue if the **only** way to dismiss our presented view controller was via an interaction, but in our case (and probably in most cases), it’s important that our users can dismiss with a gesture _or_ with the tap of a button.

So, our `ModalTransitionManager` keeps a reference to an optional interaction controller, and the interaction controller itself knows if a dismissal has been triggered by a gesture via its `interactionInProgress` property (more on that in a sec). In this way, our `ModalTransitionManager` has the information it needs to either return an interaction controller, telling UIKit “yep, let’s do an interactive dismissal,” or return nil to say “no, just animate this dismissal non-interactively.”

Of course, our `ModalTransitionManager` needs to be owned by someone from the beginning of presentation until the end of dismissal so that it can communicate with UIKit as needed. To make sure of this, let’s write a `CustomPresentable` protocol that every custom-presented view controller will conform to, and that ensures that the presented view controller keeps a strong reference to the `ModalTransitionManager`:

```swift
protocol CustomPresentable: AnyObject {
    var transitionManager: UIViewControllerTransitioningDelegate? { get set }
    // ...
} 
```

With this in place, we’ve taken the transitioning delegate responsibility out of our view controllers, and moved it into a separate object that the presented view controller keeps a reference to, but otherwise doesn’t care about. 

The second thing to address is implementing a gesture handler on our presented view that can control the dismissal interaction. Instead of doing it inside the presented view controller, we can move this responsibility into the interaction controller itself. First, we create a simple protocol that inherits from `UIViewControllerInteractiveTransitioning`:

```swift
protocol InteractionControlling: UIViewControllerInteractiveTransitioning {
    var interactionInProgress: Bool { get }
}
```

This enforces that any interaction controller we build must be able to report when an interaction has started; we used this property in our `ModalTransitionManager` above.

Now, we can build an interaction controller that’s responsible for handling gestures, and therefore knows when to set `interactionInProgress` to `true`:

```swift
class StandardInteractionController: NSObject, InteractionControlling {
    var interactionInProgress = false
    private weak var viewController: (UIViewController & CustomPresentable)!

    init(viewController: UIViewController & CustomPresentable) {
        self.viewController = viewController
        super.init()
        prepareGestureRecognizer(in: viewController.view)
    }

    private func prepareGestureRecognizer(in view: UIView) {
        let gesture = UIPanGestureRecognizer(target: self, action: #selector(handleGesture(_:)))
        view.addGestureRecognizer(gesture)
    }

    @objc func handleGesture(_ gestureRecognizer: UIPanGestureRecognizer) {
        guard let superview = gestureRecognizer.view?.superview else { return }
        let translation = gestureRecognizer.translation(in: superview).y
        let velocity = gestureRecognizer.velocity(in: superview).y

        switch gestureRecognizer.state {
        case .began: gestureBegan()
        case .changed: gestureChanged(translation: translation + interruptedTranslation, velocity: velocity)
        case .cancelled: gestureCancelled(translation: translation + interruptedTranslation, velocity: velocity)
        case .ended: gestureEnded(translation: translation + interruptedTranslation, velocity: velocity)
        default: break
        }
    }
    
    private func gestureBegan() {
        if !interactionInProgress {
            interactionInProgress = true
            viewController.dismiss()
        }
    }
    // ...
}
```

Of course, there’s a lot more than that going on in the full implementation, but this gives you the idea: our interaction controller is initialized with the presented view controller, and is responsible for installing and responding to the gesture recognizer, allowing us to set `interactionInProgress` to `true` before having UIKit kick off the dismissal, and making it trivial to get subsequent gesture updates through our interaction controller to UIKit. 

Finally, this is all tied together with a simple UIViewController extension:

```swift
enum InteractiveDismissalType {
    case none
    case standard
    case input
}

extension UIViewController {
    func present(_ viewController: (UIViewController & CustomPresentable), 
                 interactiveDismissalType: InteractiveDismissalType, 
                 completion: (() -> Void)? = nil) {

        let interactionController: InteractionControlling?
        switch interactiveDismissalType {
        case .none:
            interactionController = nil
        case .standard:
            interactionController = StandardInteractionController(viewController: viewController)
        case .input:
            interactionController = InputInteractionController(viewController: viewController)
        }

        let transitionManager = ModalTransitionManager(interactionController: interactionController)
        viewController.transitionManager = transitionManager
        viewController.transitioningDelegate = transitionManager
        viewController.modalPresentationStyle = .custom
        present(viewController, animated: true, completion: completion)
    }
}
```

Here’s where we actually kick off the custom presentation. The caller can specify the type of interactive dismissal, which dictates which interaction controller is created. The `ModalTransitionManager` is created and assigned to the view controller we want to present — we need to hold onto it strongly in the `transitionManager` property, because `transitioningDelegate` is a weak reference — and then we present the view controller. 

What does this all mean? It means that we can now present any view controller, from anywhere, and get all our desired animation and interaction behaviour, by simply making sure our presented view controller conforms to CustomPresentable:

```swift
final class CoolViewController: UIViewController, CustomPresentable {
    var transitionManager: UIViewControllerTransitioningDelegate?
    // ...
}
```

…and then calling our presentation method:

```swift
let coolViewController = CoolViewController()
present(coolViewController, interactiveDismissalType: .standard)
```

...and everything else works like magic. That's a whole lot of complexity we no longer have to think about. Pretty cool 😎

**For a working example of the techniques described here and in the other two articles in this series on custom view controller transitions, check out [this repo on GitHub](https://github.com/danielmgauthier/ViewControllerTransitionExample).**

<br/>

---

<br/>

Thanks for reading! This is a relatively quick look at a pretty tricky topic, so if you need some help or have any follow-up questions, Let me know! Find me on Twitter [here](https://twitter.com/danielmgauthier) 🐦
