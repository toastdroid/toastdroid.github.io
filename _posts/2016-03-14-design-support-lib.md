---
  layout: post
  title: "Intro to the Design Support Library - Part 1"
  author: Curtis Martin
  categories: [development]
  tags: [ui, material, lollipop]
---

If you're an Android developer (and you probably are if you're reading this blog), you've probably heard of the Design Support Library. It was [unveiled at Google I/O](http://android-developers.blogspot.com/2015/05/android-design-support-library.html) last year, and comes with several classes to assist you in making your apps follow the [Material Design Guidelines](https://www.google.com/design/spec/material-design/introduction.html). This leads to not only a great user experience, but consistency across apps all over the Android ecosystem. This is something that has been sorely lacking up until the introduction of Lollipop. In this post, I'm going to show you how to use several elements of the Design Support Library in your app, as well as point out a few gotchas I've come across.<!--more-->

Before we begin, I should note that this post is based on version __23.1.0__ of the library, and with Google's constant updates there could be changes down the road that are not reflected here.

### CoordinatorLayout

`CoordinatorLayout` is, in Google's own words, a "super-powered `FrameLayout`". It behaves similarly to a `FrameLayout` in that it uses z-ordering and doesn't give you many methods for laying out child views (margins and layout gravity are about all you get), but it has special functions: it facilitates Material animations and visual flourishes, as well as coordinates the location and behaviors of child views with respect to other children. Most of the views in this library make use of the `CoordinatorLayout` so it is essential that you know how to use it. Fortunately, it's very easy!

Simply put a `CoordinatorLayout` at the root of your `Activity` or `Fragment's` layout xml, and any child views will automatically be able to take advantage of it. It should look something like this:

```xml
<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_height="match_parent"
    android:layout_width="match_parent">

   <!-- 
     Add your layout's views here just like you normally would. Don't forget to add a root 
     LinearLayout, RelativeLayout, etc. if CoordinatorLayout isn't flexible enough for your needs.
   -->

</android.support.design.widget.CoordinatorLayout>
```

That's all there is to it. Now let's take a look at `Snackbar`, which takes advantage of `CoordinatorLayout`.

### Snackbar

You've probably seen `Snackbars` before. They have essentially replaced `Toasts` in Lollipop, and all of Google's apps have been updated to use them.

![Snackbar](/assets/2016-03-14-design-support-lib/snackbar.gif)

`Snackbars` are used to display a brief message to a user and can also take a single input action, such as Undo if you swipe away an email in the Inbox app. Like `Toasts`, they can be shown for a short or long duration, but they can also be shown indefinitely until the user swipes them away. Here's where `CoordinatorLayout` comes in: if you display a `Snackbar` inside one, you get swipe dismissal for free!

You can display a `Snackbar` in a couple lines of code:

```java
CoordinatorLayout root = (CoordinatorLayout) findViewById(R.id.root);
Snackbar.make(root, R.string.txt_snackbar, Snackbar.LENGTH_LONG).show();
```

It's important to be aware that the view passed in as the first parameter of `Snackbar.make()` should either be a `CoordinatorLayout` or be a child of one, otherwise swiping will not work. If you want to also display an action on the right side, you can call `setAction(int resId, View.OnClickListener listener)` to indicate the action's text and set a click listener to handle the action.

### TextInputLayout

The concept of the "floating label" `EditText` became pretty popular after the announcement of Lollipop in 2014. In response, Google has added `TextInputLayout`. This widget has the magical power of adding floating label functionality to any normal `EditText` inside it. You may be asking, "What does floating label mean?" Here's an example:

![TextInputLayout](/assets/2016-03-14-design-support-lib/textinputlayout.gif)

To take advantage of this effect, just wrap your `EditText` with a `TextInputLayout` like so:

```xml
<android.support.design.widget.TextInputLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:hintTextAppearance="@style/TextInputWidgetAppearance">

    <EditText
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="@string/txt_hint_input_1"/>

</android.support.design.widget.TextInputLayout>
```

When an `EditText` that is inside a `TextInputLayout` receives focus, the hint text will automatically shrink and float up above the field. That hint text will stay above the field as long as there is text in it, even if focus is changed to another view. However, if the field is empty and focus changes to, say, a different `EditText`, the hint text will grow and move back into its origin position inside the field.

Additionally, `TextInputLayout` comes with a `setError(CharSequence error)` method. Calling this and passing in some text will cause the text to appear below the `EditText` in red (by default). It's an easy way to show a validation error if a user provides invalid input in your field. To hide the error, just call `setError(null)` and the text will be cleared. Here's an example:

```java
Resources resources = getResources();
TextInputLayout textInput = (TextInputLayout) findViewById(R.id.txt_input_2);
textInput.setError(resources.getString(R.string.txt_input_error));
```

A few extra tips on using `TextInputLayout`:

* Calling `setError()` increases the height of the `TextInputLayout`, so depending on how your layout is structured other views may be shifted when the error is displayed.
* When setting hint text in XML, set it on the `EditText`, not `TextInputLayout`. `android:hint` is accepted by `TextInputLayout` but has no effect. Oddly enough though, calling `setHint()` on the `TextInputLayout` works fine!
* You can use `app:hintTextAppearance` to set the appearance of the floating hint text, and `app:errorTextAppearance` for the error text that appears underneath the field.

![TextInputLayout Error](/assets/2016-03-14-design-support-lib/textinputlayouterror.gif)

### FloatingActionButton (FAB)

If you've used any Google app in the last year or so, you've seen `FloatingActionButton` in use. The FAB is a button that floats on top of an app's content and provides a single action for the user to take, or in Google's words, it is "used for a promoted action". The color of your FAB is dictated by the `android:colorAccent` value in your app's theme.

There are many different use cases for FABs. They can be used to bring up a contextual search, expand into other action buttons to provide quick actions, and morph into new pieces of UI. For some examples of such cases, refer to the [Material Design Guidelines](https://www.google.com/design/spec/components/buttons-floating-action-button.html#buttons-floating-action-button-transitions).

The various uses outlined by the guidelines don't come for free unfortunately; all you get out of the box is a simple round button with a shadow that you can position where needed.

![FloatingActionButton](/assets/2016-03-14-design-support-lib/fab.gif)

To add a FAB, simply put it in your layout xml inside a `CoordinatorLayout`:

```xml
<android.support.design.widget.CoordinatorLayout>

  <!-- Rest of your content -->

  <android.support.design.widget.FloatingActionButton
      android:layout_height="wrap_content"
      android:layout_width="wrap_content"
      android:layout_gravity="right|bottom"
      android:visibility="invisible"
      android:src="@android:drawable/ic_input_add"
      app:fabSize="normal"
      android:id="@+id/fab"/>

</android.support.design.widget.CoordinatorLayout>
```

Remember earlier how `Snackbars` had extra functionality as a result of being inside a `CoordinatorLayout`? The same goes for FABs too! If you show a `Snackbar` and your FAB is anchored to the bottom of the screen (as seen above by `layout_gravity="right|bottom"`), the FAB will automatically translate up and down when the `Snackbar` is shown and hidden, with no extra work required. Neat!

![FAB with Snackbar](/assets/2016-03-14-design-support-lib/fab-snackbar.gif)

The Material Design Guidelines include several animations for FABs to add that Material flourish to your app. One of them is a grow/shrink animation when a FAB is shown/hidden. The `FloatingActionButton` class comes with `show()` and `hide()` methods that will automatically perform these animations, and it works whether your FAB is inside a `CoordinatorLayout` or not! There's also an `isShown()` method that will tell you if the FAB is already being shown. Here's a basic example of how to toggle a FAB's visibility, with animation:

```java
private void toggleFab() {
  FloatingActionButton fab = (FloatingActionButton) findViewById(R.id.fab);
  if (fab.isShown()) {
      fab.hide();
  } else {
      fab.show();
  }
}
```

`FloatingActionButtons` are an important component of Android apps now, and there's way more to cover than what this post will allow for. To get some ideas on how to incorporate them into your own apps, I'd recommend looking at the Design Guidelines linked earlier, as well as Google's apps like Keep, Inbox, Gmail, and Hangouts.

That's all for Part 1. Next time we'll have Part 2, which will detail the rest of the Design Support Library, including `AppBarLayout`, `TabLayout`, and `NavigationView`. In the meantime, feel free to check out [this sample project on Github](https://github.com/curtinmartis/Design-Support-Demo). It demonstrates example usage of each element of the library, including the ones not covered in Part 1.
