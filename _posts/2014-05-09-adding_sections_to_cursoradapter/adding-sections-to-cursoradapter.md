---
layout: post
title: "Adding Sections to CursorAdapter"
author: Chris Pierick
categories: [libraries, tools]
tags: [cursoradapter, java, opensource, tutorial]
---

Creating a [`ListAdapter`][ListAdapter] from a [`BaseAdapter`][BaseAdapter] with sections is fairly easy, though it can be non-trival. Here at Two Toasters we often use SQLite databases with a [`CursorAdapter`][CursorAdapter]. [`CursorAdapter`][CursorAdapter] abstracts `getView` from you by having you implement `newView` and `bindView` instead. This is great because then you don’t have manage any of the view recycling yourself and the cursor is given to you already moved to the position for your list item. This is where things get tricky when you want to add sections.

[__`SectionCursorAdapter`__][SectionCursorAdapter] was created to make adding sections to the [`CursorAdapter`][CursorAdapter] with fast scroll trival. It is a scientific fact that it’s 348.265% easier to implement compared to adding sections to a [`BaseAdapter`][BaseAdapter] on array data(clinical trial data available on request). The question is, is it really that hard to implement your own sectioning [`CursorAdapter`][CursorAdapter]? Also, is this library really that versatile, and what does this give you over creating one yourself?<!--more-->

### How does the SectionCursorAdapter work?

At first your mind might hit the problem like this. `bindView` gives you the cursor already moved to the current list position. So you can more than likely get the position for the view from `cursor.getPosition`. Well then newView more than likely needs a different view so you get the position from your cursor there too. Now for the recycler to work correctly you need to override `getViewPositionType`. Not to bad but you’ll have to handle knowing what are to correct views in both new and bind, and now that you’re adding extra items your count is off so you have to override `getCount`. How do you know how many sections you’ve added before at the current position: You cannot just keep a counter because the user can scroll up and down? On top of that, how are you going to determine where cursor sections start and stop?

[`SectionCursorAdapter`[SectionCursorAdapter] handles all of this for you. It works by overriding `getView` and delegating to the abstract methods `newSectionView`, `bindSectionView`, `newItemView` and `bindItemView`. [`SectionCursorAdapter`][SectionCursorAdapter] does this in a similar way to the [`CursorAdapter`][CursorAdapter] but with section checks.

```java
public View getView(int position, View convertView, ViewGroup parent) {
    boolean isSection = isSection(position);
    Context context = parent.getContext();
    Cursor cursor = getCursor();
    View view;

    if (!isSection) {
        int newPosition = getCursorPositionWithoutSections(position);
        if (!cursor.moveToPosition(newPosition)) {
            throw new IllegalStateException("couldn't move cursor to position " + newPosition);
        }
    }

    if (convertView == null) {
        view = isSection ? newSectionView(context, getItem(position), parent)
                : newItemView(context, cursor, parent);
    } else {
        view = convertView;
    }

    if (isSection) {
        bindSectionView(view, context, position, getItem(position));
    } else {
        bindItemView(view, context, cursor);
    }

    return view;
}
```

It then uses the abstract method `getSectionFromCursor` so that it knows which positions are going to be a section. That information is used to build a TreeMap which is used to normalizes cursor positions against section positions.

```java
protected SortedMap<Integer, Object> buildSections(Cursor cursor) {
    TreeMap<Integer, Object> sections = new TreeMap<Integer, Object>();
    int cursorPosition = 0;
    while (cursor.moveToNext()) {
        Object section = getSectionFromCursor(cursor);
        if (cursor.getPosition() != cursorPosition)
            throw new IllegalStateException("Do no move the cursor's position in getSectionFromCursor.");
        if (!sections.containsValue(section))
            sections.put(cursorPosition + sections.size(), section);
        cursorPosition++;
    }
    return sections;
}
```

The adapter then makes sure to rebuild this information on `notifyDataSetChanged` and `notifyDataSetInvalidated`. It also nicely handles `getItem`, `getItemId`, `getItemViewType`, `getViewTypeCount` and `getCount`. On top of all of that, [`SectionCursorAdapter`][SectionCursorAdapter] uses this information to implement [`SectionIndexer`][SectionIndexer], giving you fast scrolling dialogs for free.

## How can they implement it?

### Building Sections

If you have used [`CursorAdapter`][CursorAdapter] before then the [`SectionCursorAdapter`][SectionCursorAdapter] is dead simple. To reiterate from above the `newView` and `bindView` have been removed and replaced with `newSectionView`, `bindSectionView`, `newItemView` and `bindItemView`. You can use `getLayoutInflater` to help inflate your new views. The main difference is the abstract method `getSectionFromCursor`. If your sections are just a number or a String then all you have to do is get it from your cursor and return it. Before returning it you can manipulate it however you want or you can return an object where you’ve overriden the `toString` method. The following example builds sections by job title.

```java
@Override
protected Object getSectionFromCursor(Cursor cursor) {
    int columnIndex = cursor.getColumnIndex(ToasterModel.SHORT_JOB);
    return cursor.getString(columnIndex);
}
```

__Noob tip__: you have to sort your cursor when you query.

### FastScroll, it's free!

```java
listView.setFastScrollEnabled(true);
```

or

```xml
<ListView ... 
    android:fastScrollEnabled="true" />
```

## The Power!

Let’s say though you want to do something a little more complicated. You want to make a custom adapter once but you might have a lot of different ways you want to build sections. You can then pass in your own map at the same time that you pass in the cursor so you can easily change up the way sections are being displayed.

```java
SortedMap<Integer, Object> mSectionsMap;
 
public MyAdapter(Context context) {
    // I recommend using LoaderManager so I'm passing in 0 to not use the data observer.
    super(context, null, 0);
}
 
public void setData(Cursor cursor,SortedMap<Integer, Object> sectionsMap) {
    // Order is important as swapCursor will end up calling buildSections.
    mSectionsMap = sectionsMap;
    swapCursor(cursor);
}
 
@Override
protected SortedMap<Integer, Object> buildSections(Cursor cursor) {
    return sectionsMap;
}
 
@Override
protected Object getSectionFromCursor(Cursor cursor) {
    throw new IllegalStateException("getSectionFromCursor should not being called in this adapter as buildSections is overriden.");
}
```

You could even pass in different maps with different object types for values and just override the `toString` methods on those objects.

## Conclusions

Making a [__`CursorAdapter`__][CursorAdapter] with sections is fairly complicated but using [__`SectionCursorAdapter`__][SectionCursorAdapter] is not. It’s easy to implement, versatile and powerful. The github README also has great information on implementation. Make sure to star the [__github project__][SectionCursorAdapter] and don’t forget to tell your friends!


[CursorAdapter]: http://developer.android.com/reference/android/support/v4/widget/CursorAdapter.html
[BaseAdapter]: http://developer.android.com/reference/android/widget/BaseAdapter.html
[ListAdapter]: http://developer.android.com/reference/android/widget/ListAdapter.html
[SectionCursorAdapter]: https://github.com/twotoasters/SectionCursorAdapter
[SectionIndexer]: http://developer.android.com/reference/android/widget/SectionIndexer.html

