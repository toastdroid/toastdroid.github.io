---
layout: post
title: "CWAC-Camera - An Easier Way to Take Photos in Android"
author: Curtis Martin
categories: [development, libraries, tools]
tags: [camera, opensource]
---

One of Android's greatest strengths is its ability to run on a plethora of different types of devices and hardware configurations. For developers, however, this presents an interesting problem: how do you go about supporting all these different hardware variations? While Google has done a fine job of abstracting the finer details of hardware interaction away from the developer in most cases, the Android camera API still poses a substantial number of hurdles to this day.<!--more-->

In order to use a device's camera, an application must perform roughly the following steps:

1. Declare the **android.permission.CAMERA** permission in the manifest
2. Get a reference to a device's camera (front-facing, back, etc.) and **open()** it
3. Set parameters on the camera, including the focus mode, flash mode, photo resolution, etc.
4. Create a SurfaceView on which to render the camera preview and pass its reference to **setPreviewDisplay()**
5. Start the camera preview
6. Take the photo
7. Handle the data that the camera passes back to save your photo properly
8. Stop the preview
9. Release the camera so other applications can now use it

Note that this just covers taking a single photo. If you want to handle retaking a photo, taking video, or any other action, that list grows significantly. The trickiest part, however, is that performing any of these actions at the wrong time can result in the camera outright failing until it is reinitialized. It's harrowing stuff, to be sure.

Enter **CWAC-Camera**: https://github.com/commonsguy/cwac-camera

The CWAC-Camera library was written by Mark Murphy, aka CommonsWare (that guy who has answered half of your Android-related Stack Overflow questions). It removes much of the pain that can come with the Android camera API, as well as the guesswork sometimes required to perform simple camera actions. The CWAC-Camera API revolves around three main components: **CameraView**, **CameraFragment**, and **CameraHost**.

CameraView abstracts away the main components of the camera API, and largely doesn't even need to be touched by the developer. It handles creation of the preview rendering surface automatically, including maintaining the aspect ratio of the camera preview on all device resolutions and screen sizes, and provides simplified API calls to access the main features of the camera.

CameraFragment is a Fragment that houses and interacts with the CameraView. The developer simply has to create an instance of a CameraFragment, add it to the Activity using the FragmentManager, and call methods like **autofocus()**, **takePicture()**, and **startRecording()** on it to control the camera with very little setup required.

CameraHost provides an interface class that receives callbacks from the camera. By implementing CameraHost, you can set the camera's configuration, preview size, and picture size, as well as handle the data coming back from the camera once a photo is taken. CommonsWare provides a simple implementation of this interface, appropriately called SimpleCameraHost, that serves as a great template for how to handle these callbacks. You can check it out [here](https://github.com/commonsguy/cwac-camera/blob/master/camera/src/com/commonsware/cwac/camera/SimpleCameraHost.java). Here's a snippet of some of the major pieces of this CameraHost implementation:

```java
public class SimpleCameraHost implements CameraHost {

	/**
	*This callback happens just before the camera takes a picture. Here you
	*can adjust the passed in Camera.Parameters object with methods like
	*parameters.setFocusMode(), parameters.setFlashMode(), etc. Any modifications
	*Made here are passed back to the camera automatically.
	*/
	@Override
  	public Camera.Parameters adjustPictureParameters(PictureTransaction xact,
                                                   Camera.Parameters parameters) {
    	return(parameters);
  	}

	/**
	*This callback happens just before the camera starts showing a preview. This method
	*works the same way as adjustPictureParameters, but the callback occurs earlier
	*in case the camera needs to behave differently prior to the user snapping a picture.
	*/
  	@Override
  	public Camera.Parameters adjustPreviewParameters(Camera.Parameters parameters) {
    	return(parameters);
  	}

	/**
	*Implementing this method allows you to get a callback at the moment a picture is taken.
	*One example of a good use for this is displaying unhiding Views that assist the user
	*in previewing the captured photo.
	*/
	@Override
	public Camera.ShutterCallback getShutterCallback() {
	    return(null);
	}

	/**
	*SimpleCameraHost does not use this particular saveImage method (there's
	*another one that gives you a byte[] representation of the image), but implementing
	*this gives you a chance to save your photo off to the filesystem, display it
	*in an ImageView, or any number of other things. It also packages the photo
	*up in a Bitmap, which is a bit easier to deal with than a byte[].
	*/
	@Override
	public void saveImage(PictureTransaction xact, Bitmap bitmap) {
	  	// no-op
	}
}
```


So, how easy is it to use CWAC-Camera to perform the steps I outlined above? Let's look at an example:

```java
public class CameraDemoActivity extends Activity {

    private final String TAG_CAMERA_FRAGMENT = "camera_fragment";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_camera_demo);

        //Create the CameraFragment and add it to the layout
        CameraFragment f = new CameraFragment();
        getFragmentManager().beginTransaction()
                .add(R.id.container, f, TAG_CAMERA_FRAGMENT)
                .commit();

        //Set the CameraHost
        f.setHost(new SimpleCameraHost(this));

        //Set an onClickListener for a shutter button
        findViewById(R.id.shutter_button).setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View view) {
                takePicture();
            }
        });
    }

    /**
     * Checks that the CameraFragment exists and is visible to the user,
     * then takes a picture.
     */
    private void takePicture() {
        CameraFragment f = (CameraFragment) getFragmentManager().findFragmentByTag(TAG_CAMERA_FRAGMENT);
        if (f != null && f.isVisible()) {
            f.takePicture();
        }
    }
}
```

Assuming that your CameraHost implementation is correct (or you're extending SimpleCameraHost), then this is all you need to take a picture. By default, SimpleCameraHost will allow the user to continue taking photos. If you want to limit the user to one photo and then show a preview of the captured image, all you have to do is implement the **useSingleShotMode()** method in your CameraHost, like so:

```java
@Override
public boolean useSingleShotMode() {
	return true;
}
```

Want to make the camera focus? That's easy too. Just get a reference to your CameraFragment and call **autofocus()**:

```java
CameraFragment f = (CameraFragment) getFragmentMananger().findFragmentByTag(TAG_CAMERA_FRAGMENT);
f.autofocus();
```

There's a lot more you can do with the CWAC-Camera library than what you see here, such as video recording. It's extremely useful for any developer out there who wants to create an app with a custom camera implementation, and it is updated frequently (9 days ago as of this writing). If you're rolling your own camera app, I highly recommend checking it out.
