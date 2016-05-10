---
layout: post
title: "Creating Reusable Custom Components"
author: Curtis Martin
categories: [development]
tags: [ui, views]
---

The stock Views provided by Android are pretty flexible, and usually they're good enough if you're adhering to the [Android Design Guidelines](https://developer.android.
com/design/index.html). Sometimes you might want to add some special functionality to an existing View, or modify its look outside of what the stock attributes will allow. 
In those cases, you need to make a custom component. Fortunately for us, that's quite easy to do!

In this example, we need to make a CheckBox with the following requirements:
* Two lines of text: a title and a description
* The "box" itself aligns to the right side of the View
* Tapping anywhere in the View will toggle the state of the CheckBox<!--more-->

## Creating a layout for the CheckBox

Let's start with our xml layout. It's a pretty simple layout, with a couple LinearLayouts, two TextViews, and a CheckBox.

```xml
<?xml version="1.0" encoding="utf-8"?>
<merge xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">
    <LinearLayout
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:layout_marginTop="16dp"
        android:layout_marginBottom="16dp"
        android:layout_gravity="center_vertical"
        android:layout_weight="1">
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            style="@style/TextAppearance.Styled.P1Reg.Grey"
            android:id="@+id/txt_checkBox_title"/>
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/txt_checkBox_detail"
            style="@style/TextAppearance.Styled.P1Light.Grey"/>
    </LinearLayout>
    <CheckBox
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center_vertical"
        android:id="@+id/chk_checkBox"/>
</merge>
```

Here's how it looks:

![How it looks](/assets/2014-06-21-creating_reusable_custom_components/checkbox_sample.png)

## Creating a custom View subclass

Next, we need to make a class for the View. The class will take care of inflating our layout, and it'll contain any additional functionality we may need.

```java
public class MyCheckBox extends LinearLayout {

    CheckBox checkBox;
    TextView title;
    TextView detail;
    Rect hitRect;

    public MyCheckBox(Context context) {
        super(context);
        init(context);
    }

    public MyCheckBox(Context context, AttributeSet attrs) {
        super(context, attrs);
        init(context);
    }

    public MyCheckBox(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        init(context);
    }

    private void init(Context context) {
        View.inflate(context, R.layout.my_checkbox, this);
        setDescendantFocusability(FOCUS_BLOCK_DESCENDANTS);
        title = (TextView) findViewById(R.id.txt_checkBox_title);
        detail = (TextView) findViewById(R.id.txt_checkBox_detail);
        checkBox = (CheckBox) findViewById(R.id.chk_checkBox);
        hitRect = new Rect();
        setOnTouchListener(new OnTouchListener() {
            @Override
            public boolean onTouch(View view, MotionEvent motionEvent) {
            	view.getHitRect(hitRect);
                if (hitRect.contains((int) motionEvent.getX(), (int) motionEvent.getY())) {
                    motionEvent.setLocation(0.0f, 0.0f);
                    checkBox.dispatchTouchEvent(motionEvent);
                }
                return true;
            }
        });
    }

    public void setTitleText(String text) {
    	title.setText(text);
    }

    public void setDetailText(String text) {
    	detail.setText(text);
    }

    public void setChecked(boolean checked) {
        checkBox.setChecked(checked);
    }


    public void setOnCheckedChangeListener(
            CompoundButton.OnCheckedChangeListener listener) {
        checkBox.setOnCheckedChangeListener(listener);
    }
}
```

The ```init(Context context)``` method handles common initialization logic. It inflates the layout, gets a reference to the right-aligned CheckBox, and sets an 
onTouchListener on the entire View. When the View is touched, a touch event is dispatched to the CheckBox to toggle its checked state.

To mimic the behavior of the stock Android CheckBox, we also have a couple more methods: ```setChecked(boolean checked)``` and ```setOnCheckedChangeListener(
OnCheckedChangeListener listener)```. These methods simply make the same calls on the CheckBox itself.

In order to make it so that tapping anywhere in the layout will trigger the CheckBox, we must call ```setOnTouchListener``` and dispatch all resulting MotionEvents that 
occur inside the parent View's hitbox to the CheckBox. It's important to note here that a MotionEvent's location is relative to the bounds of the View that is receiving the 
event, so all MotionEvents passed to the CheckBox must have their locations reset to be inside its bounds. The easiest way to do this is to call ```motionEvent.
setLocation(0.0f, 0.0f);```, since 0, 0 will always
be within the bounds of a View of any size.

## Using the custom component

To add our fancy new custom component to an xml layout, we simply need to put the following lines in the layout:

```xml
<com.myapp.MyCheckBox
	android:layout_width="match_parent"
	android:layout_height="wrap_content"
	android:id="@+id/chk_customCheckBox"/>
```

## Adding custom attributes

The custom component will work just fine as-is, but it's missing a key feature: the ability to set the content of the TextViews and the checked state of the CheckBox in xml.
 It's pretty standard fare, and unless you want to only use the custom View once in your app it'd be good to add that capability. We'll need to do three things:

1. Create a custom attribute set
2. Set the attributes in the layout that includes the custom component
3. Add logic in the View subclass to handle reading the attributes and assigning their values

### Creating a custom attribute set

Custom attribute sets go in your project in the ```res/values/attrs.xml``` file. Attribute sets can contain several simple datatypes, such as ints, floats, and Strings. In 
our case, we'll need two String attributes and one boolean attribute.

```xml
<resources>
   <declare-styleable name="MyCheckBox">
       	<attr name="titleText" format="string" />
        <attr name="detailtext" format="string" />
        <attr name="checked" format="boolean" />
   </declare-styleable>
</resources>
```

One important thing to note is that attribute names can conflict, even if they are contained in different attribute sets. If you have multiple custom components/Views in 
your app, make sure that you give your attributes unique names.

### Setting the attributes in the layout

Remember how we added the MyCheckBox View to a layout earlier? We need to make some slight modifications to use the new attributes.

In the parent layout, we need to add a reference to the namespace for our attributes. Android has made this pretty easy for us. Let's say we have a MyCheckBox inside a 
LinearLayout. The layout xml may look like this:

```xml
<LinearLayout
	xmlns:android="http://schemas.android.com/apk/res/android"
	android:layout_width="match_parent"
	android:layout_height="match_parent"
	android:orientation="vertical">

	<com.myapp.MyCheckBox
		android:layout_width="match_parent"
		android:layout_height="wrap_content"
		android:id="@+id/chk_customCheckBox"/>

</LinearLayout>
```

In order to reference our custom attributes, we need to add a namespace that references the custom attribute set. This is done by using the ```xmlns``` attribute. You've 
probably seen ```xmlns:android="http://schemas.android.com/apk/res/android"``` before, since it needs to be in every layout you make. The ```xmlns:android``` part of that 
line declares a namespace named "android", which is why all Android layout attributes begin with ```android:```. We'll call our new namespace "myapp" and add it to the 
LinearLayout like so:

```xml
<LinearLayout
	xmlns:android="http://schemas.android.com/apk/res/android"
	xmlns:myapp="http://schemas.android.com/apk/res-auto"
	android:layout_width="match_parent"
	android:layout_height="match_parent"
	android:orientation="vertical">
```

The ```http://schemas.android.com/apk/res-auto``` value will automagically point to the ```attrs.xml``` file we created earlier. Neato. Next, we need to set the attributes 
on the MyCheckBox View:

```xml
<com.myapp.MyCheckBox
	android:layout_width="match_parent"
	android:layout_height="wrap_content"
	android:id="@+id/chk_customCheckBox"
	myapp:titleText="My custom title text"
	myapp:detailText="My custom detail text"
	myapp:checked="true"/>
```

The completed layout looks like this:

```xml
<LinearLayout
	xmlns:android="http://schemas.android.com/apk/res/android"
	xmlns:myapp="http://schemas.android.com/apk/res-auto"
	android:layout_width="match_parent"
	android:layout_height="match_parent"
	android:orientation="vertical">

	<com.myapp.MyCheckBox
		android:layout_width="match_parent"
		android:layout_height="wrap_content"
		android:id="@+id/chk_customCheckBox"
		myapp:titleText="My custom title text"
		myapp:detailText="My custom detail text"
		myapp:checked="true"/>

</LinearLayout>
```

### Reading and setting the attributes on the custom component

The last step is actually using the attributes we set in xml. Android isn't quite smart enough to do this on its own, so we need to help it out. In the MyCheckBox class, 
you may have noticed that two of the constructors take an ```AttributeSet attrs``` parameter. That parameter actually contains all of the attributes that we set in the xml. 
Our job is to parse those attributes, read their values, and assign them appropriately. We can do this in our ```init``` method:

```java
private void init(Context context, AttributeSet attrs) {
	View.inflate(context, R.layout.my_checkbox, this);
    setDescendantFocusability(FOCUS_BLOCK_DESCENDANTS);
    title = (TextView) findViewById(R.id.txt_checkBox_title);
    detail = (TextView) findViewById(R.id.txt_checkBox_detail);
    checkBox = (CheckBox) findViewById(R.id.chk_checkBox);
    setOnTouchListener(new OnTouchListener() {
        @Override
        public boolean onTouch(View view, MotionEvent motionEvent) {
            checkBox.dispatchTouchEvent(motionEvent);
            return true;
        }
    });

	// Assign custom attributes
	if (attrs != null) {
		TypedArray a = context.getTheme().obtainStyledAttributes(
            attrs,
            R.styleable.MyCheckBox,
            0, 0);

        String titleText = "";
        String detailText = "";
        boolean checked = false;

        try {
            titleText = a.getString(R.styleable.MyCheckBox_titleText);
            detailText = a.getString(R.styleable.MyCheckBox_detailtext);
            checked = a.getBoolean(R.styleable.MyCheckBox_checked, false);
        } catch (Exception e) {
        	Log.e("MyCheckBox", "There was an error loading attributes.");
       	} finally {
            a.recycle();
        }

        setTitleText(titleText);
        setDetailText(detailText);
        setChecked(checked);
	}
}
```

This is a little more complicated, so let's step through it.

First, we obtain a TypedArray containing the attributes that were passed into the View that belong to the MyCheckBox attribute set:

```java
TypedArray a = context.getTheme().obtainStyledAttributes(
	attrs,
	R.styleable.MyCheckBox,
	0, 0);
```

Next, we need to pull the values set on any of our custom attributes, and recycle the TypedArray when done:

```java
String titleText = "";
String detailText = "";
boolean checked = false;

try {
	titleText = a.getString(R.styleable.MyCheckBox_titleText);
	detailText = a.getString(R.styleable.MyCheckBox_detailtext);
	checked = a.getBoolean(R.styleable.MyCheckBox_checked, false);
} catch (Exception e) {
	Log.e("MyCheckBox", "Error loading custom attributes!");
} finally {
	a.recycle();
}
```

Finally, we assign the values from the attributes to our custom View:

```java
setTitleText(titleText);
setDetailText(detailText);
setChecked(checked);
```

And there you have it, your very own custom CheckBox. You can extend this even further and include attributes for enabling the View, setting a style on the two TextViews, 
or setting the drawable for the CheckBox. The MyCheckBox class could also intelligently handle hiding/showing the TextViews depending on if their text is empty or not. 
That's the beauty of custom Views: you can make them work however you want to suit your needs.
