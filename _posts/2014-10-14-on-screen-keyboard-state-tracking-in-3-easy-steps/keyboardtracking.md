---
layout: post
title: "On-Screen Keyboard State Tracking in 3 Easy Steps"
author: Ben Berry
categories: [development]
tags: [keyboard]
---

If you've never been frustrated by the inability in Android to know whether or not the on-screen keyboard is open, this article probably isn't for you. Even if you have hit that roadblock in the past, it's very possible that Android was trying to save you from a bad design decision. But if you really want to know how to deal with whether or not the soft keyboard is open, read on.<!--more-->

So everyone still here knows that Android doesn't provide any method for determining if the soft keyboard is currently open, which can mean only one thing: we have to do it ourselves.

### Step 1: The Flag
In order to track the state of the on-screen keyboard, first we need a boolean in our activity:

```java
public class MainActivity extends Activity {
	private boolean isKeyboardOpen = false;
```

Tough stuff, right? But seriously, to make sure that by default the keyboard is closed in your activity, make sure to set `android:windowSoftInputMode="stateHidden"` in the AndroidManifest.xml entry for your activity.

### Step 2: Listening for Opening
The next part is pretty straightforward as well: detecting when the keyboard is opened. This pretty much only happens one way, which is an EditText getting clicked on. In our sample app, we just have one EditText (which receives focus by default since it is the first visible, editable EditText on the screen), so we'll listen for it being clicked:

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    BackAwareEditText editText = (BackAwareEditText) findViewById(R.id.edit_text);
    editText.setOnClickListener(new OnClickListener() {
        @Override
        public void onClick(View view) {
            isKeyboardOpen = true;
        }
    });
```

What's that `BackAwareEditText` all about? Well that's needed for...

### Step 3: Listening for Closing
Now comes the hard part. There is no easy way to hook into the input method event for the user pressing the back button to close the keyboard. To get that, we have to extend EditText and expose a new callback.

```java
public class BackAwareEditText extends android.widget.EditText {

    private BackPressedListener mOnImeBack;

    /* constructors */

    @Override
    public boolean onKeyPreIme(int keyCode, KeyEvent event) {
        if (event.getKeyCode() == KeyEvent.KEYCODE_BACK && event.getAction() == KeyEvent.ACTION_UP) {
            if (mOnImeBack != null) mOnImeBack.onImeBack(this);
        }
        return super.dispatchKeyEvent(event);
    }

    public void setBackPressedListener(BackPressedListener listener) {
        mOnImeBack = listener;
    }

    public interface BackPressedListener {
        void onImeBack(BackAwareEditText editText);
    }
}
```

This is the key that makes everything else easy. Back in our MainActivity, we can just bind to the new callback and detect when the keyboard is closed, updating our internal state tracking boolean accordingly: 

```java
editText.setBackPressedListener(new BackPressedListener() {
    @Override
    public void onImeBack(EditTextBackEvent ctrl, String text) {
        isKeyboardOpen = false;
    }
});
```

### Extra Credit: Hooking in to the Activity Lifecycle
If the user presses the back button to exit your activity, that'll cause the BackPressedListener to fire, closing the keyboard and updating the internal state. But if the user exits the activity another way, like pressing the Home button, the keyboard will be closed without updating our variable. It's annoying, but it's all a part of the manual bookkeeping we have to do on the keyboard's state.

Luckily, the solution is pretty easy:
	
```java
@Override
protected void onPause() {
    super.onPause();
    isKeyboardOpen = false;
}
```

And that's it. Now, whenever you want to check if the keyboard is on-screen, just poll your variable. In the sample app ([source](https://github.com/twotoasters/toastdroid/tree/master/ben/SimpleKeyboardTracker) and [apk](https://github.com/twotoasters/toastdroid/raw/master/ben/SimpleKeyboardTracker/app/simple-keyboard-tracker.apk) available), we use it to display a crouton saying whether or not the keyboard is open.

![Demo of the keyboard tracker](https://github.com/twotoasters/toastdroid/raw/master/ben/SimpleKeyboardTracker/demo.gif)

This is just a quick rundown of the bare minimum it takes, but there are some ways you can use keyboard tracking to take your user experience to the next level, which we'll cover in part 2.