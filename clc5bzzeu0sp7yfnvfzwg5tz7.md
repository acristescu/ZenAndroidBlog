# Mobile animations with Lottie

![Mobile animations with Lottie](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091302054/cTyWJaGqH.png align="left")

In this article I'm going to briefly review what options we have for animations for mobile apps in early 2018 and investigate the new kid on the block, the Lottie animation library from Airbnb. There will be a code sample available in this [repo](https://github.com/acristescu/LottieInvestigation).

## What we are trying to achieve

We're going to try to reproduce a toggle button that behaves like this:  

![Mobile animations with Lottie](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091303372/IBt2wnqyh.gif align="left")

We also want it to behave correctly when the user abuses it by toggling it repeatedly in a very short timespan:  

![Mobile animations with Lottie](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091304999/3vImADlgb.gif align="left")

## The olden way

When faced with animations, you generally have the option of creating them from code, using an animated vector resource or, as a last resort, having your graphical artist export a few frames of the animation as PNGs or similar formats. All of these have their (quite significant) disadvantages. Let's see them one at a time.

### Animations from code

This is the easiest solution if the animation is simple. For example, to rotate items on the screen you might employ `view.animate().rotation()`. As shown in my previous [article](http://zenandroid.io/building-complex-android-animations-with-rxjava/) on RxJava and animations, with a bit of reactive magic you can even string these together to create some nice effects.

The limitations of this method are that:

1. the complexity of the code increases quite rapidly with the number of moving parts
    
2. you are limited to a few things such as translations, rotations and fading in/out. What about morphing animations, such as when the hamburger menu icon morphs into an arrow?
    
3. your team will have to implement it differently for iOS and Android
    
4. very minor changes in the animation sometimes require massive changes in the code
    

### Animated vector

If you can get the animation into an animated vector XML you can easily display it into your app. All that is easier said than done though, because:

1. there are no good tools that support exporting this format. Shapeshifter might be a good bet, but it's not completed yet. Bodymovin (the same Adobe After Effects plugin discussed below) can export such files, but support is quite limited.
    
2. Different features are supported on different android versions, making their behaviour quite hard to predict. Supporting older android version (below L) is a major pain.
    
3. not supported on iOS
    

### Exporting individual frames as PNGs

This used to be pretty much your best bet until not long ago (and might still be, depending on circumstances). You get your graphical artist to supply you with a few frames and then use a thread to animate between frames. This is not desirable though because:

1. The result is going to be quite a jerky animation unless you export a LOT of frames (30 per seconds)
    
2. The result is going to be scaled (the fuzzy edge effect) unless you are going to export all possible densities.
    
3. The size of the APK is going to be HUGE. Consider that for a half a second animation loop at 30 FPS you would have to hardcode 5 x 30 / 2 = 75 PNGs.
    

## Enter Lottie

Lottie is the solution to this problem proposed by the AirBNB tech department. It is a free library that allows importing animation in .json format (that you can generate from Adobe After Effects via the Bodimovin plugin).

Here are its main "selling" points:

1. Allows you to offload animation creation and maintenance to the graphical artists, in a similar fashion that you do with icons and other such assets. With a correctly setup workflow, your artist can create an animation as an AEP file (the equivalent of a PSD file for images) and export it as a bodymovin json file (as they would a PNG file). Should they desire to change it later on, they could modify the AEP file, export it again and give the new .json file to you to put in the app.
    
2. Allows use of one of the best animation creation tool out there (Adobe After Effects)
    
3. It's fully cross-platform, meaning that the same animation resource can be used for both Android and iOS (and for that matter web).
    
4. Very simple to use (see example below).
    
5. Very small .json resource file (typically no bigger than a few hundreds of bytes)
    
6. There is a community-driven site for sharing lottie animations [lottiefiles.com](http://zenandroid.io:80/mobile-animations-with-lottie/lottiefiles.com)
    

Disadvantages:

1. not all features of AE are supported, although support is a lot more comprehensive than for Animated Vectors
    
2. slightly worse performance than Animated Vectors
    
3. Some glitches can still be observed in certain versions of Android
    

## Sample code

We're going to start by downloading an animation from LottieFiles (link) and putting the `toggle_switch.json` file into our `res/raw` directory. This is our animation resource, the equivalent of a `.svg` or `.png` file in our images analogy.

Next we're going to add the library dependency to our project and then sync the gradle files:

```java
implementation 'com.airbnb.android:lottie:2.5.1'
```

Once that is done, we add the following view to our layout:

```xml
        <com.airbnb.lottie.LottieAnimationView
            android:layout_marginLeft="10dp"
            android:id="@+id/animation_view"
            android:layout_width="60dp"
            android:layout_height="60dp"
            android:layout_alignParentRight="true"

            app:lottie_rawRes="@raw/toggle_switch"
            />
```

Note that we have to specify the json file we downloaded earlier as the animation to be shown in the last line.

Now, switching back to our Activity class we put in the following code (note that the `row` view is the parent of our `LottieAnimationView`, we only set the click listener there so that we can tap it without covering the animation with the finger).

```kotlin
class MainActivity : AppCompatActivity() {

    lateinit var row: View
    lateinit var animationView: LottieAnimationView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        animationView = findViewById(R.id.animation_view)
        row = findViewById<View>(R.id.row)

        animationView.apply {
            speed = -2.5f
            useHardwareAcceleration(true)
        }
        row.setOnClickListener {
            animationView.apply {
                speed = -speed
                setMinAndMaxFrame(0, 30)
                if (!isAnimating) {
                    playAnimation()
                }
            }
        }
    }
}
```

The setup part of the animation is done in the apply block. We set the speed to 2.5 since the animation is too slow for my tastes and initially a negative number since we're going to toggle that every time we tap. Finally, we call `useHardwareAcceleration(true)` to improve performance (note this does not work on some old android phones).

The onclick listener is what triggers the animation. What we do is we change the direction of the current animation by changing the speed to be either positive or negative. We make sure the animation only plays the first 30 frames (since the one we downloaded is longer than we need) and then make sure that the animation is started if not playing already. Pretty intuitive stuff.

Here is the result:

![Mobile animations with Lottie](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091306550/Ou-LKgylr.gif align="left")

## What else we can do

There are plenty of things that we can use lottie for besides toggle buttons. Let's see a few examples from [LottieFiles](http://lottiefiles.com):

1. Loading indicators  
    
    ![Mobile animations with Lottie](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091308631/joVbXubo5.gif align="left")
    
2. Splash screens transitions and/or logos  
    
    ![Mobile animations with Lottie](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091311148/OW7JJL34W.gif align="left")
    
3. Icons morphing into different ones  
    
    ![Mobile animations with Lottie](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091312526/UeHrOblVw.gif align="left")
    
4. Funny screens to ask permissions  
    
    ![Mobile animations with Lottie](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091314842/EaS4DNFOG.gif align="left")
    
5. Animated onboarding screens
    
6. Parallax effects (like Monzo's waiting for card delivery screen)
    

## Conclusion

We've reviewed the options we have for animations on Android at the beginning of 2018 and we've explored in more details what I think is the most promising of the alternatives: the Lottie library. We've also saw a simple example of how can we employ that in the code. Hope this encourages you guys to put more awesome animations into your apps.