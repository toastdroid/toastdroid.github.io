---
layout: post
title: "NFC On Android, Part 1: History and Concepts"
author: Ben Berry
categories: [background]
tags: [nfc, near field communication, wireless]
---

Near Field Communication (NFC) is an RFID technology that's been rattling around in the mobile world for years and never quite managed to take off. More general forms RFID still dominate badging and physical access control. Apple Pay and Android Pay are struggling to find acceptance without anybody mentioning they run on NFC. And nobody's quite figured out why you would use an NFC sticker to control your Android phone when an app like [Tasker](https://play.google.com/store/apps/details?id=net.dinglisch.android.taskerm&hl=en) can tell your location, movement mode, and connected wireless networks. 

So I was quite surprised to find that modern versions of Android have surprisingly robust support for NFC and it's the rare Android phone that omits hardware support for it. It may be the case that NFC hasn't really taken off in any major way, but that's certainly not for lack of support in Android hardware or software. There's plenty of cool stuff to be done with NFC.<!--more-->

## Near Field Communication

NFC is a little over a decade old, governed by the Near Field Communication Forum, a trade group founded in 2004. The goal of NFC is to create a short-range version of RFID, the same technology we use for everything from badge access for buildings to electronic highway tolls. NFC was designed to be very short range, operating at a distance of a few inches, which gives it a few useful properties. 

First, by requiring you to get close, it requires less power. Almost all RFID technologies are powered by the reader providing a magnetic field that induces a current in the passive, battery-less tag. Because of the inverse-square law, every time you cut the distance in half, you cut the power consumption by a quarter. Short distance means less power. The shorter the range, the smaller the induction coil has to be and the smaller the tag can be. Being passive (not containing a battery) makes them simpler and cheaper to produce. Rather than a $10 [E-ZPass](https://en.wikipedia.org/wiki/E-ZPass) the size of a deck of cards, you can get an NFC tag the size of a quarter for less than a dollar.

Second, using very short range communication solves problems with interference and mistaking one tag for another. Even in a close-knit line of people, it's easy to tell one person's wristband from another as they pass sequentially over a scanner. Low power also reduces the risk of sniffing connections, although it doesn't eliminate it.

### History

NFC first appeared on the Android scene in 2010 in Gingerbread, with the introduction of the Nexus S as the flagship phone for the release and the first handset to have NFC hardware support. Since then, more and better support has been incrementally added, most notably in API 19’s addition of [NfcAdapter.enableReaderMode()](http://developer.android.com/reference/android/nfc/NfcAdapter.html#enableReaderMode(android.app.Activity,%20android.nfc.NfcAdapter.ReaderCallback,%20int,%20android.os.Bundle)). 

Unfortunately, many Android users are only aware of NFC because of Android Beam, the neat tech demo that turned out to be unreliable and ineffectual in real life. NFC, a low-power, low-bandwidth protocol for communicating with cheap tags was used to transfer data between expensive smartphones. At best it is finicky to precisely align two phones long enough to transfer some short like a contact, and at worst it crumbles trying to send larger data like pictures. 

Fundamentally, NFC is not technologically revolutionary. It’s a form of RFID, which has been around for decades, but the thing that makes it special is its open nature. The RFID badge you use to get access to your office building is a proprietary piece of tech that is a part of a closed system. But NFC lets your Android smartphone, made by any of a dozen manufacturers, work with tags from *totally different* manufacturers and pay for goods on payment terminals from still other manufacturers for payment processors like Visa and Discover that just use NFC as “the last mile” to let you access your money instead of a piece of plastic.

That innovation and openness is also a boon to tinkerers and developers. By being interoperable, we can insert ourselves in the middle and build cool things using existing tech where constructing a completely new closed system wouldn’t be feasible.

## NFC Players

In a basic NFC scenario, there are two sides: the reader and the tag. 

The reader is the brains of the operation, a powered device that emits a magnetic field and communicates with any tags that come into range. Most of the surface area of the tag itself is taken up by an induction coil that, when moved through this magnetic field, produces the current to power the tiny chip at the center of a tag. When a chip is brought near a reader, it powers up from the available magnetic field, and responds to requests to read and write its internal memory. As soon as it moves far enough away that it loses power, the connection is broken and the communication ends. 

Android phones, in their most basic mode, can act as a reader, putting out the magnetic field to power a tag and reading its data. This can be used for simple tasks like reading a contact card off an NFC-enabled business card or disabling the ringer when placed on the NFC tag you set on your desk at work for that purpose.

A smartphone can also “play dumb” and pretend to be a simple tag. Even though it doesn’t need to use the reader’s magnetic field for power (it has a battery), it can still communicate and respond to queries as though it were a tag. This is the mode used for most modern touchless payment systems, where a phone stands in for an NFC-enabled credit card.

Finally, two powered readers can communicate in a peer-to-peer mode where they take turns being the sender and receiver. This, for example, is the underlying technology for Android Beam. This turns out not to be all that useful, especially for communicating between phones because it’s such a short-range, low-bandwidth connection that sending anything more than brief plaintext sequences (like images or video) will be flaky and slow.

We’ll start by looking at that simple case, using a phone.

## NDEF Records: Contacts, WiFi Passwords, and More

NDEF, the NFC Data Exchange Format, defines a protocol and formats for storing data on NFC tags, and includes a number of convenient options. Simple ones include basic pieces of data like phone numbers and addresses, while advanced ones can include the SSID and password for a WiFi access point or even an application record to invoke an Android app with certain parameters. 

You can also write multiple records to a tag, and let the reader scanning it handle each in turn. For example, you could have a plaintext record that explains the contents of the tag if anyone wants to know, a second record with an app-specific record, and a third one with a simple “http://” URL in it. If the Android device doesn’t have the application installed to handle the second record, it’ll fall through to just opening the URL, which could be your website or a link to install your app on the Play Store. (Although, as a convenience, if you try to run a specific package from an NDEF record and it’s not installed, Android will take you to the Play Store listing for the app.) 

Join us next month when we’ll dive into how to read and write NDEF records on tags, respond to requests from other NFC readers, and talk about NFC security.
