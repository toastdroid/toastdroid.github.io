---
layout: post
title: "2013: The Death of Gingerbread"
author: Chris Pierick
categories: [opinion]
tags: [GB, ICS]
---

Gingerbread is dead. That's right, not dying but dead. 

This happened in 2013 but most developers have not realized it yet. Unless you're coding internal apps for your company and have an unfortunate requirement, there is no reason to code for an Android version before API 15 (IceCreamSandwich). Even that is  unfortunate as API 15 was released on December 16, 2011(1), well over two years ago.

## Gingerbread and IceCreamSandwich are losing marketshare

Right now, if you set your app's minimum API level to 16, you are missing out on 35% of the market. That is changing rapidly as both Gingerbread and IceCreamSandwich are on a downward slope(Fig 1).<!--more-->

Gingerbread accounts for 20% of Android devices. At current loss of market, an average of -2.17% per month, in just over 4 months Gingerbread will only have 10% of the market. On top of that, IceCreamSandwich's current loss of market is -1.28% per month. If this holds true, then in four months API level 15 and lower will contain less then 20% of the total android market. This leaves about 80% of the market at API level 16 or higher.

If you start building an app now by the time it's built Android will have lost most of what remains of that piece of the market. For most people, supporting Gingerbread should not be a discussion topic. The real discussion now is whether to set your minimum API level to 15 or to 16.

![Fig 1.](/assets/2014-03-04-death_of_gb/DeathOfGB-Fig1.png)

## Can you make money from Gingerbread?

At the end of the day, what matters is whether you can make money from your Gingerbread users. While I cannot publicly release the data to back up this argument, I can tell you the answer is no. Gingerbread users will not give you a significant amount of in app revenue. The main reasons for this are disposable income and app quality. 

Users with better devices have more disposable income. People with 3+ year old phones or budget phones aren't going to spend money in an app. Why would they when they have already shown that they are not going to spend money on a "good" device or upgrading their current one.

I don't know if you've been on Gingerbread lately and tried to install apps, but don't waste your time. Most apps simply crash at startup or within five minutes, due to low memory errors. I personally wouldn't try to spend money using one of these devices. Gingerbread users don't have confidence in apps, and simply are not interested in purchasing things in apps.

If Gingerbread users don’t spend money, IceCreamSandwich users are only marginally better. To even make a reasonable amount of money from these users you have to be doing millions of dollars in revenue in your app. In a couple months even that won't be true.

## Manufactures are finally updating devices

If we continue on the idea that people who upgrade their phones are the ones who spend money in apps, then let's look back at some of the best selling android phones of 2012. All but one of these devices in fig. 2 have been updated to some form of Jelly Bean, with some of them being promised KitKat.

Manufacturer | Model | Launch Version | Current Version
--- | --- | --- | --- |
HTC | Droid DNA | 4.1.1	 | 4.2.2
HTC	| One X | 4.0.3	| 4.2.2
HTC | One S | 4.0.3	 | 4.0.3
Motorola | Droid RAZR MAXX | 2.3.6 | 4.1.2*
Motorola | Droid RAZR HD | 4.0.3 | 4.1.2*
Motorola | Droid RAZR M | 4.0.3 | 4.1*
Samsung | GS3 | 4.0.4 | 4.3
Samsung | Galaxy Note 2 | 4.1.2 | 4.3
LG | Nexus 4 | 4.2 | 4.4.2
LG | Optimus G | 4.0.3 | 4.1
			
*Motorola had announced an update to 4.4		
Fig 2 references Wikipedia

On top of that, 2013 is looking even better with the Moto G, Moto X, Samsung S4, Samsung Note 3, LG G2 and HTC One all updated to KitKat API level 19, the latest version of Android. Some of these devices even received their KitKat updates before Nexus devices did. Manufactures such as HTC are now giving a 2 year upgrade guarantee for all their new devices.

The question then comes to low end devices and emerging markets. Google is attempting to kill Gingerbread there by reducing the memory requirement of KitKat to 512M(4). By reducing this memory requirement there is little incentive for manufacturers to use Gingerbread on budget devices when they could produce these devices with a more competitive OS. 

There is also a leaked document which states that the Play Store will no longer be approved for new devices running Gingerbread(3). Google is making a statement here, which is that Gingerbread is now dead.

## Conclusions

As Gingerbread takes its last dying breaths, we in the Android community can celebrate that our old friend is no longer suffering (or making us suffer). By dropping Gingerbread and going to API 15 or 16 you get a lot of new and great APIs. These new APIs are not just limited to the UI and animations. They focus around notifications, networking, NFC, bluetooth, media, camera, renderscript, non-backward compatible animations and more (5, 6).

Sure there are libraries like NineOldAndroids, ActionBarSherlock or HoloEverywhere to help support Gingerbread, but they are a pain to use and still cannot cover all functionality. The cost to support these older APIs will remain the same, but the revenue potential no longer exists.

## References

1. [http://en.wikipedia.org/wiki/Android_version_history#Android_4.0.3.E2.80.934.0.4_Ice_Cream_Sandwich_.28API_level_15.29](http://en.wikipedia.org/wiki/Android_version_history#Android_4.0.3.E2.80.934.0.4_Ice_Cream_Sandwich_.28API_level_15.29)
2. [http://en.wikipedia.org/wiki/Android_version_history](http://en.wikipedia.org/wiki/Android_version_history)
3. [http://www.androidpolice.com/2014/02/10/rumor-google-to-begin-forcing-oems-to-certify-android-devices-with-a-recent-os-version-if-they-want-google-apps/](http://www.androidpolice.com/2014/02/10/rumor-google-to-begin-forcing-oems-to-certify-android-devices-with-a-recent-os-version-if-they-want-google-apps/)
4. [https://developer.android.com/about/versions/kitkat.html](https://developer.android.com/about/versions/kitkat.html)
5. [https://developer.android.com/about/versions/android-4.0-highlights.html](https://developer.android.com/about/versions/android-4.0-highlights.html)
6. [https://developer.android.com/about/versions/android-4.1.html](https://developer.android.com/about/versions/android-4.1.html)


