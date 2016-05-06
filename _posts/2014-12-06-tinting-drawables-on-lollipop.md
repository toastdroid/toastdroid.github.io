---
layout: post
title: "Byte of Toast: Tinting Drawables on Lollipop"
author: Chris Pierick
categories: [development, lollipop, byte of toast]
---

One of the cool things about Android 5.0 Lollipop is that you can colorize system UI widgets through the themes and styles.
 While this is nice, it doesn't work for every drawable you provide.
 But thankfully this is also easy.
 First create a png resource, your drawable, which only has transparency and white, this can be a 9-patch.
 Next in `res/drawable` or `res/drawable-v21` make an xml drawable.
 For our example we'll call this drawable ic_back_red.xml.
 In this file the `<bitmap>` tag should be our root and we need to set the `src` and `tint` fields.
 
```xml
<?xml version="1.0" encoding="utf-8"?>
<bitmap
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:src="@drawable/ic_back"
    android:tint="@color/red_tint"/>
```

###### Note: that `tint` can be a color or a color statelist. You can also use the current colorPrimary with `?android:colorPrimary` as your tint.

Boom, your drawable has been tinted.
 You can reference this anywhere drawables can be referenced and it will use the appropriate resource bucket for the original png.
