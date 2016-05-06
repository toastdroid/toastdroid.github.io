---
layout: post
title: "What is data binding and what does it solve?"
author: Adam Shea
categories: [development]
tags: [data binding, support library]
---

Data binding allows us to remove all the boilerplate `findViewById()` calls as well as having to manually update those views in the code. 
With Android now officially supporting data binding it opens the doors for a [MVVM][1] architecture. MVVM stands for Model-View-ViewModel. With MVVM the ViewModel and the View are really de-coupled making it easy to test the ViewModel without even needing a View. Since the View updates its components via a data binder there is no need for the ViewModel to be dependent on it. The ViewModel can simply update the Model and whichever View is currently binding to that Model will also update.
<!--more-->

## MVVM Architecture example
![](http://i.imgur.com/AryUZmj.jpg)

## Requirements
To get started you need to be using `'com.android.tools.build:gradle:1.3.0-beta1'` or higher inside your project's `build.gradle` file. Inside that same file you also need to add `'com.android.databinding:dataBinder:1.0-rc0'` so that it looks something like this:

```groovy
dependencies {
    classpath 'com.android.tools.build:gradle:1.3.0-beta1'
    classpath 'com.android.databinding:dataBinder:1.0-rc0'
}
```

The last step is to add `apply plugin: 'com.android.databinding'` to your application's build.gradle file.

You can find more specific information on how to set up data binding [here.][2]

## Solving real world problems 

An interesting use we found for data binding here at Ticketmaster Mobile Studio is to create a countdown timer that will span across multiple Activities or Fragments. With data binding this problem proved to be trivial. We needed to have our model (in this case a timer) attach to whatever view is currently being displayed. The first thing to do is create the CountdownViewModel class. This singleton class contains our model, which will just be a `long` of the current time left, and will modify it when the timer is started and during each tick. 

```java
public class CountdownViewModel extends BaseObservable {
    private static CountdownViewModel instance;
    private CountDownTimer countDownTimer;
    private long timeLeftMillis;

    public static CountdownViewModel instance() {
        if (instance == null) {
            instance = new CountdownViewModel();
        }
        return instance;
    }
}  
```

Notice how our ViewModel class extends `BaseObservable`. Subclassing `BaseObservable` allows us to use `@Bindable` on getters, and `notifyPropertyChanged()` when a `@Bindable` property changes.

```java
@Bindable
public long getTimeLeftMillis() {
    return timeLeftMillis;
}

public void setTimeLeftMillis(long timeLeftMillis) {
    timeLeftMillis = timeLeftMillis;
    notifyPropertyChanged(timeLeftMillis);
}
```

The BR class file is generated in the module package and is similar to the R class file. It will contain a list of properties that are `@Bindable` so you can call `notifyPropertyChanged()` on it. In this case `timeLeftMillis` corresponds to the field that changed and we notify any layout that is binded to that property to update itself by calling the getter associated with that ID. The BR file also contains a BR._all property that will update all the properties at once.

Now that we sort of know what our ModelView and Model will look like we can create our XML layout.

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools">
    <data class="CountdownBinder">
        <variable name="countdownViewModel" type="your.package.name.CountdownViewModel"/>        
    </data>

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/timeLeftMillis"
            android:layout_gravity="right"
            android:textSize="45dp"
            android:text="@{String.valueOf(countdownViewModel.timeLeftMillis)}"
            android:layout_centerVertical="true"
            android:layout_centerHorizontal="true" />

        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Restart"
            android:id="@+id/restartButton"
            android:layout_below="@+id/button"
            android:layout_centerHorizontal="true"/>

        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Next Activity"
            android:layout_below="@id/restartButton"
            android:id="@+id/nextButton"
            android:layout_centerHorizontal="true"/>

    </RelativeLayout>
</layout>
```

The magic happens with the `<layout>` tag that wraps the `<data>` element as well as the root view (and subsequent views) of our layout. In the `<data>` section there is an optional class attribute where you can define what the generated binder class will be called. If the class attribute isn't specified, this binding class is generated based on the name of the layout file, without underscores and capitalizing the following letter and then suffixing "Binding". The CountdownViewModel variable within `<data>` describes a property that may be used within this layout. We specify a name and the type that this variable will represent. Further down in the layout we reference CountdownViewModel's timeLeftMillis variable. If this variable is declared public inside CountdownViewModels then we don’t need to provide a getter. However if it is a private variable then the layout will attempt to call the getter in our ViewModel class. Since this field is declared as a long in our ViewModel we can’t just use CountdownViewModel.timeLeftMillis. If you try to do this, the layout and BR files will not compile. Just like in code you can use functions like `String.valueOf()` to convert non-String values to a String.

At this point our Model, ViewModel, and View are taking form and now we need to bind them all together. Modify the CountdownViewModel class by adding some methods to modify the timer.

```java
public void startTimer(long timeLeftMillis) {
    if (!isRunning) {
        isRunning = true;
        countDownTimer = new CountDownTimer(timeLeftMillis, 1000) {
            @Override
            public void onTick(long millisUntilFinished) {
                setTimeLeftMillis(millisUntilFinished / 1000);
            }

            @Override
            public void onFinish() {
                isRunning = false;
                setTimeLeftMillis(0);
            }
        };
        countDownTimer.start();
    } else {
    Log.i(TAG, "Timer already started, call restart first");
    }
}

public void cancelTimer() {
    if (isRunning()) {
        countDownTimer.cancel();
        isRunning = false;
    }
}

public void restartTimer() {
    cancelTimer();
    startTimer(60000);
}
```


This function uses an Android CountDownTimer class and when it is started every second that goes by we update our model by setting the appropriate time left.

Create a MainActivity class that will be our launcher activity for our application and will act as our View.

```java
public class MainActivity extends AppCompatActivity {

    CountdownBinder binder;
    CountdownViewModel countdownViewModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binder = DataBindingUtil.setContentView(this, R.layout.activity_countdown);
        countdownViewModel = CountdownViewModel.instance();
        countdownViewModel.startTimer(60000);
        binder.setCountdownViewModel(countdownViewModel);
        binder.restartButton.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                countdownViewModel.restartTimer();
            }
        });
        binder.nextButton.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(DetailActivity.this, SecondActivity.class);
                startActivity(intent);
            }
        });
    }
}
```

Now that the timer is started, when the "Next Activity" button is pressed on the `MainActivity` we want to launch our `SecondActivity` and display that same timer on the screen.

```java
public class SecondActivity extends AppCompatActivity {

    PersistantCountdownActivityBinder binder;
    CountdownViewModel countdownViewModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binder = DataBindingUtil.setContentView(this, R.layout.activity_second);
        countdownViewModel = CountdownViewModel.instance();
        binder.setCountdownViewModel(countdownViewModel);
        binder.restartButton.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                if (countdownViewModel.isRunning()) {
                    countdownViewModel.restartTimer();
                }
                else {
                    countdownViewModel.restartTimer();
                }
            }
        });            
    }
}
```

This activity will get an instance of our CountdownViewModel that contains the running timer and bind that CountdownViewModel instance to the PersistantCountdownActivityBinder, which corresponds to our XML layout file.

`R.layout.activity_second`

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data class="PersistantCountdownActivityBinder">
        <variable name="countdownViewModel" type="com.ticketmaster.mobilestudio.myapplication.blog.CountdownViewModel"/>
    </data>
    <RelativeLayout
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/countDownView"
            android:text='@{String.valueOf(countdownViewModel.timeLeftMillis)}'
            android:textSize="45dp"
            android:layout_centerVertical="true"
            android:layout_centerHorizontal="true" />

        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Restart Timer"
            android:id="@+id/restartButton"
            android:layout_centerHorizontal="true"/>
    </RelativeLayout>
</layout>
```

## Pitfalls
While data binding with this solution can make life easier with some trivial examples like the one we talked about, it's not without its problems.

*   Merging business logic and XML. 

Relating back to our Countdown Timer example, let's say we wanted to hide the restart button when the countdown is running and show it when it is not. Putting these expressive statements in the XML should be avoided.

```xml
<Button
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="Restart"
    android:id="@+id/restartButton" 
    android:layout_below="@+id/button"
    android:layout_centerHorizontal="true"
    android:visibility="@{countdown.timeLeftMillis == 0 ? View.VISIBLE : View.GONE}"/>
```

*   Putting expressions inside the XML layout creates a maintenance nightmare and can make it difficult to test the view. Instead, you can put this logic inside the ViewModel class and when you need to change the visibility make the call to your View class to hide the UI component. This lets you easily test that logic.
*   Putting (sometimes) complex code in XML attributes instead of encapsulating it in code seems clunky, and I'm not sure how well it would handle very complex adapters and views.
*   Currently Android Studio may say, “cannot resolve symbol” when trying to access the BR class file, ignore that and build anyway!
*   When Android Studio says that your binding class doesn't exist (CountdownBinder in our case), it's usually because there is an error in your XML layout file and it can't generate the binding object.
*   Currently in XML the variable type needs to be spelled out clearly (`java.util.List<String>` not `List<String>`)
*   When changing an XML file a rebuild will help with any weird compile errors.
*   Displaying an `int` as text needs `String.valueOf()`, or use a getter in the View that returns a `String`.
*   BR sometimes needs full path instead of just BR.

## Alternative

[Butterknife](http://jakewharton.github.io/butterknife/) v7.0.1 is similar to Android's data binding library, except that there is no code being thrown into the XML layouts and it is all done in Java files. 


[1]: https://en.wikipedia.org/wiki/Model_View_ViewModel
[2]: https://developer.android.com/tools/data-binding/guide.html
