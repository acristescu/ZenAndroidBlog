# Android Things - Part 3: Creating a remote control app

In the third and last part of this series of articles on the Android Things platform, we look at creating a companion app for the Raspberry Pi Android module we created in [part 2](http://zenandroid.io/android-things-2-controlling-a-led/). This companion app will run on a normal Android phone or tablet and will connect to the Firebase Database to display real-time information on the status of the LEDs from the fleet of devices and allow us to toggle the status of any of those LEDs in almost real time.

\[TOC\]

# The app

Technically, this app can use any of the platforms supported by Firebase (for example it could be a Web App), however, we're going to be building an Android app that will allow you to control the led using an android phone. It will consist of a single screen containing a list of the known devices, each showing the status of their LED. Clicking on a device will cause the led of the device to change state.

## Creating an extra module

Although not very common, Android Studio and Gradle allow you to create several distinct modules in the same project. We're going to use that feature to create the remote control app.

1. Click **File -&gt; New -&gt; New module...**
    
2. Choose **Phone/Tablet**
    
3. Name the new application `remote`
    
4. Choose an empty Activity and name it `RemoteControlActivity`.
    

## Adding Firebase to the remote module

> **Note:** The following steps are quite similar to the ones we did previously for the Android Things module, however you must repeat them for each module, since each module has a different package and hence a different JSON file.

1. Install the [Firebase Android SDK](https://firebase.google.com/docs/android/setup) into your app project. The simplest way is to click **Tools &gt; Firebase** to open the Assistant window, select **Database** and follow the wizard.
    
2. In the [Firebase console](https://firebase.google.com/console/), select **Import Google Project** to import the Google Cloud project you created using the Assistant into Firebase.
    
3. Download and install the `google-services.json` file as described in the instructions.
    
4. Add the Firebase Realtime Database dependency to your app-level `build.gradle` file:
    

```java
dependencies {
//    ...

    compile 'com.google.firebase:firebase-core:9.6.1'
    compile 'com.google.firebase:firebase-database:9.6.1'
    compile 'com.firebaseui:firebase-ui-database:0.5.3'
}
```

> **Note**: we're adding the extra dependency on the `firebase-ui-database` which we'll be using to simplify the implementation.

## Model classes

The remote app exchanges the following two model classes with Firebase Database:

```java
public class Status {
	private boolean ledOn;

	public boolean isLedOn() {
		return ledOn;
	}

	public void setLedOn(boolean ledOn) {
		this.ledOn = ledOn;
	}

}
```

```java
public class Device {
	private Status currentStatus;
	private Status desiredStatus;

	public Status getDesiredStatus() {
		return desiredStatus;
	}

	public void setDesiredStatus(Status desiredStatus) {
		this.desiredStatus = desiredStatus;
	}

	public Status getCurrentStatus() {
		return currentStatus;
	}

	public void setCurrentStatus(Status currentStatus) {
		this.currentStatus = currentStatus;
	}
}
```

## Firebase UI Adapter

The Firebase UI library offers an implementation of RecyclerView adapter that responds immediately to changes in the Firebase Database. This makes it easier to display Firbase Data since we don't have to listen to the events ourselves.

```java
public class DeviceAdapter extends FirebaseRecyclerAdapter<Device, DeviceAdapter.DeviceViewHolder> {
	public static class DeviceViewHolder extends RecyclerView.ViewHolder {
		@BindView(R.id.indicator) AppCompatImageView indicator;
		@BindView(R.id.text) TextView text;

		public DeviceViewHolder(View v) {
			super(v);
			ButterKnife.bind(this, v);
		}
	}

	public DeviceAdapter(DatabaseReference reference) {
		super(Device.class, R.layout.item_device, DeviceViewHolder.class, reference);
	}

	@Override
	protected void populateViewHolder(DeviceViewHolder viewHolder, final Device device, final int position) {
		if(device.getCurrentStatus() != null && device.getCurrentStatus().isLedOn()) {
			viewHolder.indicator.setImageResource(R.drawable.ic_led_on);
		} else {
			viewHolder.indicator.setImageResource(R.drawable.ic_led_off);
		}
		viewHolder.text.setText(String.format(Locale.getDefault(), "Device %d", position));
		viewHolder.itemView.setOnClickListener(new View.OnClickListener() {
			@Override
			public void onClick(View view) {
				Status newStatus = new Status();
				newStatus.setLedOn(!device.getCurrentStatus().isLedOn());
				device.setDesiredStatus(newStatus);

				getRef(position).setValue(device);
			}
		});
	}

}
```

## The remote control activity

The last piece of the puzzle is implementing the Activity. Since all the logic is handled by the adapter, the code here is quite trivial

```java
public class RemoteControlActivity extends AppCompatActivity {
	@BindView(R.id.recycler) RecyclerView mRecycler;

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_remote_control);

		ButterKnife.bind(this);

		mRecycler.setLayoutManager(new LinearLayoutManager(this));
		mRecycler.setAdapter(new DeviceAdapter(FirebaseDatabase.getInstance().getReference()));
		mRecycler.addItemDecoration(new DividerItemDecoration(this, DividerItemDecoration.VERTICAL));
	}

}
```

# Bringing it all together

You can run the remote control app by selecting the **remote** module in the module drop-down (to the left of the **Run** button) and then pressing the **Run** button in Android Studio. Select a connected Android phone or emulator and wait for the app to be deployed. Once it starts, you should see a single item in the list, corresponding to the device. The status of the LED should be reflected in the app.

Tapping the device in the app should toggle the state of the led. Please note there is a small delay (depending on your network latency) between the moment you tap and the moment the icon changes. For me it was ~100ms. This is due to the fact that the icon does not change until Firebase notifies the app the status changed, and this takes two roundtrips + the processing time in the remote, Firebase and the Raspberry Pi. The response time is quite small all things considering.

As a last test, start the remote app on two different phones (or a phone and an emulator). Notice that tapping the device on one is almost instantly reflected on the other (the event is broadcast).

# Conclusion

In these 3 articles we've explored a simple application of the Android Things platform. As a personal opinion, I've found the platform to be quite stable and easy to use. With very few exceptions (e.g. Google Maps) you can implement on the Pi anything you can implement on an Android phone with very little fuss.

On the minus side, some of the drivers for the hardware is not yet present (e.g. RC522 RFID reader) - but if the platform becomes successful, they will surely appear. Also, Android does require a bit of horsepower, so it's hard to imagine running this on some of the less powerful IoT devices, however even for those there are advantages in using the Android Things as an early prototype to prove that your idea.