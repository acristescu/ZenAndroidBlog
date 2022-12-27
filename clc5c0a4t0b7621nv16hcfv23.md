# Building complex Android animations with RxJava

![Building complex Android animations with RxJava](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091324545/7e5F-G97b.png align="left")

In this article I'm going to discuss a little technique I am using to choreograph complex animations on Android by leveraging the power of RxJava. I'm going to show how it can result in clean and simple code that communicates intent and hides the boilerplate in reusable Kotlin extension methods (or JAVA utility classes). As an example we're going to be building a two-step Floating Action Button menu with slide in and fade in animations (see the GIF below). The full code for this example is available in this [repo](https://github.com/acristescu/RxAnimationExample).

## What we are trying to achieve

For the purposes of this tutorial we're going to be building the next animation:

![Building complex Android animations with RxJava](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091326308/FmYHn68WC.gif align="left")

Let's slow that down so we can see better what's going on.

![Building complex Android animations with RxJava](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091328889/2x6B6LWN4.gif align="left")

Basically, what we have here is:

* Two sets of 3 mini FABs (Floating Action Button), each accompanied by an explanation label.
    
* Another big FAB is always visible and can be used to hide or show the first set of FABs.
    
* The second set of FABs shows up when the user selects something from the first set of FABs.
    
* Each of the mini FABs has an enter animation that slides it and grows it at the same time, and the accompanying label fades in as soon as the mini FAB has reached the final position.
    
* The big FAB also rotates for the duration of the entire animation.
    

This is a real world example that I pulled out of my OnlineGo application and it's the flow that allows the user to select the board size and clock speed for new games.

## The classic way: ViewPropertyAnimator

The simplest way of doing basic view animations at the time of this writing would be to use the `ViewPropertyAnimator` framework. It offers an easy and concise way of doing animations *on a single view*. For example, to fade out a view, all you have to do is

```java
view.animate().alpha(0)
```

That's it! If you want to shrink it at the same time, you would add the `scaleX` and `scaleY` calls:

```java
view.animate()
    .alpha(0)
    .scaleX(0.5f)
    .scaleY(0.5f)
```

This works very nicely, the code is concise and conveys intent very clearly and the community understands the technique quite easily.

How about creating an animation that runs as soon as the first one finishes? For this the framework offers the `withEndAction` call. The code to make two views fade out *one after another* would look like this:

```kotlin
view.animate()
    .alpha(0)
    .scaleX(0.5f)
    .scaleY(0.5f)
    .withEndAction {
        secondView.animate()
            .alpha(0)
            .scaleX(0.5f)
            .scaleY(0.5f)
    }
```

Already you can see the code is starting to lose a lot of the elegance, even if Kotlin Lambda Expression helps a bit here and we've only choreographed two views. The example animation we set ourselves as a target has 13 different moving views and writing code like this will quickly result in incredibly nested `withEndAction` lambda expressions that will be hard to understand and maintain.

Not to mention that subsequent calls to `withEndAction` overwrite themselves instead of adding (i.e. the `ViewPropertyAnimator` only has one listener, not a list of them) hence you can inadvertently cancel some behaviour if you're not careful.

So basically the problem as outlined above is that we have the means to animate one view, but the code to choreograph multiple animations between themselves is the tricky part.

## Enter RxJava

Animations themselves are asynchronous calls. You start an animation, it does its thing for a few milliseconds and later on it completes. You as a programmer are interested when that animation finishes to perhaps start another one or progress the state of the program.

As it turns out there is a RxJava 2 object that is perfect to emulate such a behaviour: `Completable`.

> RxJava 2 introduced `Completable`, but in RxJava 1 you could simply use an Observable that does not emit anything to achieve the same results.

The basic gist of it is that you can wrap your animation inside a `Completable` and use the `withEndAction` to call `onComplete`. An example is worth 1000 words:

```kotlin
val animation = Completable.create {
        view.animate()
            .alpha(0f)
            .withEndAction(it::onComplete)
}
animation.subscribe()
```

As you can see, when the Completable is subscribed to, we're executing the same code as above `view.animate().alpha(0)`, however we've used the `withEndAction` to signal completion. The advantage of this is that we can chain together animation (and other RxJava operations such as network calls!) with the powerful operators that RxJava uses. For example, if you want to run two animations one after the other you would use `andThen` and to run them simultaneously you can use `Completable.mergeArray`.

For example, to fade one view after another you can write:

```kotlin
val animation = Completable.create {
        view.animate()
            .alpha(0f)
            .withEndAction(it::onComplete)
}
val animation1 = Completable.create {
        view1.animate()
            .alpha(0f)
            .withEndAction(it::onComplete)
}
animation.andThen(animation1).subscribe()
```

Fading both of them at the same time can be achieved by:

```kotlin
val animation = Completable.create {
        view.animate()
            .alpha(0f)
            .withEndAction(it::onComplete)
}
val animation1 = Completable.create {
        view1.animate()
            .alpha(0f)
            .withEndAction(it::onComplete)
}
Completable.mergeArray(animation, animation1).subscribe()
```

How about animating two views, waiting for them to finish, then animating two others?

```kotlin
val animation = ...
val animation1 = ...
Completable.mergeArray(animation, animation1)
       .andThen { Completable.mergeArray(animation2, animation3) }
       .subscribe()
```

As you can see, as long as you are familiar with RxJava syntax, you can choreograph incredibly complicated animation sequences and return another animation (as a Completable) that can be executed (by subscribing to it) or used as building blocks in and even more complex animations. Just don't forget to subscribe to it at the end or it will never start :)

Arguably, the first part of the code (in which we're wrapping the basic animations into Completables) is not very nice, but it's easy to see that it's repetitive and that it can be moved in a Kotlin extension function or Java static utility method. Which brings us to the next section

## Bonus: using Kotlin extension functions for animations

The nicest way I've found for hiding away the boilerplate is to do create functions like this:

```kotlin
fun View.fadeOut(duration: Long = 30): Completable {
    return Completable.create {
        animate().setDuration(duration)
                .alpha(0f)
                .withEndAction {
                    visibility = View.GONE
                    it.onComplete()
                }
    }
}
```

These utility methods will then be usable by doing something like:

```kotlin
myView.fadeOut().subscribe()
```

They are also reusable between projects and you could conceivably even make a small library out of them. For now, you can see a collection of such functions in [ViewExtensions.kt](https://github.com/acristescu/RxAnimationExample/blob/master/app/src/main/java/io/zenandroid/rxanimationexample/extensions/ViewExtensions.kt):

```kotlin
fun View.slideIn(offset: Float): Completable {
    return Completable.create {
        visibility = View.VISIBLE
        alpha = 0f
        scaleX = 0f
        scaleY = 0f
        translationY = offset
        animate().alpha(1f)
                .translationY(0f)
                .scaleX(1f)
                .scaleY(1f)
                .setDuration(200)
                .setInterpolator(OvershootInterpolator())
                .withEndAction(it::onComplete)
    }
}

fun View.slideOut(offset: Float): Completable {
    return Completable.create {
        animate().alpha(0f)
                .scaleX(0f)
                .scaleY(0f)
                .translationY(offset)
                .setDuration(200)
                .withEndAction {
                    visibility = View.GONE
                    it.onComplete()
                }
    }
}

fun View.fadeOut(duration: Long = 30): Completable {
    return Completable.create {
        animate().setDuration(duration)
                .alpha(0f)
                .withEndAction {
                    visibility = View.GONE
                    it.onComplete()
                }
    }
}

fun View.fadeIn(): Completable {
    return Completable.create {
        visibility = View.VISIBLE
        alpha = 0f
        animate().alpha(1f)
                .setDuration(200)
                .withEndAction(it::onComplete)
    }
}

fun View.rotate(degree: Float): Completable {
    return Completable.create {
        animate().rotation(degree)
                .setDuration(200)
                .withEndAction(it::onComplete)
    }
}
```

With these in place your application code looks like this:

```kotlin
longFab.slideIn(fabMiniSize).andThen(longLabel.fadeIn())
```

## Putting it all together

Let's now come back to the real-world example we've set ourselves as a target at the beginning of this article. Armed with the View extension functions described above let's try to piece together all the animations described at the beginning.

The two sets of FABs are similar, let's call them `Speed Menu` and `Size Menu` respectively. First, let's look at the code for showing one of these menus:

```kotlin
    private fun showSpeedMenu() = Completable.mergeArray(
            fab.rotate(45f),
            longFab.slideIn(fabMiniSize).andThen(longLabel.fadeIn()),
            normalFab.slideIn(2 * fabMiniSize).andThen(normalLabel.fadeIn()),
            blitzFab.slideIn(3 * fabMiniSize).andThen(blitzLabel.fadeIn())
    )
```

As we can see, we're defining the `show speed` animation as running 4 animations at the same time and waiting for all to finish:

1. rotating the large FAB 45 degrees
    
2. sliding in the first mini FAB and then fading in the fist label
    
3. sliding in the second mini FAB and then fading in the second label
    
4. sliding in the third mini FAB and then fading in the third label
    

Showing the menu with all the 9 moving parts can then be achieved by

```kotlin
showSpeedMenu().subscribe()
```

Hiding a menu is done in a similar fashion:

```kotlin
    private fun hideSpeedMenu() = Completable.mergeArray(
            fab.rotate(0f),
            longLabel.fadeOut().andThen(longFab.slideOut(fabMiniSize)),
            normalLabel.fadeOut().andThen(normalFab.slideOut(2 * fabMiniSize)),
            blitzLabel.fadeOut().andThen(blitzFab.slideOut(3 * fabMiniSize))
    )
```

The beauty of this is that after you describe both menus like this you can have chain hiding the first step with showing the second step for a beautifully choreographed animation involving 13 moving parts with just the following code:

```kotlin
    @OnClick(R.id.blitz_fab, R.id.long_fab, R.id.normal_fab)
    fun onSpeedClicked() {
        hideSpeedMenu()
                .doOnComplete { menuState = FabMenuState.SIZE }
                .andThen(showSizeMenu())
                .subscribe()
    }
```

Note how we could do both set the menu state and chain the animation at the same time which would have been hard to do within the limitations of having a single `withEndAction` listener.

Talking about the menu state, the only less nice part of the code is the one handling that variable. If you have comments on how to improve on that, I'm listening :)

```kotlin
    @OnClick(R.id.fab)
    fun onFabClicked() {
        menuState = when(menuState) {
            FabMenuState.OFF -> {
                showSpeedMenu().subscribe()
                FabMenuState.SPEED
            }
            FabMenuState.SPEED -> {
                hideSpeedMenu().subscribe()
                FabMenuState.OFF
            }
            FabMenuState.SIZE -> {
                hideSizeMenu().subscribe()
                FabMenuState.OFF
            }
        }
        fadeOutMask.showIf(menuState != FabMenuState.OFF)
    }
```

The full code is available in [MainActivity.kt](https://github.com/acristescu/RxAnimationExample/blob/master/app/src/main/java/io/zenandroid/rxanimationexample/MainActivity.kt).

## Conclusion

In this article we've seen how RxJava can be leveraged to make Android animations easier to write and understand. We've explored how and why we would create such a mix and we've also looked at a real-world example of such an animation. Let me know what your thoughts are about this technique.