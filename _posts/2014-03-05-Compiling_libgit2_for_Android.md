---
layout: post
title: "Compiling libgit2 for Android"
categories: [libgit2]
modified: 2014-04-11 00:00:00
commentIssueId: 2
published: true 
---

This is a step-by-step guide to cross compiling [libgit2](http://libgit2.github.com/) for Android
on a Linux system. The goal is to be able to use the git library written in C in Android
applications via the [NDK (Native Development Kit)](https://developer.android.com/tools/sdk/ndk/index.html)
and [JNI](http://developer.android.com/training/articles/perf-jni.html).

# Preparation

First of all you should [get the latest NDK](https://developer.android.com/tools/sdk/ndk/index.html).
Extract the archive to your disk and save the folder's path to an environment variable as this will be
useful later.

{% highlight bash %}
tar xvzf android-ndk-r9c-linux-x86_64.tar.bz2
export NDK=<path>/android-ndk-r9c-linux-x86_64
{% endhighlight %}

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

{% highlight bash %}
$NDK/build/tools/make-standalone-toolchain.sh \
    --toolchain=arm-linux-androideabi-clang3.3 \
    --platform=android-9 \
    --arch=arm \
    --install-dir=$TOOLCHAIN
{% endhighlight %}

Of course you can choose another compiler, architecture or platform target to fit your needs.
If your host system is 64Bit, 
you can also specify `--system=linux-x86_64`. For more details and available options see
STANDALONE-TOOLCHAIN.html in NDK's documentation folder 
(also available [online](http://www.kandroid.org/ndk/docs/STANDALONE-TOOLCHAIN.html)).

Everything needed to cross compile (compiler, NDK headers, libraries, ...) is installed now and 
you are ready to actually build libgit2.

# <a id="Optional_Dependencies"></a>Optional Dependencies

libgit2 has some optional dependencies to provide additional features.
Threading support will be available, because the pthreads library is built into
Android's C library.

The other optional dependenices (OpenSSL and LibSSH2) won't be available, unless you cross-compile them, too.
Luckily, GitHub user [mevansam](https://github.com/mevansam/cmoss) created a couple of scripts capable of building
some popular C/C++ open source libraries for Android. I created a [fork](https://github.com/worblehat/cmoss)
with some minor modifications, tailored to build OpenSSL and LibSSH2 for Android. I configured it to work
with NDK r9c, targeting the arm platform, but you can change some variables, like target platform and
toolchain version in
[`build-all.sh`](https://github.com/worblehat/cmoss/blob/libgit2/build-droid/build-all.sh#L91).

After running `build-all.sh` just copy `libssh2.a`, `libssl.a`, `libcrypto.a` and `libgpg-error.a` to
`$TOOLCHAIN/sysroot/usr/lib` and the corresponding headers to `$TOOLCHAIN/sysroot/usr/include`. 
Then libgit2's build system will be able to find and link against them.

# Compile libgit2

Download the [libgit2 sources](https://github.com/libgit2/libgit2/releases) and create a file named
toolchain.cmake with the following content in it's root directory
(actually you can create the file wherever you like as long as you provide the correct path to CMake later).

{% highlight CMake %}
SET(CMAKE_SYSTEM_NAME Linux)
SET(CMAKE_SYSTEM_VERSION Android)

SET(CMAKE_C_COMPILER   ${TOOLCHAIN}/bin/arm-linux-androideabi-clang)
SET(CMAKE_CXX_COMPILER ${TOOLCHAIN}/bin/arm-linux-androideabi-clang++)
SET(CMAKE_FIND_ROOT_PATH ${TOOLCHAIN}/sysroot/)

SET(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
SET(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
SET(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
{% endhighlight %}

Note that we use the `$TOOLCHAIN` variable again, so make sure it is set correctly or hardcode the path
to the toolchain installation. The path should be absolute.

Then create a build directory and configure CMake from within. Make sure you set `$LIBGIT2_INSTALL`
(or replace it with) the path where you want the libgit2 binaries and header to be installed.

{% highlight bash %}
mkdir build_android
cd build_android
cmake -DCMAKE_TOOLCHAIN_FILE=../toolchain.cmake \
        -DANDROID=1  \
        -DBUILD_SHARED_LIBS=0 \
        -DTHREADSAFE=1 \
        -DBUILD_CLAR=0 \
        -DCMAKE_INSTALL_PREFIX=$LIBGIT2_INSTALL \
        .. 
{% endhighlight %}

As `BUILD_SHARED_LIBS` is set to false, a static library will be built. This makes sense for Android
because the library will be bundled with the application anyway. So even if you use a shared library,
it will not really be shared among multiple apps that use it. Instead every app will have it's own 'shared'
version of libgit2.

If you have a version of libgit2 that is older than version 0.21, you might get this error when running cmake:

{% highlight bash %}
ld: error: cannot find -lrt
{% endhighlight %}

This can be fixed by
[patching CMakelists.txt](https://github.com/libgit2/libgit2/commit/5af69ee96af6dfae0f9069c6cda5281861b0da5c)
manually.

If the configuration was successful, the only step left is to finally build libgit2.

{% highlight bash %}
cmake --build . --target install
{% endhighlight %}

Congratulations! Now you have libgit2 built as a static library for Android.  
Don't hesitate to leave a comment below and read my other post on
[how to actually use libgit2 on Android](http://worblehat.github.io/Using_libgit2_on_Android/).
