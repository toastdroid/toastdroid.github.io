---
layout: post
title: "Charles Proxy with Android"
author: Patrick Jackson
categories: [tools]
tags: [Charles proxy, Nougat]
---

Viewing the network traffic of your app is a great debugging tool.  Sure, we can add logs or use [Stetho](http://facebook.github.io/stetho/) (which I recommend, it's great!), but sometimes you need to see what is ___actually___ going out over the wire. This will walk you through getting Charles Proxy  [Charles Proxy](https://www.charlesproxy.com/) setup with Android so you can see all your requests, including SSL.  If you've used Charles in the past you may want to pay attention because there are some new changes with Nougat.

## 5 Easy Steps


### 1) Install Charles 4.x
Limited free version available at [https://www.charlesproxy.com/](https://www.charlesproxy.com/)

### 2) Install Charles cert on Android device 
 	
First you will need to set the proxy for your wifi connection.  For this to work your device and your proxy computer must be on the same wifi network, and Charles must be running.  

On the Android device go to `Settings --> Wifi`.  You will then see a list of wifi networks.  Long press on the one you are using and then select "Manage network settings".  You can then check "Show advanced options" and this will show the proxy settings.  You must enter your ip address and port.  This can be found by going to the `Help --> SSL Proxy --> Install Charles Root Certificate to a mobile device or browser`.  

If you're on an emulator, you can set the proxy by using -http-proxy from the command line.

<p align="center">
<img src="https://storage.googleapis.com/tmapp/proxy_settings.png" style="max-height: 600px; max-width: 600px;"  />
</p>

### 3) Install Charles cert on Android device 

Now that the proxy is set we need the Charles root certificate installed so Charles can decrypt your SSL traffic.  To do this navigate to [https://chls.pro/ssl](https://chls.pro/ssl) on your device and you will be prompted to download the cert.  If this is your first time using Charles with your device, you will get a dialog asking if you want to allow incoming traffic.  Click yes.  Once downloaded you can open it and you will be asked to name the cert.  You will also need to add security to your lock screen, if not enabled already.  You should now see the cert in your security settings under "View security certificates".
   
<p align="center">
<img src="https://storage.googleapis.com/tmapp/user_certs.png" style="max-height: 600px; max-width: 600px;"  />
</p>

### 4) Add a Network Security Configuration file

Starting in Android 7.0 (API level 24), apps do not trust user installed certs by default.  This is a security measure. So if your app is targeting API level 24 or above, you will need to add a Network Security Configuration file to you app.  A more detailed explanation is [here.](https://developer.android.com/training/articles/security-config.html)   You probably don't want your app to trust all user installed certs, so instead we can opt in only in debug builds.    Below is a config that will trust the Charles cert only in debug builds.  You will also need to point the app to the config file in your AndroidManifest.xml.
 
network_security_config.xml:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <debug-overrides>
        <trust-anchors>
            <!-- Trust user added CAs while debuggable only -->
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

AndroidManifest.xml:

```xml
 <Application
 	android:networkSecurityConfig="@xml/network_security_config">
```
 

### 5) View network traffic

You should be good to go now, and able to see all network traffic going through your device.  This is a great way to see headers, payloads, parameters etc, so you know exactly what is being sent to your backend.