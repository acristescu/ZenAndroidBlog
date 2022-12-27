# The easy way to ask for runtime permissions: the PermissionDispatcher library

In this article I'm going to introduce my favourite way to handle permissions in Android, namely the `PermissionDispatcher` library. We're going to be looking at a simple example and hopefully illustrate why it is in my opinion the easiest way of requesting permissions. The full code is available [here](https://github.com/acristescu/PermissionsSample).

## The problem

As you are probably aware, starting with API 23 (Marshmallow) all permission requests must be done at runtime and not when the app is installed. While this has been a very good feature for the end users, it created a world of pain for the developers as the code to check if you had the permissions and to request it if you don't is convoluted, hard to read and error prone.

The proper way to handle it is that when the user performs an action that requires a permission (say, clicks a button to open the camera app) you are supposed to first check if you have the proper permissions and if so take action immediately. But if not, you have to request the permission and receive an asynchronous response later on. Then you check if the answer is yes and then continue the requested action. This gets even more complicated if the same activity requires more than one permission (for example picking an image from the gallery OR camera) since you have to remember which action the user is trying to accomplish and do it when the positive response comes in.

## An easier way

Enter [PermissionDispatcher](https://github.com/permissions-dispatcher/PermissionsDispatcher). This is an annotation-driven library that generates all the boilerplate code for you and hides it in a class that you will never see (similar to ButterKnife or Dagger). After the initial setup (which involves copy-pasting a few lines) you can then simply annotate your methods with `@NeedsPermission` and simply call them without worrying about the permissions. The library is completely reflection-free and quite easy to use.

## Setting it up

To use the library you have to do the following:

* Add the dependency inside your module's `build.grade` file:
    

```java
    compile "com.github.hotchemi:permissionsdispatcher:3.0.1"
    annotationProcessor "com.github.hotchemi:permissionsdispatcher-processor:3.0.1"
```

* Annotate your activity class with `@RuntimePermissions`:
    

```java
@RuntimePermissions
public class MainActivity extends AppCompatActivity {
//...
}
```

* Delegate the `onRequestPermissionsResult` method to the generated class:
    

```java
    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        // NOTE: delegate the permission handling to generated method
        MainActivityPermissionsDispatcher.onRequestPermissionsResult(this, requestCode, grantResults);
    }
```

* annotate the methods with `@NeedsPermission`:
    

```java
    @NeedsPermission(Manifest.permission.CAMERA)
    void showCamera() {
        // ...
    }
```

* change all calls to the affected methods (e.g. `showCamera()`) to go through the generated class instead:
    

```java
    private void onRequestButtonClicked() {
        // instead of showCamera()
        MainActivityPermissionsDispatcher.showCameraWithPermissionCheck(this);
    }
```

While that's arguably not trivial, it is a lot simpler and easier to understand than the alternative and the process should not feel so strange to devs that have used other annotation-driven libraries in the past (for example Dagger 2).

> Note: `MainActivityPermissionsDispatcher` in the example above is the generated class that is a companion of our `MainActivity` class. It contains `onRequestPermissionsResult` delegate that needs to be called from your activity's `onRequestPermissionsResult` and also a static `<methodName>WithPermissionCheck` for each metod annotated with `@NeedPermission`.

## Optional bits

The library offers 3 more callbacks that allow you to improve the user experience and increase the chances that he will play ball and accept your request:

* Show Rationale: You can annotate one method with `@OnShowRationale` and it will get called when the android system thinks it's a good moment for you to explain to the user why you need the request. Usually this happens when the user repeatedly tries to use the feature but keeps denying the permission request. You would usually display a dialog to the user here:
    

```java
    @OnShowRationale(Manifest.permission.CAMERA)
    void showRationaleForCamera(final PermissionRequest request) {
        new AlertDialog.Builder(this)
                .setMessage("Use this (optional) dialog to explain to the user why you need the permission")
                .setPositiveButton("OK", (dialog, button) -> request.proceed())
                .setNegativeButton("Cancel", (dialog, button) -> request.cancel())
                .show();
    }
```

* On permission denied: if you annotate one method with `@OnPermissionDenied` it will get called when the user denies your request so you can take appropriate action.
    

```java
    @OnPermissionDenied(Manifest.permission.CAMERA)
    void showDeniedForCamera() {
        Toast.makeText(this, "User denied the permission request", Toast.LENGTH_SHORT).show();
    }
```

* On never ask again: Annotate a method with `@OnNeverAskAgain` if you want to be notified when the user has ticked the `Never Ask Again` checkbox, hence preventing your app from ever asking for the permission again. There isn't much you can do here except perhaps disabling the functionality alltogether.
    

```java
    @OnNeverAskAgain(Manifest.permission.CAMERA)
    void showNeverAskForCamera() {
        Toast.makeText(this, "User clicked 'don't ask again'", Toast.LENGTH_SHORT).show();
    }
```

## A full example

Let's look at a simple example. We'll place a single button that, when clicked, tries to open the camera and immediately close it. Without permission checking, the app would simply crash when pressing the button on an emulator with API &gt;= 23, but with the few lines of code shown below the user is asked for the permission and all the states are correctly handled. Please note that once the user has accepted the permissions he is never asked again (even if we restart the app). Full code available [here](https://github.com/acristescu/PermissionsSample).

![](/content/images/2017/11/permissions1.gif align="left")

The layout is pretty self explanatory:

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="io.zenandroid.permissionssample.MainActivity">

    <TextView
        android:id="@+id/textView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Press the button to request camera permission"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="8dp"
        android:text="Request permission"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/textView" />

</android.support.constraint.ConstraintLayout>
```

The Activity:

```java
@RuntimePermissions
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        findViewById(R.id.button).setOnClickListener(button -> onRequestButtonClicked());
    }

    private void onRequestButtonClicked() {
        // instead of showCamera()
        MainActivityPermissionsDispatcher.showCameraWithPermissionCheck(this);
    }

    @NeedsPermission(Manifest.permission.CAMERA)
    void showCamera() {
        //
        // If we reached this point, we have the permissions. We'll just open and close the camera
        // object to illustrate the point.
        //
        try {
            Camera.open().release();
        } catch (Exception ex) {
            Toast.makeText(this, "Cannot open camera: " + ex.getMessage(), Toast.LENGTH_SHORT).show();
        }
        Toast.makeText(this, "Camera opened", Toast.LENGTH_SHORT).show();
    }

    // This is optional!
    @OnPermissionDenied(Manifest.permission.CAMERA)
    void showDeniedForCamera() {
        Toast.makeText(this, "User denied the permission request", Toast.LENGTH_SHORT).show();
    }

    // This is optional!
    @OnShowRationale(Manifest.permission.CAMERA)
    void showRationaleForCamera(final PermissionRequest request) {
        new AlertDialog.Builder(this)
                .setMessage("Use this (optional) dialog to explain to the user why you need the permission")
                .setPositiveButton("OK", (dialog, button) -> request.proceed())
                .setNegativeButton("Cancel", (dialog, button) -> request.cancel())
                .show();
    }

    // This is optional!
    @OnNeverAskAgain(Manifest.permission.CAMERA)
    void showNeverAskForCamera() {
        Toast.makeText(this, "User clicked 'don't ask again'", Toast.LENGTH_SHORT).show();
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        // NOTE: delegate the permission handling to generated method
        MainActivityPermissionsDispatcher.onRequestPermissionsResult(this, requestCode, grantResults);
    }
}
```

And that's all there is to it! You can browse the full code in this repo or you can read the full docs of the library here. Happy Coding!