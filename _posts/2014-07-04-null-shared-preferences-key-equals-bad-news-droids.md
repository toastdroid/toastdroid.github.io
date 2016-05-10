---
layout: post
title: "Null Shared Preference Key == Bad News Droids"
author: Chris Pierick
categories: [tips, development]
tags: [shared preference, bug, android, java]
---

There is a little [known bug][1] in Android’s SharedPreference that will have devastating effects on your app. This revolves around what happens when you try to save a null key into SharedPreferences. The first question is:

## Why would you have a null key?

This is a good question. In most cases, if you have "good" code, you’ll never run into this. Let me tell you how I ran into this problem.<!--more-->

In my case I was working with an ever changing api. My response returned an array of objects, each with an id. As these objects could be changed on the website, when I received them from the api I wanted to update what was already stored in the SharedPreferences.

```java
public void saveResponse(List<Response> memeList) {
    for (Response response : memeList) {
        String key = null;
        switch (response.getId()) {
            case DOGE: key = KEY_DOGE;
            case MUCH_WOW: key = KEY_MUCH_WOW;
            case SUCH_DROID: key = KEY_SUCH_DROID;
        }
        editor.putString(key, response.getMeme());
    }
    editor.commit();
}
```
    

This looks pretty good, though there is the possibility of a null key if more memes are added in the future. My thought was, I could bulk up my code by putting a null guard on `editor.putString` or I could see if SharedPreferences has a problem with a null key. The documentation does not address this question so I ran the above code with `case SUCH_DROID: key = null;` to see what would happen.

When the app runs the code in question, there is no error around the api logs. So far so good. Then going to another Activity, which uses this information, everything loads as expected. It would seem that we’re golden, with no need for another useless null check just cluttering up the code. Commit, Pull Request and Merge.

## BOOM your preferences just got Toasted!

A month later you start noticing that you’re randomly getting signed out of your app. An added object to the api response and the null key which it is creating is causing this but correlating these things together may take you a long time. Here is what is happening.

1.  `editor.putString(null, “Wow much Droid!”).commit();` gets called.
2.  `<string>Wow much Droid!</string>` is then saved into the SharedPreferences’ xml file. 
    *   Compare this to a correct entry `<string name=”wowDroid”>Wow much Droid!</string>`
3.  The app is closed, either by the system killing it, the user killing it from multi-tasking or the device getting restarted.
4.  When the app is started up again SharedPreferences will try to load the now corrupt SharedPreferences’ xml file.
5.  The log will display a warning, which in my opinion is barely noticeable. `W/SharedPreferencesImpl﹕ getSharedPreferences
org.xmlpull.v1.XmlPullParserException: Map value without name attribute: string`.
6.  SharedPreferences will now delete your SharedPreferences’ xml file and create a new one.

That’s right, if SharedPreferences runs into one XML tag it doesn’t like it will delete the whole file but only in the app’s `onCreate` and with a warning that says nothing about deleting a file.

## Conclusion

Luckily for us, our app had not been published before we ran into this error. Honestly SharedPreferences should never be putting bad entries into the xml file in the first place. It should also never blow away the file just because one entry doesn’t have a required attribute.

Is this likely to happen? No not very. Can you stop it by putting in a null guard? Absolutely. Will this have devastating effects on a production app and be incredibly hard to track down? You got it dude!

 [1]: https://code.google.com/p/android/issues/detail?id=63463
