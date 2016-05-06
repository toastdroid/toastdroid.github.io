---
layout: post
title: "Upgrading QA for Android"
author: Karl Smith
categories: [qa]
---

If you are a QA veteran, you have likely formed a set of skills and routines that allow you to tackle each project efficiently.  For the most part, you can re-institute your toolset if you move to a different QA position or your company begins developing for new platforms.  

Coming from a web QA background, I spent most days dealing with the 4 major web browsers and a handful of operating systems.  I had my routine of going through each, performing Black Box, Regression, Ad-Hoc, and Smoke testing.  Eventually, the company I was with decided to take their product mobile and I was introduced to iOS for the first time.  With little adjustment, I was able to translate my web skills to iOS testing.<!--more-->  Similar to web related testing, Apple has a handful of devices, a regularly updated OS, and just a few browsing clients.

When I started work at Two Toasters, I got my first taste of Android.  Like before, I expected my skills to easily translate over to the new platform without need of assistance or training.  I wasn't completely wrong, but there are definitely a few Android specific quirks that, had I known ahead of time, would have made for a much smoother transition.

## Devices, Devices, and more Devices

Android, unlike iOS, has a plethora of devices.  These devices have a wide range of size, style, and OS integration.  Becase of this, manual testing becomes a more extensive process than on other platforms.  

First, let's look at the OS range.  Android devices can be seen with an operating system as old as Gingerbread (2.3) all the way up to the most recent, Kit-Kat (4.4)  While Gingerbread support is fading, as you may have read in a previous blog “[Death of GingerBread](/2014/03/04/death_of_gb)”, many companies will still want their app to support it.  Since Gingerbread is so outdated (Released in Dec, 2010), it can have issues with new apps working correctly. The best course of action is to urge your your company or client to support the most current OS versions and leave GingerBread behind.  Though, if you find yourself with a GingerBread device, be on the lookout for hard crashes when launching new screens/modals, and social media integration.  

## There's How Many Screen Resolutions?!
While OS variations will cause you functional issues, the variety of Android device resolutions may cause your app visual woes. Resolutions are as follows:

Bucket | Relative Definition 
--- | --- |
ldpi | (low definition)
mdpi | (medium definition)
hdpi | (high definition)
xhdpi | (normally 720p)
xxhdpi | (normally 1080p)
xxxhdpi | (much high definition wow) ![doge](http://fc09.deviantart.net/fs71/f/2013/244/f/7/meme_doge_icon_by_euamodeus-d6kngqa.png)

Now, don't be under the assumption that newer = higher definition.  There are devices being released in 2014 with lower resolution than devices from last year.  Things to watch out for are assets stretching/disappearing, text formatting/spacing being inconsistent, and other layout related anomalies. 

## Manufacturer Specific Software

Another aspect of Android devices that can catch you off-guard is the first time your app decides to reject the software which is preloaded onto a device by the manufacturer.  Most commonly, manufacturers like to install their own front-end touch interface, commonly known as a skin.  Some of the more notorious ones are TouchWiz (Samsung), Sense (HTC), and MotoBlur (Motorola).  TouchWiz is always the first to come to mind out of this list as a hurdle to be aware of.  Related issues can arise anywhere in your app that deals with the user interacting with the app via touch; so…pretty much everywhere.  Typically, the mild cases result in a delay in response to touch input or no response at all.  Severe cases will result in the app crashing.  Thankfully, you will usually know TouchWiz is to blame as it too will crash shortly after your app.

## Why Would I Ever Want to QA Anything on Android

Trust me, I know it's a lot to take in at first.  Android has more devices, more quirks, more to watch out for.  But, that also means that the platform has more opportunity to be creative, innovative, and refreshing to work with.  On top of that, the Android community are tight knit and open to questions and feedback.  If, in your pioneer run on Android devices you come across something foreign, you are only a Google search away from a multitude of forums, faq's, and helpful sites.

## Suggested Test Devices

The following devices are what I would suggest to minimally cover all the aforementioned angles and give your QA team really good coverage.

* HTC Desire with Android 2.2
* LG P990 with Android 2.2
* Samsung Galaxy S with Android 2.3
* Sony Ericsson Arc with Android 2.3
* Motorola Razr Droid with Android 2.3.5
* Samsung Galaxy Note with Android 2.3.6
* HTC Sensation with Android 4.0.3
* Samsung Google Nexus with Android 4.0.4
* Samsung Galaxy S2 with Android 4.0.4
* Galaxy Tab 10 with Android 4.0.4 (or higher)
* Asus Nexus 7 with Android 4.3
* LG Nexus 5 with Android 4.4
* HTC One with Android 4.4
