---
layout: post
title: "On-Screen Keyboard Tracking, Part 2"
author: Ben Berry
categories: [development]
tags: [keyboard]
---

A few months ago, we discussed [the basic steps you can use to keep track of whether a screen in your app has the soft keyboard open](/2014/10/14/on-screen-keyboard-state-tracking-in-3-easy-steps). 

At Ticketmaster Mobile Studio, we've found this especially important and useful on a Fragment or Activity that has one single EditText on it and a bunch of other Views. Whether the rest of the screen is a camera preview or a complex RecyclerView, managing that one EditText can become a problem. Today’s post will help you control whether the EditText has focus and, perhaps more importantly, an annoying blinking cursor, even when the soft keyboard is closed.
<!--more-->

To understand the blinking cursor, first a quick review of the rules for focus in Android. When an activity starts, the first *focusable* view is given focus. Views can `requestFocus()` which does exactly what you'd think. They can also `clearFocus()` which abdicates focus back to the first focusable view in the View hierarchy. If the View that just tried to clear focus happens to be the only focusable view in the hierarchy? It gets unfocused and then immediately focused again. Unless we intervene.

In this example, we'll start with a screen with one EditText that's initially not focused and the keyboard is closed. If the user wants to type some text, they'll tap the EditText, and it'll gain focus. When they close the keyboard by pressing back or "Submit", the EditText will make itself unfocusable and then `clearFocus()` leaving the View Hierarchy with no focusable Views.

To do this, we'll create a new EditText similar to last time's [BackAwareEditText](/2014/10/14/on-screen-keyboard-state-tracking-in-3-easy-steps), a View suitable for being the only EditText on the screen:

```java
public class LonelyEditText extends EditText {
    @Override
    public boolean performClick() {
        setFocusableInTouchMode(true);
        requestFocus();

        return super.performClick();
    }

    @Override
    public boolean onKeyPreIme(int keyCode, KeyEvent event) {
        if (event.getKeyCode() == KeyEvent.KEYCODE_BACK && event.getAction() == KeyEvent.ACTION_UP) {
            clearFocus();
        }

        return false;
    }

    @Override
    protected void onFocusChanged(boolean focused, int direction, Rect previouslyFocusedRect) {
        super.onFocusChanged(focused, direction, previouslyFocusedRect);

        InputMethodManager inputMethodManager = (InputMethodManager) getContext()
                .getSystemService(Context.INPUT_METHOD_SERVICE);

        if (focused) {
            inputMethodManager.showSoftInput(this, 0);
        } else {
            setFocusableInTouchMode(false);
            inputMethodManager.hideSoftInputFromWindow(getWindowToken(), 0);
        }
    }

    /* constructors */
}
```

This class has four major chunks:

* `performClick`: When the View is clicked, make it focusable and then request focus.
* `onKeyPreIme`: When the user presses the back button in the system bar, clear focus.
* `onFocusChanged(focused == true)`: When the EditText gains focus, show the keyboard.
* `onFocusChanged(focused == false)`: When the EditText loses focus, make it unfocusable to stop it from boomeranging right back, and then close the keyboard. 

If we drop this in to our previous activity with a Submit button that just calls `clearFocus()` on our LonelyEditText, we get something like this: 

[/assets/2015-06-04-on-screen-keyboard-tracking-part-2/keyboard2.gif]

Blinking cursor when we want it, none when we don't.

Now that our one LonelyEditText is managing the keyboard, we can do other things easily by giving or clearing focus. Have a ListView or RecyclerView that you'd like to have close the keyboard when the user starts scrolling? Add a scroll listener that calls `clearFocus()` on the EditText.

Want to have the keyboard open when the activity starts? Have the EditText `requestFocus()`.

Want to only show the "Submit" or "Next" button once the user has opened the keyboard? Add an `onFocusChangedListener`.

Now, instead of statefully tracking whether the keyboard is open or closed, you can just react (via focus change events) to it opening and closing, a much more event-driven, Android-y way of doing things. 

