---
layout: post
title: "Smooth ActionBar Transitions when Opening a Navigation Drawer"
author: Ben Berry
categories: [development]
tags: [android, java, animation]
---

Have you ever wanted your Action Bar to animate and change as smoothly as your Navigation Drawer when the drawer opens and closes? Here at Two Toasters, this is exactly the kind of little touch we pay attention to. So when we come across this kind of janky transition, we fix it:

![What it looks like before](/assets/2014-04-25-smooth_transition_with_nav_drawer/animation_before.gif)<!--more-->

Notice how the Action Bar only changes its text and button after the drawer has finished animating? Yeah, that has got to go. What if we could get them to animate and change at the same time?

Luckily, that’s not just possible, but pretty easy. Let’s get started.

####A Brief Review of Navigation Drawers
For those of you who haven’t looked under the hood of one recently, you use a Navigation Drawer by having an Activity with a root View of DrawerLayout. In the `onCreate()` method of your Activity’s class, you can find this DrawerLayout view by ID, and then assign it a new ActionBarDrawerToggle, a callback handler that is responsible for updating the Action Bar as the drawer slides in and out.

ActionBarDrawerToggle implements three callbacks:
 * `onDrawerOpened`: Called when the drawer finishes opening
 * `onDrawerClose`: Called when the drawer finishes closing
 * `onDrawerSlide`: Called repeatedly as the drawer incrementally slides open and closed.

The Activity class also implements a few callbacks related to the Action Bar, aka the Options Menu, including the one responsible for defining what is and isn’t visible each time the Action Bar is drawn: `onPrepareOptionsMenu`. To force the Action Bar to be redrawn, we have to call `invalidateOptionsMenu()`.

For an excellent example of this, [check out Google’s sample app](http://developer.android.com/training/implementing-navigation/nav-drawer.html), which I used for the examples in this article.

####The Old and Busted Way
So let’s take a closer look at what’s going on in the situation above. The problem is that all the work of invalidating and redrawing the Action Bar is being done in `onDrawerOpened` and `onDrawerClosed` in our ActionBarDrawerToggle, which only fire once the animation finishes.

Instead, let’s take advantage of `onDrawerSlide` to pick up on the fact that we’re opening or closing the drawer while it’s happening. Try adding this `onDrawerSlide` to your ActionBarDrawerToggle. (This is a naive implementation, but it conveys the core of the change.)

```java
public void onDrawerSlide(View drawerView, float slideOffset){
    if(slideOffset > .5){
        onDrawerOpened(drawerView);
    } else {
        onDrawerClosed(drawerView);
    }
}
```

Despite the fact that this will call `onDrawer*` at least 20 times every time you open or close the drawer, it runs smoothly. However, when opening the drawer, the text changes halfway through the animation, but the icon still doesn’t change until the drawer is fully open.

This is because the icon’s visibility is updated when the drawer reports that it is open, which only happens when the animation finishes (via the `isDrawerOpen` method of DrawerLayout).

####The New Solution
So if we add a private boolean flag to tell us whether the drawer is opening or closing, and check it from onPrepareOptionsMenu (instead of `isDrawerOpen()`), the changes look something like this:

```java
public class MainActivity extends Activity {
    . . .
    private boolean isDrawerOpen = false;

    @Override
    protected void onCreate(Bundle savedInstanceState) {

        . . .

        mDrawerToggle = new ActionBarDrawerToggle(...) {

            . . .

            public void onDrawerSlide(View drawerView, float slideOffset) {
                if(slideOffset > .55 && !isDrawerOpen){
                    onDrawerOpened(drawerView);
                    isDrawerOpen = true;
                } else if(slideOffset < .45 && isDrawerOpen) {
                    onDrawerClosed(drawerView);
                    isDrawerOpen = false;
                }
            }
        };
        . . .
    }

    @Override
    public boolean onPrepareOptionsMenu(Menu menu) {
        // If the nav drawer is open, hide action items related to the content view
        menu.findItem(R.id.action_websearch).setVisible(!isDrawerOpen);
        return super.onPrepareOptionsMenu(menu);
    }
}
```

And this is the result, slowly dragging the menu in to illustrate (the same holds true for tapping Up to toggle the menu):

![The new smoothness](/assets/2014-04-25-smooth_transition_with_nav_drawer/animation_after.gif)

This design still delegates to the `onDrawerOpened` and `onDrawerClosed` methods because they provide semantic value in saying what they do; inlining them in to onDrawerSlide would be straightforward.

And there you have it: a simple, easy way to have Action Bar transitions that are as smooth as your Navigation Drawer.
