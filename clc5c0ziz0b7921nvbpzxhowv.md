# Android Gesture Detection and Animations: creating a Swipe Button control

![Android Gesture Detection and Animations: creating a Swipe Button control](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091359440/EWozf3LvC.png align="left")

In this article we're going to look at how to create a production-grade Swipe Button control that has the following features:

1. tracks the finger with a transparent overlay
    
2. supports flinging
    
3. animates to a complete state
    
4. moves right while swiping and also causes other content on the screen to move creating a parralax effect
    
5. shakes when tapped so that it hints to the user that it needs to be swiped to activate
    
6. has a slide-out animation
    

The code is available in this [repo](https://github.com/acristescu/swipebutton) and a short capture of the result can be seen below:

![Android Gesture Detection and Animations: creating a Swipe Button control](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091361862/V-0eY6ybq.gif align="center")

## The plan

To start things off we're going to create a custom view, but we're not going to be extending `View` and drawing it from scratch. While that alternative may seem tempting, it would be very difficult to modify later down the line if the design changes. I instead opted to extend a `FrameLayout` and place several Views and `TextView` items inside it to give the general look and feel of the swipe button.

In order to animate other items on the screen (such as for example the grey `CardView` in the GIF above) we're going to publish an observable that produces a float between 0 (initial state) and 1 (fully swiped state). The enclosing activity can then orchestrate the other views on the screen to move in tandem with our swipe button. A second observable is going to notify the activity when the swiping gesture is performed and the animations have ended. Please note that I am using RxJava observables but you could easily use listeners for the same effect and a more "oldschool" approach.

We're going to employ a `GestureDetector` to determine when the user swipes, taps or flings over the button. We're going to also use the design library's `FlingAnimation` object to provide a realistic behavior when "flinging".

## Library roundup

The library section in the gradle file contains the usual suspects:

```java
dependencies {
    compile 'io.reactivex:rxandroid:1.2.1'
    compile 'io.reactivex:rxjava:1.2.1'
    compile 'com.android.support:cardview-v7:26.1.0'
    compile "com.android.support:support-dynamic-animation:26.1.0"
    compile 'com.jakewharton:butterknife:8.2.1'
    annotationProcessor "com.jakewharton:butterknife-compiler:8.2.1"
}
```

The only thing that is somewhat rarer here is the Dynamic Animation Library. This library contains a couple of useful physics based animations, such as `FlingAnimation` and `SpringAnimation`.

## The layout

I decided to go with a very flat layout with the different components stacking on top of eachother. The "button" part is a View, as is the white overlay that come on top of it. Between them we have a TextView that holds the text. Finally, the checkmark that appears when the swipe is complete sits on top of everything but is initially transparent.

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="54dp"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <View
        android:id="@+id/swipe_button_background"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_marginLeft="32dp"
        android:layout_marginRight="32dp"
        android:background="@drawable/rounded_corners_button"
        />

    <TextView
        android:id="@+id/swipe_text"
        android:textSize="20sp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Arrived at pick-up"
        android:textColor="#FFFFFF"
        android:layout_gravity="center"
        tools:ignore="MissingPrefix" />

    <View
        android:id="@+id/overlay"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:background="@drawable/rounded_corners_button_white"
        android:layout_marginLeft="32dp"
        />

    <android.support.v7.widget.AppCompatImageView
        android:id="@+id/swipe_check"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:srcCompat="@drawable/save_button"
        android:tint="#13c04d"
        android:alpha="0"
        android:paddingTop="15dp"
        android:paddingBottom="15dp"
        />
</FrameLayout>
```

## Gesture detection

For gesture detection we're going to go with the `GestureDetector` class. This is part of the Android package, but is less known. The way it works is that you instantiate it and forward it all the touch events and it provides a set of events you can listen to - such as `onScroll`, `onFling` etc.

```java
public class SwipeButton extends FrameLayout {
    private GestureDetector gestureDetector;

    private void init() {
        View view = inflate(getContext(), R.layout.swipe_button, this);
        unbinder = ButterKnife.bind(view);
        gestureDetector = new GestureDetector(getContext(), new GestureDetector.SimpleOnGestureListener() {
            @Override
            public boolean onSingleTapUp(MotionEvent motionEvent) {
                // ...
            }

            @Override
            public boolean onScroll(MotionEvent motionEvent, MotionEvent motionEvent1, float v, float v1) {
                // ...
            }

            @Override
            public boolean onFling(MotionEvent downEvent, MotionEvent moveEvent, float velocityX, float velocityY) {
                // ...
            }
        });
        gestureDetector.setIsLongpressEnabled(false);
    }

    @Override
    public boolean onTouchEvent(@NonNull MotionEvent event) {
        if(gestureDetector.onTouchEvent(event)) {
            return true;
        }
        return true;
    }
}
```

We'll be hooking up into the `onScroll` to provide feedback for the swiping progress, `onSingleTapUp` to make the button shake and `onFling` to animate the flinging gesture. As you can see we also hook up manually on the event of the user raising their finger since there is no equivalent gesture in the detector.

## Dragging progress

The position of the button, the width and opacity of the white overlay and the transparency of the text (in effect pretty much all the state) is controlled by a single variable that takes the values between 0 and the width of the entire layout. We also have a `triggered` boolean that disables all animations and touch events once the swipe is complete. Let's have a look at the function that accomplishes this.

```java
    private void setDragProgress(float x) {
        final int translation = calculateTranslation(x);
        setPadding(translation, 0, - translation, 0);
        if(!triggered) {
            overlayView.setAlpha(x / getWidth());
            overlayView.getLayoutParams().width = (int) Math.min(x - overlayView.getX() - translation, buttonBackground.getWidth());
            overlayView.requestLayout();

            textView.setAlpha(1 - overlayView.getAlpha());
        } else {
            overlayView.setAlpha(1);
            overlayView.getLayoutParams().width = buttonBackground.getWidth();
            overlayView.requestLayout();

            textView.setAlpha(0);
        }
    }

    private int calculateTranslation(float x) {
        return (int) x / 25;
    }
```

We achieve the move to the right of the entire layout by adding a padding to the left (and removing the same value from the right). We then compute the state of the subcomponents based on the progress value. What's left is to add the code to call this method when the user drags the finger over the layout (the `onScroll` method from the `GestureDetector`):

```java
            @Override
            public boolean onScroll(MotionEvent motionEvent, MotionEvent motionEvent1, float v, float v1) {
                cancelAnimations();
                setDragProgress(motionEvent1.getX());
                return true;
            }
```

## Handling the end of the drag

When the user finishes the dragging process (without flinging) the `GestureDetector` does not offer any callbacks. We do have the option of handling it ourselves inside the `onTouchEvent` method. The way the completed method looks is:

```java
    @Override
    public boolean onTouchEvent(@NonNull MotionEvent event) {
        if(triggered) {
            return true;
        }
        if(gestureDetector.onTouchEvent(event)) {
            return true;
        }
        switch (event.getAction()) {
            case MotionEvent.ACTION_UP:
                onDragFinished(event.getX());
                break;
        }

        return true;
    }
```

Based on the x coordinate of where the user released the drag event we then compute if the drag resulted in an activation of the button or not. In case it did, we animate the progress variable to the end state, then set the `triggered` flag and animate to the progress back to start (to reset the padding) while also fading in the checkmark. In case it was not above the threshold, we animate the progress variable directly to the start state.

```java
    private void onDragFinished(float finalX) {
        if(finalX > THRESHOLD_FRACTION * getWidth()) {
            animateToEnd(finalX);
        } else {
            animateToStart();
        }
    }

    private void animateToStart() {
        cancelAnimations();
        animator = ValueAnimator.ofFloat(overlayView.getWidth(), 0);
        animator.addUpdateListener(valueAnimator -> setDragProgress((Float)valueAnimator.getAnimatedValue()));
        animator.setDuration(ANIMATE_TO_START_DURATION);
        animator.start();
    }

    private void animateToEnd(float currentValue) {
        cancelAnimations();
        float rightEdge = buttonBackground.getWidth() + buttonBackground.getX();
        rightEdge += calculateTranslation(rightEdge);
        animator = ValueAnimator.ofFloat(currentValue, rightEdge);
        animator.addUpdateListener(valueAnimator -> setDragProgress((Float)valueAnimator.getAnimatedValue()));
        animator.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                triggered = true;
                checkmark.animate().alpha(1).setDuration(ANIMATE_TO_START_DURATION);
                animateToStart();
            }
        });
        animator.setDuration(ANIMATE_TO_END_DURATION);
        animator.start();
    }
```

## Handling the flinging gesture

Flinging is when a user quickly swipes the finger over the layout without moving all the way to the right. We want the movement of the white overlay to continue with some approximation of inertia before coming to a stop, kinda like throwing something along an abrasive surface. This behaviour is achieved by use of the `FlingAnimation` class in this manner:

```java
            @Override
            public boolean onFling(MotionEvent downEvent, MotionEvent moveEvent, float velocityX, float velocityY) {
                if(velocityX < 0) {
                    return false;
                }
                cancelAnimations();
                flingAnimation = new FlingAnimation(new FloatValueHolder(moveEvent.getX()));
                flingAnimation.setStartVelocity(velocityX)
                        .setMaxValue(getWidth())
                        .setFriction(FLING_FRICTION)
                        .addUpdateListener((dynamicAnimation, val, velocity) -> setDragProgress(val))
                        .addEndListener((dynamicAnimation, canceled, val, velocity) -> onDragFinished(val))
                        .start();

                return true;
            }
```

Note that we're reusing the `setDragProgress` and `onDragFinished` methods discussed above to act "as if" the user would continue the dragging process.

## Shake on tap

If we want the button to shake on tap, we can use the `GestureDetector` method `onSingleTapUp` event and animate the padding between several values. Please note the less common use of the varargs version of the factory method for the ValueAnimator that causes the padding to go from 0 to the max amplitude then back to 0 then half the amplitude, then to 0 again then to a quarter amplitude and then end up in 0. We also use an `AccelerateDecelerateInterpolator` to give the animation a more natural look.

```java
            @Override
            public boolean onSingleTapUp(MotionEvent motionEvent) {
                animateShakeButton();
                return true;
            }
//....
    private void animateShakeButton() {
        cancelAnimations();
        float rightEdge = buttonBackground.getWidth() + buttonBackground.getX();
        rightEdge += calculateTranslation(rightEdge);
        animator = ValueAnimator.ofFloat(0, rightEdge, 0, rightEdge / 2, 0, rightEdge / 4, 0);
        animator.setInterpolator(new AccelerateDecelerateInterpolator());
        animator.addUpdateListener(valueAnimator -> {
            final int translation = calculateTranslation((Float)valueAnimator.getAnimatedValue());
            setPadding(translation, 0, - translation, 0);
        });
        animator.setDuration(ANIMATE_SHAKE_DURATION);
        animator.start();
    }
```

## Publishing updates

In order for the activity to coordinate other views with the swiping of the button we publish two observables: one with the progress variable (normalized between 0 and 1) and one with an event for when the button is triggered and the animations finish. We achieve this with the following code:

```java
    private PublishSubject<Float> progressSubject = PublishSubject.create();
    private PublishSubject<Void> completeSubject = PublishSubject.create();

    private void setDragProgress(float x) {
        progressSubject.onNext(x / getWidth());
        //...
    }

    private void animateToStart() {
        cancelAnimations();
        animator = ValueAnimator.ofFloat(overlayView.getWidth(), 0);
        animator.addUpdateListener(valueAnimator -> setDragProgress((Float)valueAnimator.getAnimatedValue()));
        animator.setDuration(ANIMATE_TO_START_DURATION);
        animator.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                if(triggered) {
                    completeSubject.onNext(null);
                }
            }
        });
        animator.start();
    }

    public Observable<Float> getProgressObservable() {
        return progressSubject.asObservable();
    }

    public Observable<Void> getCompleteObservable() {
        return completeSubject.asObservable();
    }
```

Then, in the `MainActivity` we can add code such as:

```java
        swipeButton.getProgressObservable().subscribe(progress -> {
            final int translation = (int) (progress * cardContainer.getWidth() / 50f);
            cardContainer.setPadding(translation, 0, -translation, 0);
        });
        swipeButton.getCompleteObservable().subscribe(aVoid -> {
            final int activityHeight = findViewById(android.R.id.content).getHeight();
            swipeButton.animate().yBy(activityHeight - swipeButton.getY()).setDuration(SLIDE_OUT_DURATION);
            cardContainer.animate().yBy(activityHeight - cardContainer.getY()).setDuration(SLIDE_OUT_DURATION);
        });
```

Note that in order to achieve the parralax effect the amplitude of the translation of hte card container should be smaller than the one for the button itself.

## Putting it all together

Here is the entire code of the SwipeButton. A sample project can be found in the [repo](https://github.com/acristescu/swipebutton).

```java
public class SwipeButton extends FrameLayout {

    public static final double THRESHOLD_FRACTION = .85;
    public static final int ANIMATE_TO_START_DURATION = 300;
    public static final int ANIMATE_TO_END_DURATION = 200;
    public static final int ANIMATE_SHAKE_DURATION = 2000;
    public static final float FLING_FRICTION = .85f;

    private Unbinder unbinder;
    private GestureDetector gestureDetector;
    private ValueAnimator animator;
    private FlingAnimation flingAnimation;
    private boolean triggered = false;
    private PublishSubject<Float> progressSubject = PublishSubject.create();
    private PublishSubject<Void> completeSubject = PublishSubject.create();

    @BindView(R.id.overlay) View overlayView;
    @BindView(R.id.swipe_text) TextView textView;
    @BindView(R.id.swipe_button_background) View buttonBackground;
    @BindView(R.id.swipe_check) AppCompatImageView checkmark;

    public SwipeButton(@NonNull Context context) {
        super(context);
        init();
    }

    public SwipeButton(@NonNull Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public SwipeButton(@NonNull Context context, @Nullable AttributeSet attrs, @AttrRes int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    @TargetApi(Build.VERSION_CODES.LOLLIPOP)
    public SwipeButton(@NonNull Context context, @Nullable AttributeSet attrs, @AttrRes int defStyleAttr, @StyleRes int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
        init();
    }

    private void init() {
        View view = inflate(getContext(), R.layout.swipe_button, this);
        unbinder = ButterKnife.bind(view);
        gestureDetector = new GestureDetector(getContext(), new GestureDetector.SimpleOnGestureListener() {
            @Override
            public boolean onSingleTapUp(MotionEvent motionEvent) {
                animateShakeButton();
                return true;
            }

            @Override
            public boolean onScroll(MotionEvent motionEvent, MotionEvent motionEvent1, float v, float v1) {
                cancelAnimations();
                setDragProgress(motionEvent1.getX());
                return true;
            }

            @Override
            public boolean onFling(MotionEvent downEvent, MotionEvent moveEvent, float velocityX, float velocityY) {
                if(velocityX < 0) {
                    return false;
                }
                cancelAnimations();
                flingAnimation = new FlingAnimation(new FloatValueHolder(moveEvent.getX()));
                flingAnimation.setStartVelocity(velocityX)
                        .setMaxValue(getWidth())
                        .setFriction(FLING_FRICTION)
                        .addUpdateListener((dynamicAnimation, val, velocity) -> setDragProgress(val))
                        .addEndListener((dynamicAnimation, canceled, val, velocity) -> onDragFinished(val))
                        .start();

                return true;
            }
        });
        gestureDetector.setIsLongpressEnabled(false);
    }

    @Override
    public void onViewRemoved(View child) {
        super.onViewRemoved(child);
        unbinder.unbind();
    }

    @Override
    public boolean onTouchEvent(@NonNull MotionEvent event) {
        if(triggered) {
            return true;
        }
        if(gestureDetector.onTouchEvent(event)) {
            return true;
        }
        switch (event.getAction()) {
            case MotionEvent.ACTION_UP:
                onDragFinished(event.getX());
                break;
        }

        return true;
    }

    private void setDragProgress(float x) {
        progressSubject.onNext(x / getWidth());
        final int translation = calculateTranslation(x);
        setPadding(translation, 0, - translation, 0);
        if(!triggered) {
            overlayView.setAlpha(x / getWidth());
            overlayView.getLayoutParams().width = (int) Math.min(x - overlayView.getX() - translation, buttonBackground.getWidth());
            overlayView.requestLayout();

            textView.setAlpha(1 - overlayView.getAlpha());
        } else {
            overlayView.setAlpha(1);
            overlayView.getLayoutParams().width = buttonBackground.getWidth();
            overlayView.requestLayout();

            textView.setAlpha(0);
        }
    }

    private int calculateTranslation(float x) {
        return (int) x / 25;
    }

    private void cancelAnimations() {
        if(animator != null) {
            animator.cancel();
        }
        if(flingAnimation != null) {
            flingAnimation.cancel();
        }
    }

    private void onDragFinished(float finalX) {
        if(finalX > THRESHOLD_FRACTION * getWidth()) {
            animateToEnd(finalX);
        } else {
            animateToStart();
        }
    }

    private void animateToStart() {
        cancelAnimations();
        animator = ValueAnimator.ofFloat(overlayView.getWidth(), 0);
        animator.addUpdateListener(valueAnimator -> setDragProgress((Float)valueAnimator.getAnimatedValue()));
        animator.setDuration(ANIMATE_TO_START_DURATION);
        animator.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                if(triggered) {
                    completeSubject.onNext(null);
                }
            }
        });
        animator.start();
    }

    private void animateToEnd(float currentValue) {
        cancelAnimations();
        float rightEdge = buttonBackground.getWidth() + buttonBackground.getX();
        rightEdge += calculateTranslation(rightEdge);
        animator = ValueAnimator.ofFloat(currentValue, rightEdge);
        animator.addUpdateListener(valueAnimator -> setDragProgress((Float)valueAnimator.getAnimatedValue()));
        animator.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                triggered = true;
                checkmark.animate().alpha(1).setDuration(ANIMATE_TO_START_DURATION);
                animateToStart();
            }
        });
        animator.setDuration(ANIMATE_TO_END_DURATION);
        animator.start();
    }

    private void animateShakeButton() {
        cancelAnimations();
        float rightEdge = buttonBackground.getWidth() + buttonBackground.getX();
        rightEdge += calculateTranslation(rightEdge);
        animator = ValueAnimator.ofFloat(0, rightEdge, 0, rightEdge / 2, 0, rightEdge / 4, 0);
        animator.setInterpolator(new AccelerateDecelerateInterpolator());
        animator.addUpdateListener(valueAnimator -> {
            final int translation = calculateTranslation((Float)valueAnimator.getAnimatedValue());
            setPadding(translation, 0, - translation, 0);
        });
        animator.setDuration(ANIMATE_SHAKE_DURATION);
        animator.start();
    }

    public Observable<Float> getProgressObservable() {
        return progressSubject.asObservable();
    }

    public Observable<Void> getCompleteObservable() {
        return completeSubject.asObservable();
    }
}
```

## Conclusion

In this article we looked at how we can implement production-grade animation and gestures in Android. I hope it encourages you guys to try and experiment with more interactive views and transitions. I think there is a misconception that Android does not really support this sort of thing and I hope I managed to dispel that a little.