---
layout: post
title: "Google Maps with Custom View Overlays"
author: Patrick Jackson
categories: [development, libraries, tools]
tags: [google maps sdk, custom view]
---

Google Maps SDK on Android is great.  However, for a recent project at Two Toasters we had a use case that is not handled by the Maps SDK directly.  In our case we needed a dotted circle on top of the map and query data based on the radius of the circle.  The circle and marker remain in the same place as the user scrolls and pans.  There is nothing in the Google Maps SDK to handle this for us.  Enter Custom Views!

<img src="http://giant.gfycat.com/ShyAlarmingBoubou.gif"/><!--more-->

##Create the Custom View
 
Writing custom views can be fairly complex, but this one will be straightforward.  All views in Android are subclasses of the `View` class.  You may also subclass specific views if you, for example the Button class.  For our circle we extend `View`.  When extending `View` in Android you need to understand the view lifecycle.  The main methods you will need to override are `onDraw` and `onMeasure`.

```java
public class DottedCircleView extends View {
    
	@Override
		protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    		width = MeasureSpec.getSize(widthMeasureSpec);
    		height = MeasureSpec.getSize(heightMeasureSpec);
	
		if (radius == 0) {
		    radius = Math.min(width, height) / 2 - (int) p.getStrokeWidth();
		  }			    

    		super.onMeasure(widthMeasureSpec, heightMeasureSpec);
		}

		@Override
		public void onDraw(Canvas canvas) {
    		super.onDraw(canvas);
    		
        	canvas.drawCircle(width / 2, height / 2, radius, p);
    		//draw the map marker in middle of the circle
    		canvas.drawBitmap(pin.getBitmap(), (width / 2) - pinXOffset, (height / 2) - pinYOffset, null);
    		invalidate();
		}
}
```

A key point about the View class is that `onDraw` is called frequently, actually when the view is initially drawn and when `invalidate` is called.  So its critical to keep any cpu intensive or long running operations out of the `onDraw` method.  For our case we are going to create a `setup` method to do some calculation and object creation.  Doing this ensures it is only called once when the view is instantiated and not every time the view is redrawn.

For a deeper dive into custom views checkout the [Google I/O 2013 session on Writing Custom Views for Android](https://developers.google.com/events/io/sessions/325615129)


```java
private void setup() {
    p = new Paint();
    p.setColor(getResources().getColor(color));
    p.setStrokeWidth(
	  		getResources().getDimension(R.dimen.dotted_circle_stroke_width));
	  		DashPathEffect dashPath = new DashPathEffect(new float[]{DASH_INTERVAL,
            DASH_INTERVAL}, (float) 1.0);
    p.setPathEffect(dashPath);
    p.setStyle(Style.STROKE);
    pin = (BitmapDrawable) getResources().getDrawable(R.drawable.map_pin);
    pinXOffset = pin.getIntrinsicWidth() / 2;
    pinYOffset = pin.getIntrinsicHeight();
}
```

##Create Layout for Activity/Fragment
Extending the View class allows you to do all sorts of cool things.  One of them being you can place your view in XML layouts at set attributes just like you would any SDK views.  Using a FrameLayout we can place the our custom view (DottedCircleView) on top of the map fragment.  Note that our XML includes the entire package name for DottedCircleView 

```xml
<?xml version="1.0" encoding="utf-8"?>

<FrameLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:background="@color/White"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <FrameLayout
        android:id="@+id/mapContainer"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>

    <com.ourproject.view.DottedCircleView
        android:id="@+id/dottedCircleView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_marginLeft="24dp"
        android:layout_marginRight="24dp"
        toast:circleColor="@color/Green"
        toast:radius="50"/>

</FrameLayout>
```

##Setting Attributes From XML

Android views allow you to set attributes easily from the xml layout (i.e. `android:layout_margin="16dp"`).  You can do the same with your custom views!  This is great for creating reusable views that you can plug into a layout and quickly set attributes without writing Java code.  

First step is to add an entry into the `attrs.xml` file in your `/res/values` folder.  This is where we define the values that can be set in your view's xml:

```xml
<resources>

    <declare-styleable name="DottedCircleView">
        <attr name="radius" format="integer" />
        <attr name="circleColor" format="color" />
    </declare-styleable>

</resources>
```

Next we must get those values in your view's code.  They are passed in the constructor as the `AttributeSet` argument.  Below is how to retrieve them:

```java
public DottedCircleView(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);

    TypedArray a = context.obtainStyledAttributes(attrs,
            R.styleable.DottedCircleView, 0, 0);
    color = a.getColor(R.styleable.DottedCircleView_circleColor,
            R.color.blue);
    radius = a.getInteger(R.styleable.DottedCircleView_radius, 0);
    a.recycle();

    setup();
}
```

Finally we can set the values in the xml.  To have the xml parser recognize your custom attributes you must add the `http://schemas.android.com/apk/res-auto` namespace to the xml.  This can be named whatever you wish.  Here it is named `toasters`.

```xml
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:toasters="http://schemas.android.com/apk/res-auto"
    android:background="@color/White"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <FrameLayout
        android:id="@+id/mapContainer"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>

    <com.ourproject.view.DottedCircleView
        android:id="@+id/dottedCircleView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_marginLeft="24dp"
        android:layout_marginRight="24dp"
        toast:circleColor="@color/Green"
        toast:radius="50"/>
</FrameLayout>
```


##Get Real World Distance from Circle

We have our nice dotted circle and marker drawn on top of the map.  Awesome.  Now how in the world can we get real world distance from the radius of the circle so we can query the api?  Fortunately Google Maps SDK provides a handy util method called [fromScreenLocation(Point point)](http://developer.android.com/reference/com/google/android/gms/maps/Projection.html#fromScreenLocation(android.graphics.Point)). and the Android SDK provides a [distanceTo(Location location)](http://developer.android.com/reference/android/location/Location.html#distanceTo(android.location.Location)) method in the location class. 

fromScreenLocation(Point point) will return the lat/lng of a screen point, which is exactly what we need.  So we just get the lat/lng of the center of the circle and a point on the circle to get the real world radius. 

Below is how we are getting the query distance:

```java
/**
 * @return distance in meters from center of circle to a point on the circle
 */
public double getSearchRadius() {
    //Google Map LatLng objects must be converted to android Location objects
    //in order to use distanceTo()

    Point circlePoint = getScreenCenter();
    circlePoint.x = circlePoint.x - getCircleRadius();

    Point centerPoint = getScreenCenter();

    LatLng centerLatLng = map.getProjection().fromScreenLocation(centerPoint);
    Location center = new Location("");
    center.setLatitude(centerLatLng.latitude);
    center.setLongitude(centerLatLng.longitude);


    LatLng circleLatLng = map.getProjection().fromScreenLocation(circlePoint);
    Location circlePointLocation = new Location("");
    circlePointLocation.setLatitude(circleLatLng.latitude);
    circlePointLocation.setLongitude(circleLatLng.longitude);

    return center.distanceTo(circlePointLocation);

}
```

##Wrapping Up

If the Android SDK doesn't have a view that you need then you can roll your own by extending the View class.  This is a great way to create views that can be reused in your app, or multiple apps.  Finally putting the layout in XML and using custom attributes is a great way to make reusing and customizing the view easy.  We're just scratching the surface here, if your looking for more check out the [Google Android Developers site](http://developer.android.com/training/custom-views/index.html).

