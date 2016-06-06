---
layout: post
title: "NFC On Android, Part 2: Implementation Details"
author: Ben Berry
categories: [Development]
tags: [NFC, near field communication, wireless]
---

Last month, we discussed the history of Near Field Communication, what it is, and introduced some high level concepts about how it can be used. This week, we'll dive into how to put those ideas into practice on Android.

To start with, [add the NFC permission to your app's manifest](http://developer.android.com/guide/topics/connectivity/nfc/nfc.html#manifest). Then, to read and write NFC tags, you can use an activity like this:

```java
public class ReadWriteActivity extends AppCompatActivity implements ReaderCallback {
    @Override
    protected void onResume() {
        super.onResume();
        NfcAdapter.getDefaultAdapter(this).enableReaderMode(this, this, NfcAdapter.FLAG_READER_NFC_A | NfcAdapter.FLAG_READER_NO_PLATFORM_SOUNDS, null);
    }

    @Override
    protected void onPause() {
        super.onPause();
        NfcAdapter.getDefaultAdapter(this).disableReaderMode(this);
    }

    @Override
    public void onTagDiscovered(Tag tag) {
        // do work with the tag
    }
}
```
<!--more-->
This simple Activity gives you what you need to interact with a tag. In `onResume()`, we get the default `NfcAdapter` and subscribe to be notified any time a new NFC tag is scanned, and provide a few options to specify we're only interested in NFC Type A tags and that we don't want the system to make any noises when a tag is scanned. We also make sure to unsubscribe in `onPause()`.

`onTagDiscovered()` is the callback that gets invoked whenever a tag comes into range of the phone. The parameter provided to the method is a Tag object that you can inspect to see what functionality it supports and act accordingly. For example, here's an implementation that, if the tag is already formatted to store NDEF records, will show a Toast with the existing NDEF record and then write a new one to open the Ticketmaster app.

```java
@Override
public void onTagDiscovered(Tag tag) {
   try {
       for (String tech : tag.getTechList()) {
           if (tech.equals(Ndef.class.getName())) {
               Ndef ndef = Ndef.get(tag);
               ndef.connect();
               NdefMessage ndefMessage = ndef.getNdefMessage();
               NdefRecord record = ndefMessage.getRecords()[0];
               Toast.makeText(this, record.toString(), Toast.LENGTH_SHORT).show();


               NdefMessage newMessage = new NdefMessage(NdefRecord.createApplicationRecord("com.ticketmaster.mobile.android.na"));
               ndef.writeNdefMessage(ndefMessage);
               ndef.close();
           }

       }
   } catch (FormatException | IOException e) {
       // handle errors
   }
```

### Playing dumb with your smartphone

The second mode of function for NFC on Android is what is officially known as "Host Card Emulation," or HCE, which basically just means "pretending to be an NFC tag." This is the mode used when your phone is used for tap-and-pay services, so [the docs](http://developer.android.com/guide/topics/connectivity/nfc/hce.html) spend a lot of time talking about secure modules which can hold private account details in a read-only store so malware on your phone can't get access to it, but we'll ignore that for the purpose of this blog post.

For our purposes, the really cool thing about HCE is that it allows us to have **a low-power service that responds to particular requests from an NFC reader, even when your app isn't open or running**. To do this, first write out a small [HostApduService](http://developer.android.com/reference/android/nfc/cardemulation/HostApduService.html), something like this:

```java
public class MyApduService extends HostApduService {
    public MyApduService() { }

    @Override
    public byte[] processCommandApdu(byte[] commandApdu, Bundle extras) {
      //Read input bytes from commandApdu
      byte[] responseBytes = /* whatever you want to send back to the other device */;
      return responseBytes;
    }
}
```

Then, register the service in your `AndroidManifest.xml`:

```xml
<service
   android:name=".MyApduService"
   android:enabled="true"
   android:exported="true"
   android:permission="android.permission.BIND_NFC_SERVICE">
   <intent-filter>
       <action android:name="android.nfc.cardemulation.action.HOST_APDU_SERVICE"/>
   </intent-filter>
   <meta-data android:name="android.nfc.cardemulation.host_apdu_service"
              android:resource="@xml/nfc_application_ids"/>
</service>
```

You'll also need to define that file `nfc_application_ids` (or whatever you chose to call it). To do this, add a to your `res/xml` folder (you may have to create this) containing something like this: 

```xml
<?xml version="1.0" encoding="utf-8"?>
<host-apdu-service xmlns:android="http://schemas.android.com/apk/res/android"
   android:description="@string/service_description"
   android:requireDeviceUnlock="false">

   <aid-group android:description="@string/aid_group_description" android:category="other">
       <aid-filter android:name="1122334455"/>
   </aid-group>

</host-apdu-service>
```

An application ID is a hex string that is unique across the entire planet for identifying different payment vendors like Visa and Mastercard. (Remember that this is Host **Card** Emulation here; the protocol emulates EMV payment cards.) For this demo, I just made up a sample one for testing purposes (1122334455).

If you were writing an app to respond to an actual, bona fide Visa or Mastercard touchless payment terminal, you would use their publicly-registered Application IDs. If you just want a way to communicate between two devices, just have your sender device begin the handshake by specifying its Application ID as whatever you've told the receiver to respond to.

You may notice from this code that the end result is a two-way communication stream between reader and phone sending a byte stream back and forth. If you're thinking that's the basis for implementing pretty much whatever communication you want, you'd be correct. Happy hacking!

You can read [the HCE docs](http://developer.android.com/guide/topics/connectivity/nfc/hce.html) for more details, but the short version is that your service costs no battery or CPU until an application ID it is registered to handle is detected, at which time it's started. Instant response with no background load? Very cool.

### Peer-to-Peer, Android Beam, and SNEP

The third mode of NFC allows to to specify an NDEF message to be sent via Android Beam if another Android phone supporting NFC is brought in range (practically speak, back to back with the NFC antenna of each precisely aligned). This is what allows a browser to send a URL to another phone or contacts app to send a contact. By default, if you don't set this, the Android package name of your application will be sent in an NDEF message so that the receiving phone will open your app if it's installed or open the Play store to your app's page if not.

If you want to specify some other behavior, construct an `NdefMessage` comprising one or more `NdefRecords` and pass it to [`NfcAdapter.setNdefPushMessage()`](http://developer.android.com/intl/es/reference/android/nfc/NfcAdapter.html#setNdefPushMessage(android.nfc.NdefMessage, android.app.Activity, android.app.Activity...)). I would recommend you still set the first `NdefRecord` be an [Android Application Record](http://developer.android.com/intl/es/reference/android/nfc/NdefRecord.html#createApplicationRecord(java.lang.String)) so that if the receiving phone doesn't have your app installed, it will open the Play store appropriately.

#### Foreground Dispatch

The final interesting quirk of NFC on Android is the feature to, when your app is running and in the foreground, intercept actions that would otherwise be triggered by scanning a tag and act on them first. If you have an NFC tag that normally causes your phone to be put in vibrate mode, but your app uses `NfcAdapter.enableForegroundDispatch()`, it can register itself to intercede when particular types or classes of tags are scanned, and change what's done accordingly. This is useful, for example, for apps that allow reading and programming NFC tags, but has the considerable downside of needing to have the phone on and your app open already to work.

### A Digression on NFC Security

A significant problem to the widespread development of NFC in an open way is being able to do it securely. In short, for NFC to be worth anything, it has to protect the valuables contained in any system using it. 

Luckily, modern cryptography gives the tools to do this. High-quality, industry-reviewed crypto that secures our internet traffic, hard drives, and file backups is so robust, in part, because it has no secrets. Every aspect of it has been publicly reviewed and found, as of this writing, without flaw. So even though attackers know the algorithm, without the secret keys, they have no power over the system.

Unfortunately, NFC did not start out with a model like this. The first generation of cheap, deployed-everywhere NFC tags, the Mifare Classic used in metro cards across the US and around the world, [had numerous security flaws](https://www.youtube.com/watch?v=QJyxUvMGLr0). It used a proprietary encryption algorithm called CRYPTO1 that, once reverse engineered, was easy to exploit. It used poor random number generation that allowed attackers to generate the same "random" number over and over again. 

Having good crypto on an NFC tag is so important because the usefulness of the card hinges on the data it contains being secured *by the tag itself*. Yes, you could encrypt the data string that says "This metro card has a balance of $5." with a 1024-bit secret key on your server and then write the encrypted string to an NFC tag where everyone could read it. An attacker might not be able to decrypt it, but if they just clone the encrypted data to a new card, a reader will think the new card has a $5 balance all the same.

On the other hand, using the NFC tag's encryption, until you authenticate to the tag with the proper secret key, it won't send you a single bit of data. You cannot clone an encrypted tag unless you know the decryption key. You cannot modify the encrypted tag unless you know the decryption key. Even though it's a cheap $1 silicon wafer, until you give it the right password, you get no data.

The good news is that newer tags and cards are on the market that have good crypto. NXP originally said they had to come up with their own crypto algorithm because the tags are so low powered and modern robust algorithms weren't designed to run in such a spartan environment. But by going back in time to the 70s, when we also had less computing power, they found that DES, or it's modern variant of Triple DES, runs just fine, and that's what's used on the new MIFARE Ultralight C, the new-gen coin-sized tag. If your use case allows for full-size card, a subway system for example, you can use the MIFARE Plus which uses 128 bit AES, a solid modern cipher with a good key length.

So if I were building an NFC-based secure architecture today, I'd use one or both of those cards. I would also put a password on every page of the card, since there have been attacks that authenticate to one unprotected page and can then induce the card to leak information about the state of the system. (Storage on an NFC tag is divided into pages, conceptually similar to sectors and blocks on a disk.) 

Finally, I would stay appraised of any new developments in the NFC tag and reader space, whether it's new attacks (mostly unsuccessful on recent models) or new hardware (likely to be better as everything gets cheaper/faster/smaller).

I hope this overview was helpful in getting the lay of the land when it comes to NFC. If you want more information on implementing NFC on Android, check out [Google's developer documentation on the topic](http://developer.android.com/guide/topics/connectivity/nfc/index.html). I encourage you to buy a pack of NFC tags and start experimenting since your Android device almost certainly has the hardware and software support for NFC. There are a lot of cool things to be done with this technology and I can't wait to see what gets built in the future!
