---
layout: post
title: "MessageApi: Simple Conversations with Android Wear"
author: Curtis Martin
categories: [development]
tags: [wear, messageapi, bluetooth]
---

Android Wear has finally arrived, and we can all finally live out our childhood dreams of being Dick Tracy. A new frontier for both developers and consumers alike, Wear allows us to use our existing Android devices in new and exciting ways. In fact, the Wear platform itself is absolutely dependent on communicating with our phones and tablets. Google has come up with some pretty simple APIs that facilitate this communication, luckily for us. Today, we're going to talk about MessageApi, which is the simplest way we can pass information to and from a Wear device.<!--more-->

## What is MessageApi?

Google describes the MessageApi as "a one-way communication mechanism that's meant for 'fire-and-forget' tasks". This is opposed to Wear's DataApi, which is meant for more long-term syncing between a Wear device and a phone/tablet. All Wear APIs are included with Google Play Services 5.0, and though that particular Play Services release is available on many devices, it's important to note that Wear itself supports only 4.3 devices and above. This is because Wear requires Bluetooth LE, which is only available in 4.3 and up.

The MessageApi flow consists of two pretty obvious steps: sending a message to a device, and recieving a message from the sender. In this example, we will send a message from our Wear device to the phone it is paired with, and display a Toast on the phone as a result.

## Sending a Message

We're going to work under the following assumptions:

1. We have an Android phone
2. We have an Android Wear device that has been paired to the phone
3. Both devices are in range of each other
4. Only one Wear device is paired to the phone, and only one phone is paired to the Wear device
5. The Wear app will be doing all of its work inside an Activity

The first step in doing any Wear communication is to get a reference to the Wear API. The ```GoogleApiClient``` class will make this very easy.

```java
private GoogleApiClient getGoogleApiClient(Context context) {
    return new GoogleApiClient.Builder(context)
            .addApi(Wearable.API)
            .build();
}
```

The second step in sending a message is finding out where the message should be sent. Devices that are connected over Bluetooth are identified in the Wear API as "nodes". Since our Wear device is already connected to our phone, we need to get a list of the nodes that are connected to it. Now that we have a GoogleApiClient, we'll use it to get the node list and pull the ID of the first node we find (since we're assuming only one connection).

```java
private void retrieveDeviceNode() {
	GoogleApiClient client = getGoogleApiClient(this);
    new Thread(new Runnable() {
        @Override
        public void run() {
            client.blockingConnect(CONNECTION_TIME_OUT_MS, TimeUnit.MILLISECONDS);
            NodeApi.GetConnectedNodesResult result =
                    Wearable.NodeApi.getConnectedNodes(client).await();
            List<Node> nodes = result.getNodes();
            if (nodes.size() > 0) {
                nodeId = nodes.get(0).getId();
            }
            client.disconnect();
        }
    }).start();
}
```

This block of code is simply connecting to the Wear API, making a call to ```NodeApi.getConnectedNodes()```, setting a String called ```nodeId``` to the ID of the first node in the List, and then disconnecting from the Wear API service. Since we're using a blocking connection and awaiting the result, this action must be done in a background thread. If you wanted to do this in a more asynchronous manner, you could use ```setResultCallback(ResultCallback<R> callback)``` instead of ```await()```.

The third and final step is actually sending a message. This looks very similar to how we got the node ID, in that we make a connection to the Wear API, send a message, and disconnect.

```java
private void sendToast() {
	GoogleApiClient client = getGoogleApiClient(this);
    if (nodeId != null) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                client.blockingConnect(CONNECTION_TIME_OUT_MS, TimeUnit.MILLISECONDS);
                Wearable.MessageApi.sendMessage(client, nodeId, MESSAGE, null);
                client.disconnect();
            }
        }).start();
    }
}
```

The act of sending a message involves a call to ```Wearable.MessageApi.sendMessage()```. ```sendMessage()``` takes four arguments:

1. The GoogleApiClient that we will send the message through
2. The node ID (String) of the device we want to send to
3. A path (String), typically denoting some method or function to invoke on the destination device
4. A byte[] of data, which Google recommends be no larger than 100KB in size

In this example, the data parameter is null because we don't have any extra data to send over. It will only need to know the path, which will denote the action for the phone to take.

## Receiving a Message

Now that the Wear app is sending a message, we need to allow the phone to receive it. There are a couple ways we can receive messages:

1. Register a listener using ```Wearable.MessageApi.addListener(GoogleApiClient client, MessageListener listener)```
2. Use a ```WearableListenerService```, and be subscribed to Wear events automatically

Since the phone needs to listen for these particular messages in the background (our phone app doesn't have an Activity), we're going to go with Option #2. Option #1 also creates a little more work, as you need to make sure to add/remove listeners at proper times (such as in ```onStart()```/```onStop()``` in an Activity).

Both methods stated above will result in us having access to an ```onMessageReceived(MessageEvent messageEvent)``` callback. This method is invoked any time a message is sent from our Wear device using the phone's node ID. First, we need to add the WearableListenerService to our mobile app's AndroidManifest.xml inside its ```<application>``` tag. The Service must register itself for the ```"com.google.android.gms.wearable.BIND_LISTENER"``` action in order to receive Wear API events.

```xml
<application>
	...
	<service
        android:name=".ListenerService" >
        <intent-filter>
            <action android:name="com.google.android.gms.wearable.BIND_LISTENER" />
        </intent-filter>
    </service>
</application>
```

Now let's take a look at our ListenerService, along with the callback.

```java
public class ListenerService extends WearableListenerService {

    @Override
    public void onMessageReceived(MessageEvent messageEvent) {
        showToast(messageEvent.getPath());
    }

    private void showToast(String message) {
        Toast.makeText(this, message, Toast.LENGTH_LONG).show();
    }

}
```

The MessageEvent object passed into ```onMessageReceived``` has getter methods for retreiving the node ID of the sender, as well as the path and byte[] that we passed into ```sendMessage```. Here, we're simply taking the String we sent as the path parameter and displaying it in a Toast. Pretty darn simple eh? This is literally all the logic we need to handle the message that was sent.

There may be a case that we want to be able to send a message back to the Wear device from the phone at some point. In this case, it would be a good idea to store that node ID that came over with the message. We easily do that with a quick modification.

```java
public class ListenerService extends WearableListenerService {

    String nodeId;
    
    @Override
    public void onMessageReceived(MessageEvent messageEvent) {
        nodeId = messageEvent.getSourceNodeId();
        showToast(messageEvent.getPath());
    }

    private void showToast(String message) {
        Toast.makeText(this, message, Toast.LENGTH_LONG).show();
    }

}
```

Since we're using a WearableListenerService, it's actually easier to send a message back to the phone. When a message is received, the WearableListenerService will automatically kick off and process it in the background, so there's no need to wrap the reply logic inside its own Thread. We simply need to call the following method from ```onMessageReceived```:

```java
private void reply(String message) {
	GoogleApiClient client = new GoogleApiClient.Builder(context)
			.addApi(Wearable.API)
            .build();
	client.blockingConnect(CONNECTION_TIME_OUT_MS, TimeUnit.MILLISECONDS);
    Wearable.MessageApi.sendMessage(client, nodeId, message, null);
    client.disconnect();
}
```

That's all there is to it! Keep in mind, though we're only sending very simple data across right now, we can make the phone behave differently simply by passing different path values in a message and adding logic to respond to those paths in our Service.

If you'd like to see the full example of this, you can check out the [Two Toasters Github repo](https://github.com/twotoasters/Wear-MessageApiDemo).