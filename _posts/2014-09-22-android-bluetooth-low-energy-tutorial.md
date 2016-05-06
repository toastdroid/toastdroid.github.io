---
layout: post
title: "Android Bluetooth Low Energy Tutorial"
author: Marcelle Gibble
categories: [development]
tags: [bluetooth, btle]
---

In July 2013, the Android API 18 release introduced support for Bluetooth Low Energy (BTLE). With increasing numbers of devices hitting the market, adding support for BTLE devices has become a priority for many.  However, diving into the Android BTLE documentation can be a bit daunting for the novice. This tutorial will walk through the basic concepts of BTLE, and then show snippets of code to further illustrate the communication with remote devices.<!--more-->

## BTLE Concepts

As the name indicates, Bluetooth Low Energy was designed to provide a similar communication range as compared to Bluetooth Classic while using significantly less power. BTLE devices will go into sleep mode and wake only for connection attempts or events. As a result, the software developer needs to understand a few of the basic concepts in BTLE… concepts that we didn’t care about when doing Bluetooth Classic socket programming.

### GATT Profile

All BTLE devices implement one or more profiles. A *profile* is a high level definition that describes how services can be used to enable an application. Low energy application profiles are based on the Generic Attribute Profile (GATT). This is a general specification for sending and receiving short pieces of data (known as attributes) over a low energy link.

### Client

The *client* is the device that initiates GATT commands and accepts responses. For our examples, the Android device will act as the client as this is a typical use case. However, the Android BTLE API does allow the Android device to act as the server.

### Server

The *server* is the device that receives GATT commands or requests and returns responses. For example, heart rate monitors, health thermometers, and location and navigation devices act as servers.

### Characteristic

A *characteristic* is a data value transferred between the client and the server. For example, in addition to the heart rate measurement, a heart rate monitor can also report its current battery voltage, device name, or serial number.

### Service

A *service* is a group of characteristics that operate together to perform a specific function. Many devices implement the Device Information Service. This service is made up of characteristics such as manufacturer name, model number, serial number, and firmware revision.

### Descriptor

A *descriptor* provides additional information about a characteristic. For instance, a temperature value characteristic may have an indication of its units (Celsius or Fahrenheit) or the valid range, the upper and lower values which the sensor can measure. 

Services, characteristics, and descriptors are collectively referred to as attributes and identified by UUIDs (128 bit number). Of those 128 bits, you typically only care about the 16 bits highlighted below. These digits are predefined by the Bluetooth Special Interest Group (SIG).

xxxx**XXXX**-xxxx-xxxx-xxxx-xxxxxxxxxxxx

### GATT Operations

Below are examples of operations that can be performed using these concepts. These are all commands a client can use to discover information about the server.

* Discover UUIDs for all primary services. This operation can be used to determine if a device supports Device Information Service, for example.

* Discover all characteristics for a given service. For example, some heart rate monitors also include a body sensor location characteristic.

* Read and write descriptors for a particular characteristic. One of the most common descriptors used is the Client Characteristic Configuration Descriptor. This allows the client to set the notifications to *indicate* or *notify* for a particular characteristic. If the client sets the Notification Enabled bit, the server sends a value to the client whenever the information becomes available. Similarly, setting the Indications Enabled bit will also enable the server to send notifications when data is available, but the indicate mode also requires a response from the client.

* Read and write to a characteristic. Using the example of the heart rate monitor, the client would be reading the heart rate measurement characteristic. Or, a client might write to a characteristic while upgrading the firmware of the remote device.


## Make it work for Android

Now that we understand the basics of BTLE, let’s highlight the important steps to make this work for your Android app. Recall, it is mandatory that you are working with a device that supports API 18 or higher.

### AndroidManifest.xml

First declare the following permissions in your manifest.  The BLUETOOTH permission allows you to connect with devices and the BLUETOOTH_ADMIN permission allows you to discover devices. 

```xml
<uses-permission android:name=“android.permission.BLUETOOTH” />
<uses-permission android:name=“android.permission.BLUETOOTH_ADMIN” />
<uses-feature android:name=“android.hardware.bluetooth_le”  android:required=“true” />
```

### Get the BluetoothAdapter and Enable

The following code will get the `BluetoothAdapter` class, which is necessary for device discovery. This code also checks that the Android device has Bluetooth enabled and will request that the user enable Bluetooth if it is currently disabled. Notice, this portion of the process is identical to what would be performed if connecting with a Bluetooth Classic device. However, after this step, the similarities cease.

```java
BluetoothManager btManager = (BluetoothManager)getSystemService(Context.BLUETOOTH_SERVICE);
	
BluetoothAdapter btAdapter = btManager.getAdapter();
if (btAdapter != null && !btAdapter.isEnabled()) {
	Intent enableIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);	
	startActivityForResult(enableIntent,REQUEST_ENABLE_BT);
}
```

### Device Discovery

Usually, the first step to connecting with a device is device discovery. This is an asynchronous process. Therefore, we must create the `BluetoothAdapter.LeScanCallback` implementation which will be called each time a device is discovered.

```java
private BluetoothAdapter.LeScanCallback leScanCallback = new BluetoothAdapter.LeScanCallback() {
	@Override
	public void onLeScan(final BluetoothDevice device, final int rssi, final byte[] scanRecord) {
		// your implementation here
	}
}
```

Using the above implementation, we can start and stop device discovery with the following method calls:

```java
btAdapter.startLeScan(leScanCallback);
btAdapter.stopLeScan(leScanCallback);
```

Note: It is wise to stop the scan after you have found the desired devices. As stated in the Android documentation, the discovery process is a heavy drain on the Android device’s battery.

### Create BluetoothGattCallback

Now that you have a `BluetoothDevice` object, the next step is to start the connection process. This requires an instance of the `BluetoothGattCallback` class. There are several useful methods in this class, but the following code will only highlight a few necessary ones. 

```java
private final BluetoothGattCallback btleGattCallback = new BluetoothGattCallback() {

	@Override
	public void onCharacteristicChanged(BluetoothGatt gatt, final BluetoothGattCharacteristic characteristic) {
		// this will get called anytime you perform a read or write characteristic operation
	}

	@Override
	public void onConnectionStateChange(final BluetoothGatt gatt, final int status, final int newState) { 
		// this will get called when a device connects or disconnects
	}

	@Override
	public void onServicesDiscovered(final BluetoothGatt gatt, final int status) { 
		// this will get called after the client initiates a 			BluetoothGatt.discoverServices() call
	}
}
```

And finally, use the following call to initiate a connection:

```java
BluetoothGatt bluetoothGatt = bluetoothDevice.connectGatt(context, false, btleGattCallback);
```


### Discover Services and Characteristics

Assuming the connection attempt is successful, the callback `BluetoothGattCallback.onConnectionStateChange()` will be called with the `newState` argument set to `BluetoothProfile.STATE_CONNECTED`. After this event, discover services can be initiated. As the name implies, the goal of the following call is to determine what services the remote device supports.

```java
bluetoothGatt.discoverServices();
```

When the device responds, you will received the callback `BluetoothGattCallback.onServicesDiscovered()`. You must receive this callback before you can get a list of the supported services. And once you have a list of services a device supports, use the following code to get the characteristics for that service. 

```java
List<BluetoothGattService> services = bluetoothGatt.getServices();
for (BluetoothGattService service : services) {
	List<BluetoothGattCharacteristic> characteristics = service.getCharacteristics();
}
```

### Configure Descriptor for Notify

Thus far, we have connected to the device, discovered its supported services, and retrieved a list of the characteristics for each service.  Now that we have the basic information, let's do something useful. Using a heart rate monitor as an example, we want the monitor to send us regular readings of the user’s heart rate. Recall we talked earlier about the Client Characteristic Configuration Descriptor which will enable notifications on the remote device.  The following code illustrates how we would get the descriptor, and then enable the notification flag.

```java
for (BluetoothGattDescriptor descriptor : characteristic.getDescriptors()) {
	//find descriptor UUID that matches Client Characteristic Configuration (0x2902)
	// and then call setValue on that descriptor

	descriptor.setValue( BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE);
	bluetoothGatt.writeDescriptor(descriptor);
}
```

Notice the reference to a UUID value of 0x2902. We discussed earlier the 16 bits of the UUID that we typically care about. Each service, characteristic and descriptor has a UUID associated with it. See the [Bluetooth SIG Assigned Numbers](https://www.bluetooth.org/en-us/specification/assigned-numbers) website for a full list of the predefined UUIDs.


### Receive Notifications

When the heart rate monitor sends the heart rate measurement to the Android device, you will receive a callback in `BluetoothGattCallback.onCharacteristicChanged()`. The following code shows how to retrieve that information.

```java
@Override
public void onCharacteristicChanged(BluetoothGatt gatt, final BluetoothGattCharacteristic characteristic) {
	//read the characteristic data
	byte[] data = characteristic.getValue();
}
```

### Final step

And finally, to disconnect and close the GATT client, make the following calls:

```java
bluetoothGatt.disconnect();
bluetoothGatt.close();
```


## Hints and Observations

Hopefully this tutorial will help you develop your own BTLE applications. If that is your goal, there are a couple more things you should know. (Items NOT found in the documentation).

* Stop device discovery before attempting a connection. It has already been stated device discovery should stop to prevent battery drain. However stopping device discovery **before** attempting a connection has given me better connection results.

* Queue all GATT operations (connection attempts, device discovery, reads and writes to characteristics), and execute them one at a time. Evidence suggests the lower layer GATT operations are not being queued and some will get dropped if you don’t handle this yourself.

* Make every effort to disconnect the device and close the GATT client when your app closes (whether that is due to user action, an unexpected crash, or the OS removing your app as a result of system resource shortages). If the session is left open, the user will likely have difficulty reconnecting  until the previous connection times out, or Bluetooth is cycled on the Android device, or the device is rebooted.

* Android examples typically have the developer walk through the process of device discovery to get a BluetoothDevice object. However, in a typical use case, a user would want to connect to the same remote device each time (i.e. just connect to the heart rate monitor that you own). 
	After the initial device discovery, save the device's MAC address. See `BluetoothDevice.getAddress()`. Then, next time the user wants to connect, use that address to construct the BluetoothDevice object. See `BluetoothAdapter.getRemoteDevice()`. This will bypass device discovery which can create a smoother user experience. 

Below are some links that you might find useful. Enjoy!

• [Android BTLE API documentation](http://developer.android.com/guide/topics/connectivity/bluetooth-le.html)
• [Bluetooth SIG specification](https://developer.bluetooth.org/gatt/Pages/default.aspx)
• [Google I/O 2013 Best Practices for Bluetooth Development](http://www.youtube.com/watch?v=EC5-cEbr520)
