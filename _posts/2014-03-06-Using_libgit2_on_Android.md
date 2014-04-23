---
layout: post
title: "Using libgit2 on Android"
categories: [libgit2]
modified: 2014-04-23 12:00:00
commentIssueId: 3
published: true 
---

In this guide I will show you how to build a minimal Android application that uses the [libgit2]() C library
via Android's [NDK (Native Development Kit)](https://developer.android.com/tools/sdk/ndk/index.html)
and [JNI](http://developer.android.com/training/articles/perf-jni.html).

**NOTE:** This is the continuation of my first post about
[compiling libgit2 for Android](http://worblehat.github.io/Compiling_libgit2_for_Android/).

#Prerequisites

- **libgit2** compiled for Android (see above)
- An (Java-only) **Android project** created with Eclipse ADT or similar (see the official
    [Android documentation](http://developer.android.com/training/basics/firstapp/creating-project.html)
    for setup instructions) 
- Downloaded [NDK for Android](http://developer.android.com/tools/sdk/ndk/index.html#GetStarted)
- **Basic knowledge** on how to program Android applications and some experience with JNI would be handy.

The following examples assume that the application and project name is 'Hello_Git' and there is a Java
package called `com.example.hello_git`.

#Interfacing C/C++

First of all you should write the essential code that builds the 'bridge' between the Java based app
and the C based library. The Java Native Interface (JNI) is used for this. I won't go into detail about
JNI, so I suggest you have a look at its
[specification](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/jniTOC.html) or any tutorial
if you haven't heard of it.

Create a new Java class in your project, let's say `LibGit2` to emphasize that it can be used to interface
libgit2. Then add the signature for a static **native method** called `getVersion()` returning a String.

{% highlight Java %}
package com.example.hello_git;

public class LibGit2 {

    public native static String getVersion();

}
{% endhighlight %}

The `native` keyword indicates that the actual implementation of this method will be provided by
native (i.e. C/C++) Code. Therefore it has no method body here, but is terminated with a semicolon (like
abstract methods).

Note that our goal is to only build a minimal application that connects to libgit2, so it will just be able
to get libgit2's version for now. You can add more functionality later by adding 
more native methods to interface the library. 

Make sure your Java project is built and the `.class` files are in `bin/classes`.
Now you can generate the C headers that define the interface to the native Code by 
running javah from the root of your project:

{% highlight bash %}
javah -jni -classpath bin/classes/ -d jni/ com.example.hello_git.LibGit2
{% endhighlight %}

This will create a folder `jni` with a `.h` file. It contains the method declaration
that corresponds to the native Java method from `LibGit2`. 

{% highlight C %}
JNIEXPORT jstring JNICALL Java_com_example_hello_1git_LibGit2_getVersion
  (JNIEnv *, jclass);
{% endhighlight %}

The call to the `native` Java method will be forwarded to a C/C++ method with that signature.
So all you have to do is, to implement the generated C header. This is where you actually use libgit2
and translate its output to a format understood by Java. 
Create a file named `com_example_hello_git_LibGit2.c` next to the header file.

{% highlight C %}
#include <com_example_hello_git_LibGit2.h>

#include <git2/common.h>

JNIEXPORT jstring JNICALL Java_com_example_hello_1git_LibGit2_getVersion
  (JNIEnv * env, jclass cls)
{
    int major, minor, rev;
    git_libgit2_version(&major, &minor, &rev);
    char version[15];
    sprintf(version, "%d.%d.%d", major, minor, rev);

    return (*env)->NewStringUTF(env, version);
}
{% endhighlight %}

This gets the version of libgit2, stores it in a `char` array and afterwards uses a method from class `JNIEnv`
(defined in `jni.h`) to convert it to a `jstring`, which is the declared return type.
As I already said, I won't go into detail about the JNI related stuff here.


#Building the native Part

As the native part of the Android app depends on libgit2, you should copy its binary and headers to a place
in your project directory. I like to keep them in `include` and `lib` directories
under `jni/`. 

Create an `Android.mk` file in `jni/`. It will be the build script for Android's
build system. Your jni directory structure should now look like this:

{% highlight Bash %}
jni/
├── Android.mk
├── com_example_hello_git_LibGit2.c
├── com_example_hello_git_LibGit2.h
├── include
│   ├── git2
│   │   └── (...)
│   └── git2.h
└── lib
    └── libgit2.a
{% endhighlight %}

In `Android.mk` you define so called 'modules' that will be built from native code. Each module is
either a static or a shared library.

{% highlight Make %}
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
LOCAL_MODULE := git2
LOCAL_SRC_FILES := $(LOCAL_PATH)/lib/libgit2.a
include $(PREBUILT_STATIC_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE := com_example_hello_git_LibGit2
LOCAL_SRC_FILES := com_example_hello_git_LibGit2.c
LOCAL_C_INCLUDES := $(LOCAL_PATH)/include
LOCAL_STATIC_LIBRARIES := git2
LOCAL_LDLIBS := -lz
include $(BUILD_SHARED_LIBRARY)
{% endhighlight %}

This makefile defines two modules. The first one (`git2`) is the prebuilt static libgit2 library.
The variable `$(LOCAL_PATH)` contains the path of your `jni` directory and is used to specify the location
of the binary.

The second module is a shared library that will be built from the source file
`com_example_hello_git_LibGit2.c`. The module description says that it depends on the `git2` module and that
the compilation process should look in the `include` directory for necessary header files (of libgit2).

We also pass a linker option `-lz` to link against zlib, because libgit2 depends on zlib, which is
a shared library already included in the NDK.

If you have built libgit2 with its optional dependencies OpenSSL and libssh2
(see ['Optional Dependencies'](http://localhost:4000/Compiling_libgit2_for_Android/#Optional_Dependencies)),
add the binaries and headers of OpenSSL, libssh2, libgcrypt and libgpg-error to `lib` and `include`
respectively. Furthermore for each of them a module must be defined in Android.mk and added to `LOCAL_STATIC_LIBRARIES`. The complete makefile can be found [here](https://gist.github.com/worblehat/11056229).

By now you have everything setup to actually build the project. The compilation of
native modules is seperate from the usual build routine of the Android app itself.
So, before compiling your app to an .apk file, you have to run an NDK build.
This is pretty straightforward. Just go to the jni folder and execute the `ndk_build`
script located in the root folder of Android's NDK:

{% highlight Bash %}
cd jni
$NDK/ndk_build
{% endhighlight %}

This will build all the native modules from your `Android.mk`. The output should look
like this:

{% highlight Bash %}
[armeabi] SharedLibrary  : libcom_example_hello_git_LibGit2.so
[armeabi] Install        : libcom_example_hello_git_LibGit2.so \
    => libs/armeabi/libcom_example_hello_git_LibGit2.so
{% endhighlight %}

A shared library was created and placed in you project's `libs` directory.

Now the native part is ready to be used in your Android application. Just remember to
re-run `ndk_build` whenever you have modifications in the C/C++ code base.

#Using the native Library

As the built library is shared, it must be loaded at runtime. Go back to the `LibGit2` 
Java class created earlier and add a static initialization block. Here the library 
can be loaded.

{% highlight Java %}
package com.example.hello_git;

public class LibGit2 {

    static {
        System.loadLibrary("com_example_hello_git_LibGit2");
    }

    public native static String getVersion();

}
{% endhighlight %}

Note that the 'lib'-prefix and the '.so' extension must be stripped from the library
name. From now on you can use the native method `getVersion()` like any other Java
method in your Code. For example you can show the version of libgit2 in a TextView:

{% highlight Java %}
TextView versionText = (TextView)findViewById(R.id.versionTextView);
versionText.setText(LibGit2.getVersion());
{% endhighlight %}

Of course this is not really useful so far, but based on this project setup you
should be able to utilize all of the libgit2 functionality in your Android apps.

Feel free to leave comments with questions or ideas for improvements below.
