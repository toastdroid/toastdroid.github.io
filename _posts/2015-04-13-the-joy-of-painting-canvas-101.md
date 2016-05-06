---
layout: post
title: "The Joy of Painting: Canvas 101"
author: Curtis Martin
categories: [development]
tags: [ui, canvas, paths, drawing]
---

In [a previous blog post](/2014/06/21/creating_reusable_custom_components), we touched on creating custom UI components containing several stock Android controls. But what if you want to create a completely custom View, complete with its own drawing behavior? To accomplish that, you'll need to become familiar with the `Canvas`.

## What Canvas?

Just as in actual painting, an Android `Canvas` is a surface on which drawing can be performed. Every View subclass has an easy way to get access to that View's Canvas. Simply override the `onDraw(Canvas canvas)` method like so:

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    //Draw some stuff on the Canvas here!
}
```
<!--more-->

You may think you're ready to draw now that you've got the Canvas, and you're half right. Before you draw, you need something colorful to draw with. Just like in the real world, we're going to draw on our Canvas with some good old-fashioned `Paint`.

## Creating Paints

`Paint` objects define how you will be drawing to the Canvas. At the most basic level, Paints have a few properties you'll typically need to care about:

* `setColor(int color)` - The color that you want to draw with
* `setStyle(Paint.Style style)` - `STROKE` or `FILL`, depending if you want to draw only around the perimeter of the area you designate, or fill the area with the color
* `setAntiAlias(boolean aa)` - If this is set to true, smoothing will be applied to make your lines look less jagged
* `setStrokeWidth(float width)` - Sets the width of stroke in pixels

Creating new object instances and performing business logic in `onDraw()` will cause delays in our drawing operations, so we'll create our Paints when the View is created and save them for later. In this example, we'll create two Paints, which will be blue and red.

```java
private Paint blue;
private Paint red;

public CustomView(Context context) {
    super(context);
    setupPaints();
}

private void setupPaints() {
    Resources res = getResources();

    blue = new Paint();
    blue.setColor(res.getColor(R.color.blue));
    blue.setStyle(Style.FILL);
    blue.setAntiAlias(true);

    red = new Paint();
    red.setColor(res.getColor(R.color.red));
    red.setStyle(Style.STROKE);
    red.setStrokeWidth(10f);
    red.setAntiAlias(true);
}
```

Now that we have defined _how_ we're going to draw, we need to define _what_ we want to draw. We're going to use the `Path` class for this.

## Path Basics

`Paths` are objects that specify a set of ordered coordinates that denote, as you may have guessed, the path that we want to draw in the Canvas. Path coordinates are measured in pixels and start at the top-left of the Canvas. For instance, the coordinate (100, 100) would be 100 pixels to the right and 100 pixels below the origin.

![Origin](/assets/2015-04-13-the-joy-of-painting-canvas-101/origin.png)     ![Offset](/assets/2015-04-13-the-joy-of-painting-canvas-101/offset.png)

The Path class comes with several helpful methods for building a Path. Here are some of the noteable ones:

* `moveTo(float x, float y)` - Move the current position to the given coordinate, without extending the Path
* `lineTo(float x, float y)` - Create a line starting at the current position and ending at the given coordinate
* `arcTo(RectF oval, float startAngle, float sweepAngle)` - Add an arc to the end of the path
* `addRoundRect(RectF rect, Path.Direction dir)` - Adds a rounded rectangle to the end of the path
* `addCircle(float x, float y, float radius, Path.Direction dir)` - Add a circle to the path, with center point (x, y)
* `addPath(Path src)` - Adds a copy of an existing Path to the end of the current path
* `close()` - Close the Path, adding a straight line that spans the current position and the starting position
* `offset(float dx, float dy)` - Move the entire existing path by dx pixels horizontally and dy pixels vertically
* `reset()` - Clear all points on the Path

Each of the above method calls append to the end of the Path, building off each other sequentially. There are many more methods in Path that can help you build more complicated Paths. For our example, we're going to use `addCircle()` and build a couple of circles. Just like with Paints, we're going to allocate our Paths prior to drawing them.

```java
public CustomView(Context context) {
    super(context);
    setupPaints();
    setupPaths();
}

...

private void setupPaths() {
    bluePath = new Path();
    redPath = new Path();
}
```

The Canvas's size will match the size of our custom View. Since we don't know the size of the Canvas until the View has been measured, we're going to wait to build our Paths until `onMeasure()` is called. We could just do this in `onDraw()`, but you do __not__ want to run business logic during your draw operations for performance reasons. First let's create the Path for the blue circle:

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    float centerX = getWidth() / 2f;
    float centerY = getHeight() / 2f;

    bluePath.reset();
    bluePath.addCircle(centerX, centerY, 200f, Direction.CW);
}
```

The center of the circle will be the center of the Canvas. We also give the circle a radius of 200 pixels and define its direction as clockwise (doesn't matter for the sake of the example). We'll also go ahead and create our red circle's Path the same way, but with a smaller radius:

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    float centerX = getWidth() / 2f;
    float centerY = getHeight() / 2f;

    bluePath.reset();
    bluePath.addCircle(centerX, centerY, 200f, Direction.CW);
    redPath.reset();
    redPath.addCircle(centerX, centerY, 100f, Direction.CW);
}
```

Now that we have our Paths, it's time to make our masterpiece!

## Drawing the Paths

Now we just have to draw our circles on the Canvas with the Paints we created earlier. We can use the handy `Canvas.drawPath()` method for this. Let's start with the blue circle:

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    canvas.drawPath(bluePath, blue);
}
```

And here's our beautiful blue circle in all its glory!

![Blue Circle](/assets/2015-04-13-the-joy-of-painting-canvas-101/blue_circle.png)

Now let's draw the red circle. It will show up right in the middle of the blue one we just drew:

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    canvas.drawPath(bluePath, blue);
    canvas.drawPath(redPath, red);
}
```

![Blue and Red Circles](/assets/2015-04-13-the-joy-of-painting-canvas-101/blue_red_circle.png)

It's important to note that each `Canvas.draw...()` call draws _after_ the last call. This means that if you draw multiple overlapping Paths, they will be drawn overtop of each other in the order that the drawing operations occurred. In the above example, we drew the red circle after the blue one. If we flip those lines, the blue circle will completely cover the red one, so all we'll see is the blue circle. Also, each draw operation that occurs over the same pixel will cause overdraw on that pixel, so you should try to draw as few times over the same spot as possible to maximize performance.

So now we've drawn a couple of pretty circles. But sometimes you're hungry, and you really want a donut. Let's see what we can do about that.

## Setting Clip Paths

When drawing on the Canvas, there are times that you may want to draw only a particular portion of a Path that you've created. For instance, maybe I want to draw the blue circle from earlier, but I want to cut a hole in the middle of it. You could do this one of two ways:

1. The Bad Way - Draw the blue circle, then draw a circle in the middle that matches the color that's behind the blue circle, giving the illusion that there's a hole in the middle. This is bad because it uses multiple drawing operations, which means multiple draw operations over the same area, which means __overdraw__. Bad stuff.

2. The Good Way - Set a "clip path" on the Canvas prior to drawing the blue circle, basically telling the Canvas not to draw over the clip path. Only one draw operation is required.

So how might we draw our blue donut? Let's make a minor modification to the existing code:

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    canvas.save();
    canvas.clipPath(redPath, Op.DIFFERENCE);
    canvas.drawPath(bluePath, blue);
    canvas.restore();
}
```

![Donut](/assets/2015-04-13-the-joy-of-painting-canvas-101/donut.png)

And just like that, we have a donut. Let's step through what's going on in the code:

* `canvas.save();` - Saves the current clip state of the Canvas. We do this prior to setting the clip path so we can restore the Canvas back to it's previous state after we're done.
* `canvas.clipPath(redPath, Op.DIFFERENCE);` - Sets the clip path to the red circle's path using the `DIFFERENCE` operator. What this means is that only the difference between the bluePath and redPath will be drawn.
* `canvas.drawPath(bluePath, blue);` - Draws the blue circle, minus the clip path.
* `canvas.restore();` - Restores the Canvas's clip path back to what it was when we called `canvas.save()`. This is done so that next time we draw we don't accidentally use our previous clip path.

When setting a clip path, there are [several different operators](http://developer.android.com/reference/android/graphics/Region.Op.html), including `INTERSECT`, `DIFFERENCE`, and `REPLACE`. These operators change how the clip path affects your drawing in various ways. The best way to understand how they work is to try them out on what you're drawing. Replace `Op.DIFFERENCE` with `Op.INTERSECT` and see what happens!

That about wraps it up for this tutorial. In the future we'll cover some more advanced topics, such as using a `BitmapShader`. Now go out there and make some sweet Android art!