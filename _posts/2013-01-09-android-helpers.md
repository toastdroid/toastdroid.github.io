---
layout: post
title: "Android Helpers"
author: Fred Medlin
categories: [android, java]
---

Java helpers are simply classes that work can be delegated to. There is no difference in Android applications. Keeping helper classes small, tight and focused can improve your app. Here are some advantages:

* Enforce framework requirements in a concise way.
* Wrap platform differences
* Keep your code DRY

First things first. Create a class with a private constructor. That will prevent accidental instantiation of the helper class. For our purposes, the helper class will simply be a collection of public static methods.<!--more-->

```java
public class ExternalStorageHelper {
	private ExternalStorageHelper() {}
}
```

### Enforce framework requirements

Let's use Android's [external storage](https://developer.android.com/guide/topics/data/data-storage.html#filesExternal) as an example for building a helper class. Before working with external storage, [getExternalStorageState()](https://developer.android.com/reference/android/os/Environment.html#getExternalStorageState(\)) should be called to check it's availability. The call returns a String and checking the value can be a little ceremonial. It might be cleaner just to ask the boolean question you care about.

The method isExternalStorageMounted() shows that you can ask that simple question to test for the required availability.

```java
public static boolean isExternalStorageMounted() {
	return Environment.getExternalStorageState().equals(Environment.MEDIA_MOUNTED);
}
```

In a [recent tweet](https://twitter.com/romainguy/status/282975428136824832), Romain Guy reminded developers to put data in the right place on external storage. Your app might benefit from the media scanner by placing files in the correct location. A helper method can enforce this and you only have to remember the details once. Here, music files can be directed to appropriate directory.

```java
public static File openMusicDirectory(Context context) {
	if (isExternalStorageMounted()) {
		return (Build.VERSION.SDK_INT >= Build.VERSION_CODES.FROYO) ?
			context.getExternalFilesDir(Environment.DIRECTORY_MUSIC) :
			openDirectory("music");
	}
	return null;
}

private static File openDirectory(String dirname) {
	File f = null;
	if (isExternalStorageMounted()) {
		File storageDir = Environment.getExternalStorageDirectory();
		f = new File(storageDir, dirname);
		if (f != null && !f.exists()) {
			f.mkdirs();
		}
	}
	return f;
}
```

### Wrap platform differences

Prior to API 8, files were written using [getExternalStorageDirectory()](https://developer.android.com/reference/android/os/Environment.html#getExternalStorageDirectory(\)). Beginning with API 8, [getExternalFilesDir()](https://developer.android.com/reference/android/content/Context.html#getExternalFilesDir(java.lang.String\)) is used with a type parameter to enable the media scanner to properly categorize files.

You can make your application's files available for sharing. In openPublicPicturesDirectory(), we use the required directory to share images that will not be deleted with the application. Despite the runtime API level, the files will be saved in a sharable, public directory.

```java
public static File openPublicPicturesDirectory() {
	if (isExternalStorageMounted()) {
		return (Build.VERSION.SDK_INT >= Build.VERSION_CODES.FROYO) ?
			Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES) :
			openPublicDirectory("Pictures");
	}
	return null;
}

private static File openPublicDirectory(String dirname) {
	File f = null;
	if (isExternalStorageMounted()) {
		File storageDir = Environment.getExternalStorageDirectory();
		f = new File(storageDir, dirname);
		if (f != null && !f.exists()) {
			f.mkdirs();
		}
	}
	return f;
}
```

Another example, getCacheDir(), also shows this ability to wrap platform differences. The cache directory is a useful location to temporarily persist files. If external storage is not mounted, we fallback to the device internal memory using [Context.getCacheDir()](https://developer.android.com/reference/android/content/Context.html#getCacheDir(java.lang.String\)).

```java
public static File getCacheDir(Context context) {
	return isExternalStorageMounted() ? getExternalCacheDir(context) : getCacheDirByContext(context);
}

private static File getExternalCacheDir(Context context) {
	return (Build.VERSION.SDK_INT >= Build.VERSION_CODES.FROYO) ?
			context.getExternalCacheDir() :
			context.getCacheDir();
}

private static File getCacheDirByContext(Context context) {
	return context.getCacheDir();
}
```

If external storage is available, it will use it, provided the device is running API 8 or higher. Otherwise, it will fallback to internal memory as before.

# Keep your code DRY

These are small methods. Even so, copying and pasting little snippets throughout your app is a code smell. The helper is just enough context to wrap reusable functionality.

There are some legit objections to helper functions, especially when they are used as a dumping ground for unrelated functionality. They can resist refactoring since they aren't in front of you all the time like the classes that call them. So keep it small and focus on your application's needs. There's no need to construct a generic library for everything. So, rewrite your helpers for each application if you want. That way you're not locked into a strict library pattern of error or exception handling. Let your app be your guide.

It's a little convenience that keeps your code clean and your head free of low level framework details.
