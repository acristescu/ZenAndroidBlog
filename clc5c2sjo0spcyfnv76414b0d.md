# Writing a driver for Android Things: BME280 Humidity sensor

In this article we're going to investigate how to write a device driver for Android Things. In particular, we're going to extend the functionality offered in Google's [contrib driver](https://github.com/androidthings/contrib-drivers/tree/master/bmx280) package for the BMX280 temperature and ambient pressure to allow for humidity measurements as well.

We're also going to modify the driver so that it allows different I2C addresses, since the contrib driver hardcodes that to 0x77.

The full code for this project is available from this [repo](https://github.com/acristescu/pi-temperature/tree/article1).

## Hardware requirements

For this article we're going to be using:

* a Raspberry Pi 3 with Android Things installed. Check out this [article](http://zenandroid.io/android-things-1/) if you need a primer on how to get set up.
    
* a BME240 board, such as [this one from Amazon](https://www.amazon.co.uk/HALJIA-GY-BME280-3-3-Precision-Barometric-Temperature/dp/B01MUD07SX) that costed me about £8.
    
* 4 [female-female breadboard wires](https://www.amazon.co.uk/d/Audio-Video-Cables/SODIAL-Female-Solderless-Flexible-Breadboard/B00HUH9GOC).
    

## The problem

The need for this exercise comes from the fact that Google's [contrib drivers](https://github.com/androidthings/contrib-drivers/tree/master/bmx280) for this ambient sensor is meant to be used on both the BMP280 and BME280 (hence they call it the BMX280). The difference between the two sensors is that the BME also has the ability to measure humidity (besides temperature and ambient pressure), a feature that is not supported in Google's driver.

Furthermore, the driver hardcodes the address of the sensor to `0x77`, although the chip itself can also support `0x76`, and some escape boards (such as [this one from Amazon](https://www.amazon.co.uk/HALJIA-GY-BME280-3-3-Precision-Barometric-Temperature/dp/B01MUD07SX)) come with this latter address (and no way to change it). Our modifications will allow for an arbitrary address.

## Connecting things up

We're going to use the I2C protocol for communicating with the sensor. In order to determine which pins to connect, we're going to be referencing both the [Raspberry Pi pin-out diagram](https://developer.android.com/things/images/pinout-raspberrypi.png) and the labels on the escape board itself.

We need to connect 4 wires:

* the pin labeled VCC on the board connects to any `3.3V` pin on the Raspberry (marked with orange in the diagram) such as pin 1.
    
* the pin labeled GND on the board connects to any `Ground` pin on the Raspberry (marked with black in the diagram) such as pin 6.
    
* the pin labeled SDI (or SDA on some boards) connects to the `I2C1 (SDA)` pin on the Raspberry - pin 3.
    
* the pin labeled SCK (or SCL on some boards) connects to the `I2C2 (SCL)` pin on the Raspberry - pin 5.
    

Once that is setup we're ready to connect to the device.

![](/content/images/2017/03/IMG_20170303_102515.jpg align="left")

## Testing connectivity

Let's go back to Android Studio and test our handiwork. Let's set up a new project (like we did in this [article](http://zenandroid.io/android-things-2-controlling-a-led/)) and name the main activity `MainActivity`. First thing, we're going to open the device and try to read its device ID. Looking at the BME280 [datasheet](https://cdn-shop.adafruit.com/datasheets/BST-BME280_DS001-10.pdf) page 25, we find that:

> **Register 0xD0 “id”**: The “id” register contains the chip identification number chip\_id\[7:0\], which is 0x60. This number can be read as soon as the device finished the power-on-reset.

This means that if we read register number `0xD0` we should be getting back a response of `0x60` for the BME280. Let's code that up:

```java
public class MainActivity extends Activity {

	private static final String TAG = "MainActivity";
	private static final int ADDRESS = 0x76;
	private final PeripheralManagerService managerService = new PeripheralManagerService();

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);

		printDeviceId();
	}

	private void printDeviceId() {
		List<String> deviceList = managerService.getI2cBusList();
		if (deviceList.isEmpty()) {
			Log.i(TAG, "No I2C bus available on this device.");
		} else {
			Log.i(TAG, "List of available devices: " + deviceList);
		}
		I2cDevice device = null;
		try {
			device = managerService.openI2cDevice(deviceList.get(0), ADDRESS);
			Log.d(TAG, "Device ID byte: 0x" + Integer.toHexString(device.readRegByte(0xD0)));
		} catch (IOException|RuntimeException e) {
			Log.e(TAG, e.getMessage(), e);
		} finally {
			try {
				device.close();
			} catch (Exception ex) {
				Log.d(TAG, "Error closing device");
			}
		}
	}
}
```

> **Note**: I'm assuming the device address is `0x76`, as is in the break-out board I purchased. However, do note that the BME chip itself supports either `0x77` or `0x76`, with some boards even allowing selecting between these two addresses via a jumper. In case your board documentation does not state which address to use, the best thing is to change the `private static final int ADDRESS = 0x76` line and test out both of them and see which works. In case none of them does, check the wiring scheme again.

Here is the output if everything works correctly:

```java
02-22 12:57:51.884 9730-9730/com.example.pitepmerature W/System: ClassLoader referenced unknown path: /data/app/com.example.pitepmerature-1/lib/arm
02-22 12:57:51.977 9730-9730/com.example.pitepmerature I/MainActivity: List of available devices: [I2C1]
02-22 12:57:51.981 9730-9730/com.example.pitepmerature D/MainActivity: Device ID byte: 0x60
```

## Reading temperature and pressure

Now that we determined the sensor is wired up correctly, let's proceed to bring in the contrib driver and read a few temperature and pressure measurements (remember, the driver does not support humidity readings).

Let's start by adding the library to the `build.gradle`: `groovy dependencies { // ... compile 'com.google.android.things.contrib:driver-bmx280:0.1' }` Next, we will extend the `Bmx280` class with our own `Bme280` class. For now, we only do this in order to expose a constructor that supports another address that `0x77`, but later on we will be adding more functionality to it.

> **Hack alert**: we will be putting the `Bme280` class into the package `com.google.android.things.contrib.driver.bmx280` in order to be able to use one of the "package protected" constructors for the `Bmx280`. Due to the architecture of the driver, the only alternative would be to copy the entire code into our codebase. Hopefully, Google will change this in the future.

```java
package com.google.android.things.contrib.driver.bmx280;
//...
public class Bme280 extends Bmx280 {

	public Bme280(I2cDevice device) throws IOException {
		super(device);
	}
}
```

Now we can modify our `MainActivity` to use the driver:

```java
public class MainActivity extends Activity {

	private static final String TAG = "MainActivity";
	private static final int ADDRESS = 0x76;
	private final PeripheralManagerService managerService = new PeripheralManagerService();

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);

		printDeviceId();
		readSample();
	}

	private void readSample() {
		try (Bme280 bmxDriver = new Bme280(managerService.openI2cDevice(managerService.getI2cBusList().get(0), ADDRESS))){
			bmxDriver.setTemperatureOversampling(Bmx280.OVERSAMPLING_1X);
			bmxDriver.setPressureOversampling(Bmx280.OVERSAMPLING_1X);

			bmxDriver.setMode(Bme280.MODE_NORMAL);
			for(int i = 0 ; i < 5 ; i++) {
				try {
					Thread.sleep(2000);
				} catch (InterruptedException e) {
				}
				Log.d(TAG, "Temperature: " + bmxDriver.readTemperature());
				Log.d(TAG, "Pressure: " + bmxDriver.readPressure());
			}


		} catch (IOException e) {
			Log.e(TAG, "Error during IO", e);
			// error reading temperature
		}
	}

//...
}
```

Lets run it:

```java
02-22 13:34:17.945 7196-7196/com.example.pitepmerature I/MainActivity: List of available devices: [I2C1]
02-22 13:34:17.950 7196-7196/com.example.pitepmerature D/MainActivity: Device ID byte: 0x60
02-22 13:34:20.034 7196-7196/com.example.pitepmerature D/MainActivity: Temperature: 24.499886
02-22 13:34:20.037 7196-7196/com.example.pitepmerature D/MainActivity: Pressure: 1003.63416
02-22 13:34:22.065 7196-7196/com.example.pitepmerature D/MainActivity: Temperature: 24.51503
02-22 13:34:22.068 7196-7196/com.example.pitepmerature D/MainActivity: Pressure: 1003.60455
02-22 13:34:24.094 7196-7196/com.example.pitepmerature D/MainActivity: Temperature: 24.520079
02-22 13:34:24.098 7196-7196/com.example.pitepmerature D/MainActivity: Pressure: 1003.58563
02-22 13:34:26.124 7196-7196/com.example.pitepmerature D/MainActivity: Temperature: 24.525127
02-22 13:34:26.127 7196-7196/com.example.pitepmerature D/MainActivity: Pressure: 1003.70374
02-22 13:34:28.154 7196-7196/com.example.pitepmerature D/MainActivity: Temperature: 24.525127
02-22 13:34:28.157 7196-7196/com.example.pitepmerature D/MainActivity: Pressure: 1003.53925
```

## Adding humidity readings to the driver

The trickiest part of writing this driver is undoubtedly getting the calibration calculations right, so let's get that out of the way first.

### Calibration formula

Let's have a look at the [datasheet](https://cdn-shop.adafruit.com/datasheets/BST-BME280_DS001-10.pdf). On page 49 we find `Compensation formulas in double precision floating point`. We're going to be using those, since they offer the best precision at the cost of processing power, of which our Raspberry Pi has plenty. Sadly, the provided code is in C, so we'll have to translate it to Java. Let's have a look at the code for the humidity compensation:

```c
// Returns humidity in %rH as as double. Output value of “46.332” represents 46.332 %rH
double bme280_compensate_H_double(BME280_S32_t adc_H);
{
	double var_H;
	var_H = (((double)t_fine) – 76800.0);
	var_H = (adc_H – (((double)dig_H4) * 64.0 + ((double)dig_H5) / 16384.0 * var_H)) *
		(((double)dig_H2) / 65536.0 * (1.0 + ((double)dig_H6) / 67108864.0 * var_H *
		(1.0 + ((double)dig_H3) / 67108864.0 * var_H)));
	var_H = var_H * (1.0 – ((double)dig_H1) * var_H / 524288.0);
	if (var_H > 100.0)
		var_H = 100.0;
	else if (var_H < 0.0)
		var_H = 0.0;
	return var_H;
}
```

If you haven't yet ran away in abject terror at the sight of that piece of code, let's look at what it does. It takes a 32 bytes signed raw reading from the sensor and uses some fancy formulas (containing plenty of magic numbers) to refine it into a more precise reading. It does also use the last compensated temperature reading (`t_fine`) as well as a few parameters (aka trimming parameters: `dig_H1` through `dig_H6`) which we will have to read from the chip.

Let's look at how the function would look like in Java:

```java
	// Compensation formula from the BME280 datasheet.
	// https://cdn-shop.adafruit.com/datasheets/BST-BME280_DS001-10.pdf
	float compensateHumidity(int adc_H, float temp)
	{
		int dig_H1 = mHumCalibrationData[0];
		int dig_H2 = mHumCalibrationData[1];
		int dig_H3 = mHumCalibrationData[2];
		int dig_H4 = mHumCalibrationData[3];
		int dig_H5 = mHumCalibrationData[4];
		int dig_H6 = mHumCalibrationData[5];

		float var_H;
		var_H = (temp - 76800f);
		var_H = (adc_H - (((float)dig_H4) * 64f + ((float)dig_H5) / 16384f * var_H)) *
		(((float)dig_H2) / 65536f * (1f + ((float)dig_H6) / 67108864f * var_H *
				(1f + ((float)dig_H3) / 67108864f * var_H)));
		var_H = var_H * (1f - ((float)dig_H1) * var_H / 524288f);
		if (var_H > 100)
			var_H = 100f;
		else if (var_H < 0)
			var_H = 0f;
		return var_H;
	}
```

Note I have changed the temperature reading to be passed as a parameter to the function and that we have to actually populate the `mHumCalibrationData` array with the real data.

### Reading the trimming parameters

To read the parameters we turn to the datasheet document again, this time to pages 22 and 23, `Trimming parameter readout` chapter. The last 6 lines of the table there describe where to read those parameters from.

```java
Register Address		Register content 			Data type

0xA1					dig_H1 [7:0] 				unsigned char
0xE1 / 0xE2 			dig_H2 [7:0] / [15:8]		signed short
0xE3					dig_H3 [7:0]				unsigned char
0xE4 / 0xE5[3:0]		dig_H4 [11:4] / [3:0]		signed short
0xE5[7:4] / 0xE6		dig_H5 [3:0] / [11:4]		signed short
0xE7					dig_H6						signed char
```

As you can see, that requires a bit of bit-fu to get all the bits in the right order. This sort of thing is a bit more low-level than what most Android programmers are used to so please correct me if I got things wrong. This is the code I came up with:

```java
	private static final int BME280_REG_HUM_CALIB_1 = 0xA1;
	private static final int BME280_REG_HUM_CALIB_2 = 0xE1;
	private static final int BME280_REG_HUM_CALIB_3 = 0xE3;
	private static final int BME280_REG_HUM_CALIB_4 = 0xE4;
	private static final int BME280_REG_HUM_CALIB_5 = 0xE5;
	private static final int BME280_REG_HUM_CALIB_6 = 0xE6;
	private static final int BME280_REG_HUM_CALIB_7 = 0xE7;
//...
	private void readHumidityCalibrationData() throws IOException {
		mHumCalibrationData[0] = mDevice.readRegByte(BME280_REG_HUM_CALIB_1) & 0xFF;
		mHumCalibrationData[1] = mDevice.readRegWord(BME280_REG_HUM_CALIB_2);
		mHumCalibrationData[2] = mDevice.readRegByte(BME280_REG_HUM_CALIB_3) & 0xFF;

		int E4 = mDevice.readRegByte(BME280_REG_HUM_CALIB_4) & 0xFF;
		int E5 = mDevice.readRegByte(BME280_REG_HUM_CALIB_5) & 0xFF;
		int E6 = mDevice.readRegByte(BME280_REG_HUM_CALIB_6) & 0xFF;
		int E7 = mDevice.readRegByte(BME280_REG_HUM_CALIB_7);

		mHumCalibrationData[3] = (E4 << 4) | (E5 & 0x0F);
		mHumCalibrationData[4] = ((E5 & 0xF0) << 4) | E6;
		mHumCalibrationData[5] = E7;

		Log.d(TAG, Arrays.toString(mHumCalibrationData));
	}
```

That last line outputs values such as:

```java
02-22 13:34:17.984 7196-7196/com.example.pitepmerature D/Bme280: [75, 356, 0, 333, 0, 30]
```

### Setting the oversampling mode

Before we can actually read samples from the BME we have to first set up it's oversampling mode. All we need to do is write the value to the appropriate register, which we get from the datasheet page 26:

```java
	private static final int BME280_REG_CTRL_HUM = 0xF2;
//...
	public void setHumidityOversampling(@Oversampling int oversampling) throws IOException {
		mDevice.writeRegByte(BME280_REG_CTRL_HUM, (byte)(oversampling));
		mHumidityOversampling = oversampling;
	}
```

### Reading a humidity measurement

Let's now look at the datasheet, page 29. It states that Register 0xFD…0xFE “hum” (\_msb, \_lsb) contains the humidity reading.

```java
	private final byte[] mBuffer = new byte[2]; // for reading sensor values
//...
	public float readHumidity() throws IOException, IllegalStateException {
		if (mHumidityOversampling == OVERSAMPLING_SKIPPED) {
			throw new IllegalStateException("temperature oversampling is skipped");
		}
		int rawHum = readSample(BME280_REG_HUM);
		return compensateHumidity(rawHum, readTemperature());
	}

	private int readSample(int address) throws IOException, IllegalStateException {
		if (mDevice == null) {
			throw new IllegalStateException("I2C device is already closed");
		}
		synchronized (mBuffer) {
			mDevice.readRegBuffer(address, mBuffer, 2);
			// msb[7:0] lsb[7:0]
			int msb = mBuffer[0] & 0xff;
			int lsb = mBuffer[1] & 0xff;
			return msb << 8 | lsb;
		}
	}
```

### Finishing touches

Make sure everything is in place and that the constructor reads the calibration data before any actual measurements are made.

```java
public class Bme280 extends Bmx280 {
	private static final String TAG = "Bme280";

	private static final int BME280_REG_HUM_CALIB_1 = 0xA1;
	private static final int BME280_REG_HUM_CALIB_2 = 0xE1;
	private static final int BME280_REG_HUM_CALIB_3 = 0xE3;
	private static final int BME280_REG_HUM_CALIB_4 = 0xE4;
	private static final int BME280_REG_HUM_CALIB_5 = 0xE5;
	private static final int BME280_REG_HUM_CALIB_6 = 0xE6;
	private static final int BME280_REG_HUM_CALIB_7 = 0xE7;

	private static final int BME280_REG_CTRL_HUM = 0xF2;
	private static final int BME280_REG_HUM = 0xFD;

	private I2cDevice mDevice;
	private final byte[] mBuffer = new byte[2];

	private int mHumidityOversampling;

	public Bme280(I2cDevice device) throws IOException {
		super(device);
		mDevice = device;
		readHumidityCalibrationData();
	}
//...
}
```

Now let's go back to `MainActivity` and add the calls to read the humidity as well:

```java
	private void readSample() {
		try (Bme280 bmxDriver = new Bme280(managerService.openI2cDevice(managerService.getI2cBusList().get(0), ADDRESS))){
			bmxDriver.setTemperatureOversampling(Bmx280.OVERSAMPLING_1X);
			bmxDriver.setPressureOversampling(Bmx280.OVERSAMPLING_1X);
			bmxDriver.setHumidityOversampling(Bmx280.OVERSAMPLING_1X);

			bmxDriver.setMode(Bme280.MODE_NORMAL);
			for(int i = 0 ; i < 5 ; i++) {
				try {
					Thread.sleep(2000);
				} catch (InterruptedException e) {
				}
				Log.d(TAG, "Temperature: " + bmxDriver.readTemperature());
				Log.d(TAG, "Pressure: " + bmxDriver.readPressure());
				Log.d(TAG, "Humidity: " + bmxDriver.readHumidity());
			}


		} catch (IOException e) {
			Log.e(TAG, "Error during IO", e);
			// error reading temperature
		}
	}
```

Run the app again. Here is a sample output:

```java
02-22 15:06:51.392 17989-17989/com.example.pitepmerature I/MainActivity: List of available devices: [I2C1]
02-22 15:06:51.397 17989-17989/com.example.pitepmerature D/MainActivity: Device ID byte: 0x60
02-22 15:06:51.434 17989-17989/com.example.pitepmerature D/Bme280: [75, 356, 0, 333, 0, 30]
02-22 15:06:53.474 17989-17989/com.example.pitepmerature D/MainActivity: Temperature: 23.581142
02-22 15:06:53.478 17989-17989/com.example.pitepmerature D/MainActivity: Pressure: 1002.87085
02-22 15:06:53.481 17989-17989/com.example.pitepmerature D/MainActivity: Humidity: 59.578003
02-22 15:06:55.515 17989-17989/com.example.pitepmerature D/MainActivity: Temperature: 23.581142
02-22 15:06:55.518 17989-17989/com.example.pitepmerature D/MainActivity: Pressure: 1002.80756
02-22 15:06:55.521 17989-17989/com.example.pitepmerature D/MainActivity: Humidity: 59.578003
02-22 15:06:57.544 17989-17989/com.example.pitepmerature D/MainActivity: Temperature: 23.581142
02-22 15:06:57.548 17989-17989/com.example.pitepmerature D/MainActivity: Pressure: 1002.88983
02-22 15:06:57.550 17989-17989/com.example.pitepmerature D/MainActivity: Humidity: 59.578003
02-22 15:06:59.574 17989-17989/com.example.pitepmerature D/MainActivity: Temperature: 23.581142
02-22 15:06:59.577 17989-17989/com.example.pitepmerature D/MainActivity: Pressure: 1002.8624
02-22 15:06:59.580 17989-17989/com.example.pitepmerature D/MainActivity: Humidity: 59.578003
02-22 15:07:01.614 17989-17989/com.example.pitepmerature D/MainActivity: Temperature: 23.591238
02-22 15:07:01.617 17989-17989/com.example.pitepmerature D/MainActivity: Pressure: 1002.7972
02-22 15:07:01.620 17989-17989/com.example.pitepmerature D/MainActivity: Humidity: 59.578003
```

## Updating the user driver

Sadly, the user driver class (`Bmx280SensorDriver`) cannot be extended since the only constructor assumes the address to be `0x77`. The only workaround I found is simply to copy over the code of the class into our own class and add the code to it. Let's name the new class `Bme280SensorDriver`. We will follow the same pattern the original authors used for temperature and pressure, but remove the constructor that hardcodes the address and add the following instead:

```java
public class Bme280SensorDriver implements AutoCloseable {
//...
	private HumidityUserDriver mHumidityUserDriver;

	private HumidityUserDriver mHumidityUserDriver;

	public Bme280SensorDriver(I2cDevice device) throws IOException {
		mDevice = new Bme280(device);
	}

	public void registerHumiditySensor() {
		if (mDevice == null) {
			throw new IllegalStateException("cannot register closed driver");
		}

		if (mHumidityUserDriver == null) {
			mHumidityUserDriver = new HumidityUserDriver();
			UserDriverManager.getManager().registerSensor(mHumidityUserDriver.getUserSensor());
		}
	}

	public void unregisterHumiditySensor() {
		if (mHumidityUserDriver != null) {
			UserDriverManager.getManager().unregisterSensor(mHumidityUserDriver.getUserSensor());
			mHumidityUserDriver = null;
		}
	}

	private void maybeSleep() throws IOException {
		if ((mTemperatureUserDriver == null || !mTemperatureUserDriver.isEnabled()) &&
				(mPressureUserDriver == null || !mPressureUserDriver.isEnabled()) &&
				(mHumidityUserDriver == null || !mHumidityUserDriver.isEnabled())
				) {
			mDevice.setMode(Bmx280.MODE_SLEEP);
		} else {
			mDevice.setMode(Bmx280.MODE_NORMAL);
		}
	}

	private class HumidityUserDriver extends UserSensorDriver {
		// DRIVER parameters
		// documented at https://source.android.com/devices/sensors/hal-interface.html#sensor_t
		private static final float DRIVER_MAX_RANGE = 100f;
		private static final float DRIVER_RESOLUTION = 0.00008f;
		private static final float DRIVER_POWER = 280f / 1000.f;
		private static final int DRIVER_VERSION = 1;
		private static final String DRIVER_REQUIRED_PERMISSION = "";

		private boolean mEnabled;
		private UserSensor mUserSensor;

		private UserSensor getUserSensor() {
			if (mUserSensor == null) {
				mUserSensor = UserSensor.builder()
						.setType(Sensor.TYPE_RELATIVE_HUMIDITY)
						.setName(DRIVER_NAME)
						.setVendor(DRIVER_VENDOR)
						.setVersion(DRIVER_VERSION)
						.setMaxRange(DRIVER_MAX_RANGE)
						.setResolution(DRIVER_RESOLUTION)
						.setPower(DRIVER_POWER)
						.setMinDelay(DRIVER_MIN_DELAY_US)
						.setRequiredPermission(DRIVER_REQUIRED_PERMISSION)
						.setMaxDelay(DRIVER_MAX_DELAY_US)
						.setUuid(UUID.randomUUID())
						.setDriver(this)
						.build();
			}
			return mUserSensor;
		}

		@Override
		public UserSensorReading read() throws IOException {
			return new UserSensorReading(new float[]{mDevice.readHumidity()});
		}

		@Override
		public void setEnabled(boolean enabled) throws IOException {
			mEnabled = enabled;
			mDevice.setHumidityOversampling(
					enabled ? Bmx280.OVERSAMPLING_1X : Bmx280.OVERSAMPLING_SKIPPED);
			maybeSleep();
		}

		private boolean isEnabled() {
			return mEnabled;
		}
}
```

## Bringing it all together

We're now finally ready to have our `MainActivity` print out the readings via the our new driver instead of polling the device every two seconds. First, let's comment out the `readSample();` line and add the following to the `onCreate` method:

```java
		mListener = new SensorEventListener() {
			@Override
			public void onSensorChanged(SensorEvent sensorEvent) {
				String type = "";
				if(sensorEvent.sensor.getType() == Sensor.TYPE_AMBIENT_TEMPERATURE) {
					type = "Temp: ";
				} else if(sensorEvent.sensor.getType() == Sensor.TYPE_PRESSURE) {
					type = "Pressure: ";
				} else if(sensorEvent.sensor.getType() == Sensor.TYPE_RELATIVE_HUMIDITY) {
					type = "Humidity: ";
				}
				Log.d(TAG, type + sensorEvent.values[0]);
			}

			@Override
			public void onAccuracyChanged(Sensor sensor, int i) {

			}
		};
```

The listener's `onSensorChanged` method is what gets called when a new value is available. A true app would process the data here (for example it might post it to a back-end service) but for now we're just going to print it on the console. The code that registers the listener should be placed in the `onCreate` method as well:

```java
		mSensorManager.registerDynamicSensorCallback(new SensorManager.DynamicSensorCallback() {
			@Override
			public void onDynamicSensorConnected(Sensor sensor) {
				if (sensor.getType() == Sensor.TYPE_AMBIENT_TEMPERATURE ||
						sensor.getType() == Sensor.TYPE_PRESSURE ||
						sensor.getType() == Sensor.TYPE_RELATIVE_HUMIDITY
						) {
					mSensorManager.registerListener(mListener, sensor,
							SensorManager.SENSOR_DELAY_NORMAL);
				}
			}
		});

		try {
			I2cDevice device = managerService.openI2cDevice(managerService.getI2cBusList().get(0), ADDRESS);
			mSensorDriver = new Bme280SensorDriver(device);
			mSensorDriver.registerTemperatureSensor();
			mSensorDriver.registerPressureSensor();
			mSensorDriver.registerHumiditySensor();
		} catch (IOException e) {
			Log.e(TAG, e.getMessage(), e);
		}
```

Finally, let's clean up in the `onStop` method:

```java
	@Override
	protected void onStop() {
		super.onStop();
		try {
			((SensorManager)getSystemService(Context.SENSOR_SERVICE)).unregisterListener(mListener);
	
			mSensorDriver.unregisterTemperatureSensor();
			mSensorDriver.unregisterPressureSensor();
			mSensorDriver.unregisterHumiditySensor();
			mSensorDriver.close();
		} catch (Exception e) {
			// error closing sensor
		}
	}
```

Running the code should now present you with a lot more furiously changing output than before:

```java
02-22 19:10:47.702 6760-6760/? I/MainActivity: List of available devices: [I2C1]
02-22 19:10:47.706 6760-6760/? D/MainActivity: Device ID byte: 0x60
02-22 19:10:47.711 6760-6760/? I/SensorManager: DYNS Register dynamic sensor callback
02-22 19:10:47.740 6760-6760/? D/Bme280: [75, 356, 0, 333, 0, 30]
02-22 19:10:48.120 6760-6760/? I/SensorManager: DYNS received DYNAMIC_SENSOR_CHANED broadcast
02-22 19:10:48.121 6760-6760/? I/SensorManager: DYNS native SensorManager.getDynamicSensorList return 3 sensors
02-22 19:10:48.122 6760-6760/? I/SensorManager: DYNS dynamic sensor list cached should be updated
02-22 19:10:48.122 6760-6760/? I/SensorManager: DYNS received DYNAMIC_SENSOR_CHANED broadcast
02-22 19:10:48.124 6760-6760/? I/SensorManager: DYNS native SensorManager.getDynamicSensorList return 3 sensors
02-22 19:10:48.124 6760-6760/? I/SensorManager: DYNS received DYNAMIC_SENSOR_CHANED broadcast
02-22 19:10:48.125 6760-6760/? I/SensorManager: DYNS native SensorManager.getDynamicSensorList return 3 sensors
02-22 19:10:48.323 6760-6760/? D/MainActivity: Temp: 23.884022
02-22 19:10:48.324 6760-6760/? D/MainActivity: Pressure: 678.6115
02-22 19:10:48.337 6760-6760/? D/MainActivity: Humidity: 59.578007
02-22 19:10:48.354 6760-6760/? D/MainActivity: Temp: 23.904217
02-22 19:10:48.364 6760-6760/? D/MainActivity: Pressure: 1002.95685
02-22 19:10:48.398 6760-6760/? D/MainActivity: Temp: 23.934504
02-22 19:10:48.406 6760-6760/? D/MainActivity: Pressure: 1002.95276
02-22 19:10:48.415 6760-6760/? D/MainActivity: Humidity: 59.578014
02-22 19:10:48.442 6760-6760/? D/MainActivity: Temp: 23.964792
02-22 19:10:48.451 6760-6760/? D/MainActivity: Pressure: 1002.92114
..............
```

## Conclusion

In this article we've shown how it is possible to write low-level drivers for hardware devices in Android Things. While the implementation details are a bit too low-level for most Android developers, it should feel familiar for the IoT hardware interface programmers. In any case, if the platform does become popular one would expect these drivers to be available for most usual hardware.

The full code for this project is available from this [repo](https://github.com/acristescu/pi-temperature).