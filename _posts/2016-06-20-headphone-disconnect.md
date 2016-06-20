---
layout: post
title: "Byte of Toast: How to Detect Wired and Bluetooth Headphone Disconnects"
author: Chris Pierick
categories: [byte of toast]
tags: [headphones, bluetooth, audio, music]
---

When building an app that plays music, it is important to be able to pause the music when the headphones are disconnected. You may start this process by trying to listen to Intents such as [Intent.ACTION_HEADSET_PLUG](http://developer.android.com/reference/android/content/Intent.html#ACTION_HEADSET_PLUG) and [BluetoothA2dp.ACTION_CONNECTION_STATE_CHANGED](http://developer.android.com/reference/android/bluetooth/BluetoothA2dp.html#ACTION_CONNECTION_STATE_CHANGED). Marshmallow even has a really nice [AudioDeviceCallback](https://developer.android.com/reference/android/media/AudioDeviceCallback.html) which may be helpful once adoption of Marshmallow has gone up. Since this all seems overly complicated, you may just shrug and say, "typical Android".<!--more-->

Thankfully, handling this scenario is simple thanks to a poorly named Intent called [AudioManager.ACTION_AUDIO_BECOMING_NOISY](https://developer.android.com/reference/android/media/AudioManager.html#ACTION_AUDIO_BECOMING_NOISY). The system sends this Intent when the device's external audio has been disconnected, and audio is about to be routed to the device's internal speakers.

```java
public class HeadphoneReceiver extends BroadcastReceiver {

    public HeadphoneReceiver(Context context) {
        IntentFilter intentFilter = new IntentFilter(AudioManager.ACTION_AUDIO_BECOMING_NOISY);
        context.registerReceiver(this, intentFilter);
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        if (AudioManager.ACTION_AUDIO_BECOMING_NOISY.equals(intent.getAction())) {
            // pause music
        }
    }
}
```
In the `onReceive` of your `BroadcastReceiver` you can handle the intent action for [AudioManager.ACTION_AUDIO_BECOMING_NOISY](https://developer.android.com/reference/android/media/AudioManager.html#ACTION_AUDIO_BECOMING_NOISY) and pause your music. And remember, registering a receiver like this statically in your manifest is now [discouraged by the Android team](https://www.youtube.com/watch?v=VC2Hlb22mZM&list=PLOU2XLYxmsILe6_eGvDN3GyiodoV3qNSC&index=27).
