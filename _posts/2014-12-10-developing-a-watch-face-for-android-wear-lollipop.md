---
layout: post
title: "Developing a Watch Face for Android Wear Lollipop"
author: James Barr
categories: [development, tools]
tags: [android, wear, watch face, lollipop]
---

Today, Google announced [Lollipop for Android Wear](http://android-developers.blogspot.com/) devices. One of the most anticipated features included in this update is official support for developing watch faces. This is such an exciting feature that many developers, Two Toasters included, have already released unofficial watch faces (such as [Chron](https://play.google.com/store/apps/details?id=com.twotoasters.chron)) using [unofficial methods](/2014/07/18/developing_watchfaces_for_android_wear). Two Toasters was lucky enough to get early access to the watch face APIs and had the opportunity to partner with Specialized, a brand well known to cyclists everywhere, to develop the [Specialized Bikes Watch Face](https://play.google.com/store/apps/details?id=com.specialized.watchface) that is uniquely suited to cyclists. Now that official support exists for watch faces, let me show you how to implement one the proper way so you can gain all of the benefits and features from using the new APIs.<!--more-->

### Wear Watch Face API Overview

An Android Wear watch face is a Wear app installed on the watch which implements a service that is an extension of an Android live wallpaper. The `WatchFaceService` class provides additional lifecycle methods used by wearables and helps accommodate the additional design considerations that must be made. For debugging purposes, the Wear app can be installed by sending the Wear APK, signed with the debug key, directly to the wearable via ADB. For release, a release-signed Wear APK can be built and packaged by the Android build system inside of a release-signed mobile APK. When the release-signed mobile app is installed on a mobile device by the Play Store, the mobile device will send the Wear app to the connected wearable. One note here is that the mobile and Wear apps must be signed with the same key. Also, this method must be used for distribution through the Play Store.

Although no activities are required for implementing a watch face, many users will want to customize their watch face. The watch face APIs provide developers two ways that they can allow their users to modify watch face settings. Users will be able to change some settings on the wearable by tapping on a settings icon below the watch face preview in the watch face picker when implemented by the developer. In addition, users will be able to customize the full set of settings by tapping the settings icon overlaid on top of the watch face preview in the Android Wear companion app when implemented by the developer. Each of these two settings screens are activities that can be declared in the respective Android manifests.

### Project Setup and Manifest Config

The first step for creating a watch face is setting up a new project with two modules; we'll call them `mobile` and `wear`. Both of these modules will be Android apps, so we will apply the `com.android.application` plugin to both modules in their `build.gradle` files. It is important that both modules have the same package name, so it is best to externalize the package name to a [gradle.properties file](http://www.gradle.org/docs/current/userguide/tutorial_this_and_that.html#sec:gradle_properties_and_system_properties) and have the `build.gradle` for both modules and to read the value from there. If your watch face will have settings screens, then it will be helpful to also create a shared `common` module that consists of an Android library project that both the mobile and wear modules depend upon.

##### Wear Module

For the wear module, we want to set the minSdk to 21 since we require the watch face APIs in Lollipop. In the dependencies section of our `build.gradle`, we'll want to add `compile "com.google.android.support:wearable:1.+"` so that we have access to the wearable support library. We'll need to require the `com.google.android.permission.PROVIDE_BACKGROUND` and `android.permission.WAKE_LOCK` permissions in our wear manifest file. We'll also add the `<uses-feature android:name="android.hardware.type.watch" />` line so that the Wear app can only be installed on wearables. The last required configuration item we need to add is the service that will display the watch face. The items to notice in the service declaration below are the `allowEmbedded`, `taskAffinity`, and `permission` attributes and the `intent-filter` declaration that are required for services providing watch faces.

```xml
<service
    android:name=".service.ChronWatchFaceService"
    android:allowEmbedded="true"
    android:label="Chron Watch Face"
    android:taskAffinity=""
    android:permission="android.permission.BIND_WALLPAPER">
    <meta-data
        android:name="android.service.wallpaper"
        android:resource="@xml/watch_face" />
    <meta-data
        android:name="com.google.android.wearable.watchface.preview"
        android:resource="@drawable/preview" />
    <meta-data
        android:name="com.google.android.wearable.watchface.preview_circular"
        android:resource="@drawable/preview_circle" />
    <meta-data
        android:name="com.google.android.wearable.watchface.wearableConfigurationAction"
        android:value="com.twotoasters.chron.CONFIG_CHRON" />
    <meta-data
        android:name="com.google.android.wearable.watchface.companionConfigurationAction"
        android:value="com.twotoasters.chron.CONFIG_CHRON" />
    <intent-filter>
        <action android:name="android.service.wallpaper.WallpaperService" />
        <category android:name="com.google.android.wearable.watchface.category.WATCH_FACE" />
    </intent-filter>
</service>
```

As for the `WatchFaceService`'s meta-data:
- android.service.wallpaper (required) - Reference to resource in res/xml with `<wallpaper xmlns:android="http://schemas.android.com/apk/res/android" />`
- preview (required) - Reference to a 320x320px preview image for the watch face that will be displayed in the watch face picker
- preview_circular (optional) - Reference to a circular 320x320px preview image with transparent background for the watch face that will be displayed in the watch face picker for round devices. The square preview image will be used and cropped when this is not provided.
- wearableConfigurationAction (optional) - String used to identify the wearable settings activity. This value will be included as the intent action for that activity.
- companionConfigurationAction (optional) - String used to identify the mobile settings activity. This value will be included as the intent action for that activity.

The wearable settings activity would be declared similar to the entry below. Note that the action uses the value from the service's `wearableConfigurationAction` meta-data entry. We must also include both of the category listed in the `intent-filter`.

```xml
<activity
    android:name=".activity.ChronWearConfigActivity"
    android:label="@string/config_label">
    <intent-filter>
        <action android:name="com.twotoasters.chron.CONFIG_CHRON" />
        <category android:name="com.google.android.wearable.watchface.category.WEARABLE_CONFIGURATION" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

##### Mobile Module

As for the mobile module, we will need to tell it where the Wear app is by adding `wearApp project(':wear')` to the `dependencies` section of the mobile module’s `build.gradle` file. We'll also want to make sure that we set the minSdk to 18 since only mobile devices running Android 4.3 or higher can connect to Android Wear devices. The mobile app does not require any activities, services, or receivers to be present. Since the mobile app doesn't require an activity, no unnecessary app icon will show up in the user's app launcher on their phone, but the watch face will still be installed on the user’s watch. We do need to be sure that we specify any permissions required by the Wear app in the mobile app's manifest.

Optionally, you can declare a mobile settings screen in the manifest by adding an activity like the entry below. Both categories must be included in the intent filter and the action must use the value from the service's `companionConfigurationAction` meta-data entry.

```xml
<activity
    android:name=".activity.ChronCompanionConfigActivity"
    android:label="Chron Settings">
    <intent-filter>
        <action android:name="com.twotoasters.chron.CONFIG_CHRON" />
        <category android:name="com.google.android.wearable.watchface.category.COMPANION_CONFIGURATION" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

### Watch Face UI Modes

There are many factors that must be taken into account when developing a watchface.  First and foremost, watch faces need to accommodate square, circular, and almost-circular (like the Moto 360) faces. On top of that, faces need to accommodate peeking notification cards, hotword hints, and status bar icons. Watch faces also have many different UI modes that need to be accommodated in order to be battery efficient and protect the device's display:
- Interactive - Wearable is awake and active, backlight is fully lit, and screen can be drawn anywhere from once every 16ms to once every minute
- Ambient - Wearable is asleep, the watch face will only update every minute, and the backlight is dimmed
- Low-Bit Ambient - A variation of ambient mode where only fully white pixels will be drawn to the screen
- Burn-In Protection - Applicable to certain wearables only in ambient mode. Watch faces should not draw large patches of solid colors

For the designers out there, see what our Director of Creative Services at Two Toasters had to say about [designing watch faces](http://twotoasters.com/ideas/category/ideas/design/).

### Implementing the Watch Face

Now that the project is setup and your design accommodates all of the various states and system UI objects, all that is left to do is to implement the service that will display the watch face. Your first decision will be to choose which `WatchFaceService` subclass to extend. If you want to draw using the Java canvas, then extend `CanvasWatchFaceService`; otherwise, if you want to use OpenGL ES 2.0, then extend `GlesWatchFaceService`. We'll focus on the former right now. The only method your service must implement is `onCreateEngine()`. This should return an instance of the `WatchFaceService.Engine` used to display your watch face.

The `Engine` class contains most of the logic for your watch face:
- onApplyWindowInsets(WindowInsets insets) - Check the wearable shape with `insets.isRound()` and check for a flat tire with `insets.getStableInsetBottom() > 0`
- onPropertiesChanged(Bundle bundle) - Check if wearable uses low-bit ambient mode or burn-in protection with the PROPERTY_LOW_BIT_AMBIENT and PROPERTY_BURN_IN_PROTECTION bundle keys
- onAmbientModeChanged(boolean inAmbientMode) - Track and adjust the UI for when the wearable goes into or comes out of ambient mode
- onTimeTick() - Re-draw the display because the minute has changed. This is how the wearable redraws itself in ambient mode.
- onDraw(Canvas canvas, Rect bounds) - Draw your watch face on the canvas based on the current state of ambient mode and property values
- onVisibilityChanged(boolean visible) - This is where you should connect/disconnect to `GoogleApiClient`'s, register/unregister receivers and observers, and add/remove listeners

The engine also includes the standard `onCreate(SurfaceHolder holder)` and `onDestroy()` methods. You should create your `GoogleApiClient` object, if used, and set your watch face's style in `onCreate(...)`. You can create a `WatchFaceStyle` by using the builder. This will let you configure settings like:
- notification card peek height
- hotword indicator positioning
- status bar indicator positioning
- whether or not the hotword and status bar icons should have a transparent gray background drawn behind them
- whether or not the system ui time should be shown (most likely not, since you're developing a custom watch face)

### Handling Time Format and Time Zone Changes

In order to handle these changes, we'll register and unregister a receiver and an observer in `Engine.onVisibilityChanged(boolean visible)`. We get the current time from a `Time` object that we create during creation of our `Engine`. In our `Engine.onDraw()` method, we can call `Time.setToNow()` so that our `Time` object has the latest time. We can also listen for time zone updates by enabling a broadcast receiver. When the receiver is triggered, we'll call `Time.clear(intent.getStringExtra("time-zone"))` and then `Time.setToNow()`.

```java
private void registerTimeZoneReceiver() {
    if (mRegisteredTimeZoneReceiver) {
        return;
    }
    mRegisteredTimeZoneReceiver = true;
    IntentFilter filter = new IntentFilter(Intent.ACTION_TIMEZONE_CHANGED);
    getApplicationContext().registerReceiver(mTimeZoneReceiver, filter);
}
```

In addition, we can listen for time format changes (12-hour vs 24-hour) by setting up a content observer. When this observer is triggered, we'll check what the current time format is via `DateFormat.is24HourFormat(Context context)`.

```java
private void registerTimeFormatObserver() {
    if (mRegisteredFormatChangeObserver) {
        return;
    }
    mRegisteredFormatChangeObserver = true;
    final ContentResolver resolver = getApplicationContext().getContentResolver();
    resolver.registerContentObserver(Settings.System.CONTENT_URI, true, mFormatChangeObserver);
}
```

Now that the official watch faces API has arrived, we hope that this guide helps you get up to speed quickly so you can begin design and development of more awesome Android Wear watch faces.
