---
layout: post
title: "Developing Watchfaces for Android Wear"
author: James Barr
categories: [development, tools]
tags: [android, wear, watchface]
---

*DEPRECATED: The official Android Wear Watch Face API has been released. You can find more details in [our updated post](/2014/12/10/developing-a-watch-face-for-android-wear-lollipop).*

This past week, Two Toasters released their first Android Wear watchface called [Chron](https://play.google.com/store/apps/details?id=com.twotoasters.chron). Having worked on Chron and seen the lack of documentation about developing watchfaces, I wanted to share what we have learned and what we have done in order to make future watchfaces easier to develop.<!--more-->

### Wear Watchface Overview

An Android Wear watchface is a Wear app running on the watch with an activity that is displayed to the user.  The Wear app can be installed by one of two ways. During development, you can send the unsigned Wear app directly to the watch via ADB. For Play Store distribution, the Wear app is packaged by the Android build system inside of a mobile app that is installed on the phone. If your mobile app and Wear app are both signed with release keys, then you can also send the mobile app (containing the Wear app) to the phone via ADB and the Wear app will be installed on the connected watch.

### Project Setup and Manifest Config

The first step for creating a watchface is setting up a new project with two modules; we'll call them `mobile` and `wear`. Both of these modules will be Android apps, so we will apply the `com.android.application` plugin to both modules in their `build.gradle` files. It is important that both modules have the same package name, so it is best to externalize the package name to a gradle.properties file and have the `build.gradle` for both modules to read the value from there.

For the mobile module, we will need to tell it which is the Wear app by adding `wearApp project(':wear')` to the `dependencies` section of the mobile module’s `build.gradle` file. We'll also want to make sure that we set the minSdk to 18 since only phones running Android 4.3 can connect to Android Wear devices. The mobile app does not require any activities, services, or receivers to be present. Since the mobile app doesn't require an activity, no unnecessary app icon will show up in the user's app launcher on their phone, but the watchface is still installed on the user’s watch. We do need to be sure that we specify any permissions that the Wear app needs in the mobile app's manifest. We'll also add a basic application tag. Our watchface only requires one permission: `com.google.android.permission.PROVIDE_BACKGROUND`.

For the Wear module, we want to set the minSdk to 20. As mentioned before, we'll need to require the `com.google.android.permission.PROVIDE_BACKGROUND` permission in our Wear manifest file, too. We'll also add the `<uses-feature android:name="android.hardware.type.watch" />` line so that the Wear app can only be installed on watches. The last configuration item we need to add is the activity that will display the watchface. The things to notice in the activity description below are the `allowEmbedded="true"` attribute declaration, the preview metadata tag, and the home_background intent filter.

```xml
<activity
    android:theme="@android:style/Theme.DeviceDefault.NoActionBar"
    android:name=".activity.WatchfaceActivity"
    android:label="${watchfaceName}"
    android:taskAffinity=""
    android:allowEmbedded="true" >

    <meta-data android:name="com.google.android.clockwork.home.preview" android:resource="@drawable/preview" />

    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="com.google.android.clockwork.home.category.HOME_BACKGROUND" />
    </intent-filter>
</activity>
```

The metadata tag provides a drawable for a 320x320px preview image for the watchface that will be displayed in the watchface picker. The intent filter indicates to the Android Wear system that this activity can be displayed as a watchface background. Note that it is possible to bundle many different watchfaces within an app by including multiple activities with these metadata tags and intent filters.

### Watchface States - Normal vs Dimmed

Now that the project is setup, all that is left is to implement the activity that will be displayed. Since we treat activities as controllers, the main UI of the watchface will actually be implemented in a WatchfaceWidget that extends `View` or `ViewGroup`. We do use the activity to notify the WatchfaceWidget of the different states. A watchface can be in its normal display state or a dimmed state. In the dimmed state, watches with AMOLED screens will only draw pure white pixels to the screen and all watches will lower their display brightness. This dimmed state helps increase battery life and reduce the risk of screen burn in. In the dimmed state, it is common to remove any background images or switch to a darker black image. This state is simply managed via the onResume/onPause activity lifecycle methods. In our case, we when the activity gets its onResume/onPause callbacks, we notify the WatchfaceWidget that it should change its state and redraw itself.

### Implementing the Watchface Widget

In order to implement the watchface we need to get the current time, listen for time updates, listen for time setting changes, schedule our next update, and draw the watchface. We get the current time from a calendar object that we create during creation of our widget. We can listen for time updates by enabling a broadcast receiver. This receiver cannot be configured in the app's manifest. When the receiver is triggered, we'll call our watchface's onTimeChanged() method (more on this later).

```java
private void registerReceiver() {
	final IntentFilter filter = new IntentFilter();
	filter.addAction(Intent.ACTION_TIME_TICK);
	filter.addAction(Intent.ACTION_TIME_CHANGED);
	filter.addAction(Intent.ACTION_TIMEZONE_CHANGED);
	getContext().registerReceiver(mIntentReceiver, filter, null, getHandler());
}
```

We can listen for time setting changes through a combination of the previous receiver and setting up a content observer. When this observer is triggered, we'll call our watchface's chooseFormat() and onTimeChanged() methods (more on these later).

```java
private void registerObserver() {
    final Context context = getWatchface().getContext();
    final ContentResolver resolver = context.getContentResolver();
	resolver.registerContentObserver(Settings.System.CONTENT_URI, true, mFormatChangeObserver);
}
```

We'll register the receiver and observer in our `View`'s `onAttachedToWindow()` and unregister in `onDetachedFromWindow()`. Also in these methods, we'll manage the delayed scheduling and unscheduling of a `Runnable` using a handler that will call our onTimeChanged() method and then reschedule itself.

Now, in our widget's onTimeChanged() method, we'll get the current time, rotate any image layers we use for the watch hands, draw anything we need to, and invalidate the watchface root view to trigger a painting of the widget. One thing to note is that when we draw image assets or paint with colors, we need to honor the `isScreenActive` flag that is managed by the activity when selecting resources or colors.

### Using Libraries for Easier Setup

During the development of Chron, we noticed that some pieces would likely be reusable by future watchfaces. We pulled these pieces out into a library called [watchface-gears](https://github.com/twotoasters/watchface-gears). This contains a `Watch` object that makes it much easier to manage the registration of your time receiver, observer, and updating runnable. It also contains a simple base watchface activity, a watchface interface for your custom view to implement, as well as common watchface-related utilities.

In addition, we've created [watchface-template](https://github.com/twotoasters/watchface-template), which is a sample project with instructions for use. The basic idea is to clone this project, run a project generation script, reinit the Git repo for your purpose, and you'll have a basic watchface that uses watchface-gears all ready to go. All you'll have to do is modify the Watchface widget, layout, and drawing.

While watchfaces are still only unofficial at this point, we hope that this guide helps you begin design and development of more awesome Android Wear watchfaces to hold you over until the official API arrives.

