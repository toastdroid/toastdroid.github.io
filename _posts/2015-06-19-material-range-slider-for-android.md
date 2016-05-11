---
layout: post
title: "Material Range Slider"
author: Patrick Jackson
categories: [development, material, design]
tags: [support library]
---

Material design is the new hotness and there are tons of [guidelines and mock ups that Google has given to us](https://www.google.com/design/spec/material-design/introduction.html).  While they have given us a few implementations with the new [Design Support Library](http://android-developers.blogspot.com/2015/05/android-design-support-library.html), we are still left to implement lots of widgets ourselves.  Recently we made a custom range picker so the user could select a range of prices.  We think it's nice so we are sharing it here with you ([source and sample app on github](https://github.com/twotoasters/MaterialRangeSlider)), and perhaps you can get ideas for implementing your own material-inspired widgets.

![](http://i.imgur.com/2hou4LT.gif)
<!--more-->

Our slider is based on Google's [sliders](http://www.google.com/design/spec/components/sliders.html) design guidelines, but with two sliders rather than one.  The design is simple - a few lines and circles representing the selected values.  The circles grow and shrink as the user touches and releases to provide visual feedback.  So where do we start?


##Custom View

The simple design of the slider lets us use primitive canvas drawing methods (line & circle) to create this widget.  This makes a custom view perfect for implementing this slider.  Essentially MaterialRangeSlider draws   4 things:

  * 1 line for the entire range
  * 1 line for the selected range
  * 2 circles for the selected min & max
  
In fact, that is all the ```onDraw``` method does:

```java
@Override
protected void onDraw(Canvas canvas) {
    drawEntireRangeLine(canvas);
    drawSelectedRangeLine(canvas);
    drawSelectedTargets(canvas);
}

private void drawEntireRangeLine(Canvas canvas) {
    paint.setColor(outsideRangeColor);
    paint.setStrokeWidth(outsideRangeLineStrokeWidth);
    canvas.drawLine(lineStartX, midY, lineEndX, midY, paint);
}

private void drawSelectedRangeLine(Canvas canvas) {
    paint.setStrokeWidth(insideRangeLineStrokeWidth);
    paint.setColor(insideRangeColor);
    canvas.drawLine(minPosition, midY, maxPosition, midY, paint);
}

private void drawSelectedTargets(Canvas canvas) {
    paint.setColor(targetColor);
    canvas.drawCircle(minPosition, midY, minTargetRadius, paint);
    canvas.drawCircle(maxPosition, midY, maxTargetRadius, paint);
}
```
Now we need to wire up touch events so we can update the location of the targets and length of the selected range line.

##TouchEvents
You've probably handled touch events in a subclass of view by providing an OnClickListener 1000 times, but if you're new to custom views, there is a lot more to consider.  Touch events must be handled in the ```onTouchEvent(MotionEvent event)``` method of the custom view. MotionEvent is the raw data indicating where the screen is being touched.  It is up to the custom view to handle these as it sees fit. 

Another consideration is multi-touch.  What happens when the user uses more than one finger at one time?  It's a small detail, but we really want the UX of our apps to be excellent, smooth and delightful, so we made sure the user can move both at the same time just as they would expect.  

The basics of ```onTouchEvent(MotionEvent event)``` are as follows:  any time the screen is touched or a finger touching the screen moves ```onTouchEvent()``` is called.  Movement of any pointer (or addition and removal of a pointer) will also fire ```onTouchEvent()```.  MotionEvent has data for all the fingers touching the screen (called pointers) even for the ones that did not move since the last ```onTouchEvent()```.  Every pointer has an id assigned to it when it first touches the screen and remains the same until the pointer is removed.  MotionEvent also contains the action for touch events.  These include:

  * MotionEvent.ACTION_DOWN - a gesture has started
  * MotionEvent.ACTION_UP - a gesture has ended
  * MotionEvent.ACTION_POINTER_DOWN - additional finger touched
  * MotionEvent.ACTION_POINTER_UP - additional finger removed
  * MotionEvent.ACTION_MOVE - finger drag is occurring
  * MotionEvent.ACTION_CANCEL - motion aborted

For MaterialRangeSlider we must know when a target is touched and moved.  The targets are very small, in fact way smaller than [recommended 48dp x 48dp size](http://www.google.com/design/spec/layout/metrics-keylines.html#metrics-keylines-touch-target-size).  To make sure our users can easily touch and drag the targets we do a bounds check to see if the touch event is within a range that is 80dp x 80dp.  This is done in ```isTouchingMinTarget(int pointIndex, MotionEvent event)``` and ```isTouchingMaxTarget(int pointerIndex, MotionEvent event)```

Once we have detected a pointer touching one of the targets we need to keep track of that pointerId between touch events so we know to move the target if that pointer moves.  We will also need to shrink the target when the user lifts that pointer.  We store these ids in ```HashSet<Integer> isTouchingMinTarget``` & ```HashSet<Integer> isTouchingMaxTarget```.  

In ```onTouch(MotionEvent event)``` you will also find some logic to handle jumping the target to a position on the line that is tapped and allowing the user to 'push' the other target when dragging past it.  Again, it's all about making the app behave as a user would expect and allowing them to quickly make a selection.

##ObjectAnimators
As you can see, during ```onDraw()``` the target circles are drawn with a radius of minTargetRadius and maxTargetRadius.  So in order to change the size of these target circles we just need to change these values.

Luckily the android framework provides us with [ObjectAnimators](http://developer.android.com/reference/android/animation/ObjectAnimator.html) which allow us to do some really cool stuff.  We can specify a starting and ending value and change a field on an object over a period of time.  In our case, we need to change the ```minTargetRadius``` and ```maxTargetRadius```.  One requirement is to have a getter and setter for these fields.  Then we can create the animators like this:

```java
private ObjectAnimator getMinTargetAnimator(boolean touching) {  
    final ObjectAnimator anim = ObjectAnimator.ofFloat(this,
        "minTargetRadius",
        minTargetRadius,
        touching ? pressedRadius : unpressedRadius);
    anim.addUpdateListener(new AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
            invalidate();
        }
    });
    anim.addListener(new AnimatorListenerAdapter() {
        @Override
        public void onAnimationEnd(Animator animation) {
            anim.removeAllListeners();
        }
    });
    anim.setInterpolator(new AccelerateInterpolator());
    return anim;
}

```

##Theming
To make the widget even easier to use and work with Material themes, MaterialRangeSlider will default to colors from the Material theme or AppCompat Material theme if you are using the AppCompat lib.  The outside range color will be ```colorControlHighlight``` and the inside range and targets will be ```colorControlNormal```.  The colors can also be set in xml:

```xml
<com.ticketmaster.mobilestudio.materialrangeslider.MaterialRangeSlider 
        android:id="@+id/price_slider"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:minHeight="96dp"
        app:insideRangeLineColor="@color/my_color1"
        app:outsideRangeLineColor="@color/my_color2"
        app:targetColor="@color/my_color2"
        app:outsideRangeLineStrokeWidth="2dp"/>
```

All of the theming and xml attribute code can be found in the ```init()``` method in MaterialRangeFinder.  

That's it for now!  Remember, if you want to dig deeper or use this in your own project the [code and sample project are up on Github](https://github.com/twotoasters/MaterialRangeSlider).
