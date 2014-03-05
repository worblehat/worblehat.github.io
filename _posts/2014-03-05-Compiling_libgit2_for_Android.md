---
layout: post
title: "Compiling libgit2 for Android"
categories: [libgit2]
date: 2014-03-05
commentIssueId: 2
published: true 
---

This is a step-by-step guide to cross compiling [libgit2](http://libgit2.github.com/) for Android
on a Linux system. The goal is to be able to use the git library written in C in Android
applications via the [NDK (Native Development Kit)](https://developer.android.com/tools/sdk/ndk/index.html)
and [JNI](http://developer.android.com/training/articles/perf-jni.html).

**NOTE:** Here libgit2 will be compiled *without* the optional depenendcy OpenSSL, but I plan to write another
article on how to include OpenSSL into the build (as soon as I figure it out).

# Preparation

First of all you should [get the latest NDK](https://developer.android.com/tools/sdk/ndk/index.html).
Extract the archive to your disk and save the folder's path to an environment variable as this will be
useful later.

```shell
$ tar xvzf android-ndk-r9c-linux-x86_64.tar.bz2
$ export NDK=<path>/android-ndk-r9c-linux-x86_64
```

The version and architecture might be different for you
and you need to replace `<path>` by whatever path you extracted the NDK to.

# Create a Standalone Toolchain

There are two options when compiling native code for Android:

1. Using the NDK's build system with the [ndk-build script](http://www.kandroid.org/ndk/docs/NDK-BUILD.html)
    and Android make files.
2. Using your own build system, e.g. with conventional make or CMake build files in conjunction with
    a [standalone toolchain](http://www.kandroid.org/ndk/docs/STANDALONE-TOOLCHAIN.html).

libgit2 is already configured to be built with CMake so you want to choose the second option.
As libgit2 needs to be cross compiled for a different architecture with Android specific standard libraries
etc., you have to tell CMake where to find the proper toolchain.

At first this toolchain needs to be created. Fortunately a script for this task comes along with the NDK.

Below is an invocation of this script to make a standalone toolchain that contains the clang compiler and that
is targeting Android Platform level 9 for devices with ARM architecture.
`$TOOLCHAIN` should be a path of your own choice where the toolchain
will be installed. This is not really important because you are free to move the folder afterwards.

```Shell
$ $NDK/build/tools/make-standalone-toolchain.sh \
    --toolchain=arm-linux-androideabi-clang3.3 \
    --platform=android-9 \
    --arch=arm \
    --install-dir=$TOOLCHAIN
```

Of course you can choose another compiler, architecture or platform target to fit your needs.
If your host system is 64Bit, 
you can also specify `--system=linux-x86_64`. For more details and available options see
STANDALONE-TOOLCHAIN.html in NDK's documentation folder 
(also available [online](http://www.kandroid.org/ndk/docs/STANDALONE-TOOLCHAIN.html)).

Everything needed to cross compile (compiler, NDK headers, libraries, ...) is installed now and 
you are ready to actually build libgit2.

# Compile libgit2

Download the [libgit2 sources](https://github.com/libgit2/libgit2/releases) and create a file named
toolchain.cmake with the following content in it's root directory
(actually you can create the file wherever you like as long as you provide the correct path to CMake later).

```CMake
SET(CMAKE_SYSTEM_NAME Linux)
SET(CMAKE_SYSTEM_VERSION Android)

SET(CMAKE_C_COMPILER   $ENV{TOOLCHAIN}/bin/arm-linux-androideabi-clang)
SET(CMAKE_CXX_COMPILER $ENV{TOOLCHAIN}/bin/arm-linux-androideabi-clang++)
SET(CMAKE_FIND_ROOT_PATH $ENV{TOOLCHAIN}/sysroot/)

SET(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
SET(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
SET(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

Note that we use the $TOOLCHAIN variable again, so make sure it is set correctly or hardcode the path
to the toolchain installation.

Then create a build directory and configure CMake from within. Make sure you set `$LIBGIT2_INSTALL`
(or replace it with) the path where you want the libgit2 binaries and header to be installed.

```Shell
$ mkdir build_android
$ cd build_android
$ cmake -DCMAKE_TOOLCHAIN_FILE=../toolchain.cmake \
        -DANDROID=1  \
        -DSONAME=0 \
        -DCMAKE_INSTALL_PREFIX=$LIBGIT2_INSTALL \
        .. 
```

Note that the output tells you:

```Shell
"Could NOT find OpenSSL"
```
As mentioned earlier this will be the subject of another article on this site.

If you have a version of libgit2 that is older than version 0.21 you might get this error:

```
ld: error: cannot find -lrt
```

This can be fixed by
[patching CMakelists.txt](https://github.com/libgit2/libgit2/commit/5af69ee96af6dfae0f9069c6cda5281861b0da5c)
manually.

If the configuration was successful only step left is to finally build libgit2.

```Shell
$ cmake --build . --target install
```

Congratulations! Now you have libgit2 built as a shared library for Android.  
An article on how to actually use this native library in an Android application will follow...
