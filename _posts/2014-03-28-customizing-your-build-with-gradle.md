---
layout: post
title: "Customizing Your Build With Gradle"
author: James Barr
categories: [development, tools]
tags: [android, opensource, tools]
---

If you haven’t already heard, Gradle is the new build system for Android apps.  Android Studio, the new development IDE for Android apps, leverages Gradle for doing the actual building of the app.  The great thing about Gradle is that it is very customizable.  You could even setup your Gradle build configuration such that you would not have to alter your Eclipse ADT project structure. So basically, there is no reason not to go ahead and try out Gradle right now. And when you do, here are some cool things that you can try out...<!--more-->

## Parsing the Android manifest file

Since the configuration build file is backed by a Groovy-based domain specific language, the build script can actually contain code.  With this method defined below, you can read in your `AndroidManifest.xml` file and parse an attribute, such as versionName, from the file so that you do not have to manage the version name in multiple places.

```groovy
def manifestVersionName() {
    def manifestFile = file(project.projectDir.absolutePath + '/src/main/AndroidManifest.xml')
    def ns = new groovy.xml.Namespace("http://schemas.android.com/apk/res/android", "android")
    def xml = new XmlParser().parse(manifestFile)
    return xml.attributes()[ns.versionName].toString()
}
```

## Versioning build files

We follow the Agile process here, and as such, we create builds for our clients at the end of each sprint.  I like to archive all of the .apk files, but I do not want to worry about what the version name for the file I just built should be.  With the code below, you can configure gradle to automatically add a build suffix, such as the version name from the Android manifest file, to your built .apk file name.

```groovy
project.ext.versionName = manifestVersionName()

android {
    ...
    defaultConfig {
        ...
        applicationVariants.all { variant ->
		    def file = variant.outputFile
            variant.outputFile = new File(file.parent, file.name.replace(".apk", "-" + buildSuffix + ".apk"))
        }
    }
}
```

## Enhancing your BuildConfig

With the Eclipse ADT, when your app was compiled, a BuildConfig class was automatically generated for you.  All you could pretty much use the BuildConfig for was checking whether the build was in Debug mode or not via a boolean flag.  With Gradle, we now have the ability to generate additional fields in the BuildConfig class.  This can be useful for such things as configuring server URLs and easily toggling features on and off.

In your `build.gradle`:

```groovy
android {
    ...
    buildTypes {
        debug {
            buildConfigField "boolean", "REPORT_CRASHES", "true"
        }
        ...
    }    
}
```

In your java code:

```java
if (BuildConfig.REPORT_CRASHES) {
    Crashlytics.start(this);
}
```

## Setting up multiple build types

In order to fully take advantage of the BuildConfig fields mentioned above, you can create different build types for your app.  For example, we have debug builds for our developers, client builds that we share with our clients at the end of a sprint, and release builds that we upload to the Play Store for publicly released versions of an app.  There are certain features that we want enabled only during development, such as extra logging, that we don’t want to release to the Play Store.

```groovy
android {
    ...
    buildTypes {
        def BOOLEAN = "boolean"
        def TRUE = "true"
        def FALSE = "false"
        def LOG_HTTP_REQUESTS = "LOG_HTTP_REQUESTS"
        def REPORT_CRASHES = "REPORT_CRASHES"
        def ENABLE_VIEW_SERVER = "ENABLE_VIEW_SERVER"
        def ENABLE_SHARING = "ENABLE_SHARING"
        def DEBUG_IMAGES = "DEBUG_IMAGES"

        debug {
            ...
            buildConfigField BOOLEAN, LOG_HTTP_REQUESTS, TRUE
            buildConfigField BOOLEAN, REPORT_CRASHES, FALSE
            buildConfigField BOOLEAN, ENABLE_VIEW_SERVER, TRUE
            buildConfigField BOOLEAN, ENABLE_SHARING, TRUE
            buildConfigField BOOLEAN, DEBUG_IMAGES, TRUE
        }

        client {
            ...
            buildConfigField BOOLEAN, LOG_HTTP_REQUESTS, TRUE
            buildConfigField BOOLEAN, REPORT_CRASHES, TRUE
            buildConfigField BOOLEAN, ENABLE_VIEW_SERVER, FALSE
            buildConfigField BOOLEAN, ENABLE_SHARING, FALSE
            buildConfigField BOOLEAN, DEBUG_IMAGES, FALSE
        }

        release {
            ...
            buildConfigField BOOLEAN, LOG_HTTP_REQUESTS, FALSE
            buildConfigField BOOLEAN, REPORT_CRASHES, TRUE
            buildConfigField BOOLEAN, ENABLE_VIEW_SERVER, FALSE
            buildConfigField BOOLEAN, ENABLE_SHARING, FALSE
            buildConfigField BOOLEAN, DEBUG_IMAGES, FALSE
        }
    }
}
```

## Control the signing keys

When you have multiple build types, another thing you may want to do is use different signing keys for different build types.  For example, we have a shared debug key checked into our repo that we use for debug and client builds so that the app only needs to be configured for APIs like Google+, Facebook, Google Maps, etc with one key hash.  We then have a release key that is not checked in to the repo.  Obviously you will not want to check in your release key credentials, but you won’t want to edit your build.gradle file whenever you want to do a release build.  With the code below, when you run either the assemble or assembleRelease tasks (whether from command line or the Gradle task list UI), Gradle will prompt you, in the terminal window or Android Studio’s Gradle console, for release key credentials.

```groovy
android {
    ...
    signingConfigs {
        debug {
            storeFile file("debug.keystore")
        }

        release {
            storeFile file("release.keystore")
        }
    }

    buildTypes {
        ...

        debug {
            signingConfig signingConfigs.debug
            ...
        }

        client {
            signingConfig signingConfigs.debug
            ...
        }

        release {
            signingConfig signingConfigs.release
            ...
        }
    }
}

def runTasks = gradle.startParameter.taskNames
if ('assemble' in runTasks || 'assembleRelease' in runTasks) {
    android.signingConfigs.release.storePassword = System.console().readPassword('\nKeyStore Password: ')
    android.signingConfigs.release.keyAlias = System.console().readPassword('Alias Name: ')
    android.signingConfigs.release.keyPassword = System.console().readPassword('Alias Password: ')
}
```

## Enabling the Checkstyle plugin

In order to maintain a common style when all of our developers are working on the same app, we use the Checkstyle plugin to verify that the style matches what is expected of us.  In Eclipse, the Checkstyle plugin hooked into the build system and a failure could prevent a successful build.  Well, in Android Studio, the Checkstyle plugin only runs as an inspection. Because of this, you will be able to spot errors in the IDE by the red tick marks along the edge of your file, but it is still possible to build your app with the style errors.  In order to allow Checkstyle to block the build, you need to hook up a custom task into the build dependencies.  This will ensure that whenever you build, Gradle will run Checkstyle and fail your build if it finds an error.

```groovy
task checkstyle(type: Checkstyle) {
    source 'src'
    include '**/*.java'
    exclude '**/gen/**'
    exclude '**/R.java'
    exclude '**/BuildConfig.java'

    def configProps = ['proj.module.dir': projectDir.absolutePath]
    configProperties configProps

    // empty classpath
    classpath = files()
}

preBuild.dependsOn('checkstyle')
```

I hope you have found these tips and tricks useful.  Feel free to share your own favorite Gradle tips and tricks in the comments below!
