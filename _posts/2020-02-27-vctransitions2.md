---
layout: post
title: "Mastering view controller transitions, part 2: Make them feel natural"
twitter_subtitle: "It's the right thing to do."
twitter_image: assets/img/indie-4.png
preview: Making interactions feel "natural" is a bit of a subjective thing, but for me, there were a few details I knew I wanted to get right when it came to dismissing my view controllers interactively.
redirect_from:
    - /2020/02/27/indie5-2.html 
---

This week, I’m doing a set of 3 short articles that explore some more advanced, concrete ideas around custom transitions. I give some additional context in the first one, so you may want to take a gander if you haven’t already:

**[Mastering view controller transitions, part 1: make them reusable](https://danielgauthier.me/2020/02/24/vctransitions1.html)**

As a quick refresher, here’s an example of the transitions we introduced in the first article, and that all this writing is based on:

<p class="screenshot">
    <video controls muted>
      <source src="/assets/img/game-demo.mov" type="video/mp4">
    Your browser does not support the video tag.
    </video>
</p>

Okay, you’re up to speed! Let’s dive into the second goal.

## Goal 2: I want my interactions to feel natural

Making interactions feel "natural" is a bit of a subjective thing, but for me, there were a few details I knew I wanted to get right when it came to dismissing my view controllers interactively. They're all fairly obvious best practices when it comes to animating and interacting with views, but some of them can be a bit tricky to get right in the context of view controller transitions. 

Keep in mind that an interactive transition can look like anything, really, and these tips aren't universally applicable to any and all interactive transitions; rather, for the sake of example, these are focused primarily on "swipe down to dismiss" interactions, and the code samples are derived from my implementation of the "Tags" view being dismissed in the video above. My hope is for these to get you thinking about how you might add similar polish to your own transitions, even if they operate slightly differently than mine.

I also want to call attention to the fact that, while it might seem like a simple `UIPercentDrivenInteractiveTransition` could get us most of the way towards implementing my "Tags" view's interactive dismissal, I ended up building my interactive transitions using my own implementations of `UIViewControllerInteractiveTransition`, rather than using or subclassing `UIPercentDrivenInteractiveTransition`. I had a whole thing written about all the issues I ran into, and why I made this decision, but it got a little bit out of hand and this post is already long enough as it is. So, although most of what's in this post would apply in either case, just keep in mind that the code examples are based on a custom interaction controller. Maybe I'll post about the trials and tribulations of adopting `UIPercentDrivenInteractiveTransition` one day; for now, let's start looking at a few tips you might be able to use to make your interactive transitions feel great.


### Use spring animations

When you’re animating the movement of views, don’t do this:

<p class="screenshot">
    <video controls muted>
      <source src="/assets/img/indie-5-2-eg1.mov" type="video/mp4">
    Your browser does not support the video tag.
    </video>
</p>

Ew, yuck.
{:.caption}

Instead, use spring animations. They make your animated views feel more like physical, real-world objects than they would with linear or easing curves, and are standard across iOS. Generally speaking, you can implement them with something like this:

```swift
let animator = UIViewPropertyAnimator(duration: 0.5, dampingRatio: 1.0) {
  // animate your thing
}
```

Or, if you’re old-fashioned, like this: 

```swift
UIView.animate(withDuration: 0.5, delay: 0.0, usingSpringWithDamping: 1.0, initialSpringVelocity: 0.0, options: .allowUserInteraction, animations: {
  // animate your thing
}, completion: nil)
```

Next!

### Take gesture velocity into account

For our views to feel like physical, real-world objects, we want them to have some momentum when we fling them around. To do this, we can take advantage of an oft-misunderstood parameter of spring animations: `initialVelocity`. This parameter is exactly what's going to allow us to carry the velocity of our gesture into the animation that immediately follows, so that we don't end up with something like this:

<p class="screenshot">
    <video controls muted>
      <source src="/assets/img/indie-5-2-eg2.mov" type="video/mp4">
    Your browser does not support the video tag.
    </video>
</p>

Nope, don't like this.
{:.caption}

Unfortunately, there’s a reason this parameter is a bit misunderstood: it’s not quite as simple as grabbing the velocity from our gesture recognizer and plugging it in. While the gesture recognizer gives its x and y velocity in points/second, `initialVelocity` expects those values to be normalized to the total distance that the animation covers. So, when your gesture ends, and you’ve decided based on that gesture that the transition should be completed, here’s a condensed version of how you might set up your animation:

```swift
@objc func handleGesture(_ gestureRecognizer: UIPanGestureRecognizer) {
  guard let superview = gestureRecognizer.view?.superview else { return }
  let translation = gestureRecognizer.translation(in: superview).y
  let velocity = gestureRecognizer.velocity(in: superview).y
  // ...
  if gestureRecognizer.state == .ended {
    gestureEnded(translation: translation + interruptedTranslation, velocity: velocity)
  }
}

private func gestureEnded(translation: CGFloat, velocity: CGFloat) {
  finish(initialSpringVelocity: springVelocity(distanceToTravel: interactionDistance - translation, gestureVelocity: velocity))
}

private func springVelocity(distanceToTravel: CGFloat, gestureVelocity: CGFloat) -> CGFloat {
  distanceToTravel == 0 ? 0 : gestureVelocity / distanceToTravel
}
```

Not so hard!

### Be smart about deciding whether to complete or cancel a transition

This one is somewhat related to the previous point: much like we should take velocity into account when performing our post-gesture animation, we should also take it into account when deciding which direction to animate in the first place. When the pan gesture is released, it might be tempting to simply evaluate whether the drag extended across a certain threshold: for example, if the gesture has travelled at least half the total dismissal distance, complete the dismissal, and otherwise, cancel it and animate back to the starting position. Unfortunately, this doesn’t end up feeling very natural:

<p class="screenshot">
    <video controls muted>
      <source src="/assets/img/indie-5-2-eg3.mov" type="video/mp4">
    Your browser does not support the video tag.
    </video>
</p>

No, this certainly won't do.
{:.caption}

If the user flings your view down with some speed, they expect it to continue moving in that direction. So, you’ll want to add a bit of logic that looks something like this:

```swift
private func gestureEnded(translation: CGFloat, velocity: CGFloat) {
  if velocity > 300 || (translation > interactionDistance / 2.0 && velocity > -300) {
    finish(initialSpringVelocity: springVelocity(distanceToTravel: interactionDistance - translation, gestureVelocity: velocity))
  } else {
    cancel(initialSpringVelocity: springVelocity(distanceToTravel: -translation, gestureVelocity: velocity))
  }
}
```

This basically says that if there’s significant velocity in the dismissal direction, dismiss the view controller no matter what. Otherwise, only dismiss if the gesture has moved at least half the total distance, and there isn’t significant velocity in the opposite direction. I’ll note that 300 is a bit of a magic number here that you might want to tweak, but you definitely don’t want to just check whether the velocity is positive or negative; if the user stops their movement and lets go, it’s anyone’s guess whether the gesture’s final velocity will be slightly positive or slightly negative.

### Make sure your interaction is always enabled

Nothing breaks the illusion of dragging physical objects around on a screen quite like suddenly and unexpectedly being unable to interact with an object you were just dragging around a second ago. Here’s what we don’t want:

<p class="screenshot">
    <video controls muted>
      <source src="/assets/img/indie-5-2-eg4.mov" type="video/mp4">
    Your browser does not support the video tag.
    </video>
</p>

Woof.
{:.caption}

Sure, it might seem like a bit of an edge case, and maybe costing your users a fraction of a second doesn’t seem like the end of the world, but this kind of behaviour can cause tiny moments of frustration and make it feel like there’s a bit of an unpredictable barrier between your users and the objects they’re trying to manipulate on screen. This was a bit tough to get right in my case, and I’m not sure this is the most elegant solution, but my basic approach is this: 

* Keep track of whether an interaction is in progress with an `interactionInProgress` flag. Set this flag to `true` as soon as a gesture begins, and only set it back to `false` when the cancellation animation finishes completely, i.e. **the dragged view is completely at rest**.
* Only ever call `viewController.dismiss()` to begin the transition when the flag is flipped from `false` to `true`. End the transition only when the flag is flipped from `true` to `false` (by calling `transitionContext.cancelInteractiveTransition()` and `transitionContext.completeTransition(false)`).

This set-up effectively keeps your interaction “in transition” for as long as the user is playing with the dragged view — if the user lets go, but then grabs the view again as it's animating back to its starting position, the interaction remains in transition throughout. This gives us the freedom to maintain the dragged view’s interactivity without having to worry about reporting the start and cancellation of the transition to UIKit over and over again (which can cause weird behaviour and crashes). How exactly you implement this pattern depends a lot on how your interaction controller is structured, so I’ll leave the details up to you!

### Deal with scroll views

If you establish a swipe-down-to-dismiss pattern in your app, you want to make sure you don’t end up with cases where the user expects this pattern to hold, but it doesn’t. A common stumbling block arises when your presented view — the one you want to dismiss with a swipe — is entirely or largely made up of a vertical scroll view. (Remember, table views and collection views are scroll views too.) If the user swipes down on the scroll view, the scroll view consumes the gesture, effectively preventing the user from swiping to dismiss.

<p class="screenshot">
    <video controls muted>
      <source src="/assets/img/indie-5-2-eg5.mov" type="video/mp4">
    Your browser does not support the video tag.
    </video>
</p>

This app is bad!
{:.caption}

Of course, being able to scroll is kind of important too. So how do we reconcile these two conflicting gestures in a way that feels natural to the user? I think a nice solution is to simply decide which gesture handler to trigger based on whether or not the scroll view is scrolled to the top. In other words: if the scroll view’s vertical content offset is 0 or less, the swipe should be handled by the dismissal gesture; otherwise, it should be handled by the scroll view.

If you read [my last post](https://danielgauthier.me/2020/02/24/indie5-1.html), you’ll know that a big priority of mine in building these transitions was to make sure it was easy to use them to present any view controller, from anywhere. So, from our interaction controller, how can we manipulate the gestures of a scroll view that may or may not exist inside a view controller that could be of any type?

**Protocols!**

In that last post, in my quest for reusability, I introduced a lightweight protocol that any view controller would have to conform to if it wanted to be presented in a custom way. Let’s add to this protocol:

```swift
protocol CustomPresentable: AnyObject {
  var transitionManager: UIViewControllerTransitioningDelegate? { get set }
  var dismissalHandlingScrollView: UIScrollView? { get }
}

extension CustomPresentable {
  var dismissalHandlingScrollView: UIScrollView? { nil }
}
```

We’ve added a read-only property, `dismissalHandlingScrollView`, and we’ve also used a protocol extension to give it a default nil value. This means that view controllers that conform to this protocol don’t _have_ to specify a `dismissalHandlingScrollView` if they don’t have one, but those that do can specify one as follows:

```swift
class FilterSelectionViewController: UIViewController, CustomPresentable {
  var transitionManager: UIViewControllerTransitioningDelegate?
  var dismissalHandlingScrollView: UIScrollView? { self.tableView }
  // ...
}
```

Note that default protocol values have their drawbacks — for example, it means that when you’re conforming to `CustomPresentable`, it’s not immediately obvious that `dismissalHandlingScrollView` is a property you can or should implement — but I like that it minimizes the overhead of creating new custom presentable view controllers. 

Since our interaction controller from last post was already initialized with a view controller that conforms to `CustomPresentable`, we now have access to any scroll view that might interfere with our dismissal gesture, and can handle it appropriately:

```swift
class StandardInteractionController: NSObject, InteractionControlling {
  private weak var viewController: (UIViewController & CustomPresentable)!

  init(viewController: UIViewController & CustomPresentable) {
    self.viewController = viewController
    super.init()
    prepareGestureRecognizer(in: viewController.view)
    if let scrollView = viewController.dismissalHandlingScrollView {
      resolveScrollViewGestures(scrollView)
    }
  }

  private func prepareGestureRecognizer(in view: UIView) {
    let dismissalGestureRecognizer = UIPanGestureRecognizer(target: self, action: #selector(handleGesture(_:)))
    view.addGestureRecognizer(dismissalGestureRecognizer)
  }

  private func resolveScrollViewGestures(_ scrollView: UIScrollView) {
    let dismissalGestureRecognizer = UIPanGestureRecognizer(target: self, action: #selector(handleGesture(_:)))
    dismissalGestureRecognizer.delegate = self

    scrollView.addGestureRecognizer(dismissalGestureRecognizer)
    scrollView.panGestureRecognizer.require(toFail: dismissalGestureRecognizer)
  }

  // ...
}

extension StandardInteractionController: UIGestureRecognizerDelegate {
  func gestureRecognizerShouldBegin(_ gestureRecognizer: UIGestureRecognizer) -> Bool {
    if let scrollView = viewController.dismissalHandlingScrollView {
      return scrollView.contentOffset.y <= 0
    }
    return true
  }
}
```

As you can see, resolving the scroll and dismissal gestures in our interaction controller is pretty simple. On initialization, we check to see if the view controller being presented has a dismissal handling scroll view. If so, we add a dismissal gesture recognizer to the scroll view, and specify that the scroll view’s `panGestureRecognizer` — the one that actually handles scrolling — should only be recognized if the dismissal gesture recognizer fails first. Finally, via conformance to `UIGestureRecognizerDelegate`, we set that dismissal gesture recognizer to fail if the scroll view isn't currently scrolled to the top. 

That's all for now, folks. Using a few simple techniques, we now have an interactive transition that is not only highly reusable, but also feels pretty great to interact with. Hopefully a few of these come in handy the next time you’re duelling with interactive transitions! 🤺

**For a working example of the techniques described here and in the other two articles in this series on custom view controller transitions, check out [this repo on GitHub](https://github.com/danielmgauthier/ViewControllerTransitionExample).**

<br/>

---

<br/>

Thanks so much for reading my weird little blog. This is a relatively quick look at a pretty tricky topic, so if you need some help or have any follow-up questions, Let me know! Find me on Twitter [here](https://twitter.com/danielmgauthier) 🦆
