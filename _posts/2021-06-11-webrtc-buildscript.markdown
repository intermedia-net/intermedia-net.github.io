---
layout: single
title:  "Resolving WebRTC compatibility issues between multiple dependencies"
date:   2021-06-10 15:09:44 -0700
categories: Android Mobile
---

In this article we continue talking about challenges we are facing during our work on Unite mobile applications. This time we would like to bring up the topic of resolving 3rd party dependencies conflicts when it comes to using the same dependency by multiple application components.

Our Android app uses a 3rd party proprietary library that drives all our VoIP functionality. Under the hood it uses WebRTC packages.

![VoIP SDK jar internals][voip-sdk-jar-internals]

While integrating another app module that also used WebRTC as its dependency we've faced the following issue at compile time due to duplicates in classpath:

```
Duplicate class org.webrtc.BaseBitrateAdjuster found in modules jetified-libwebrtc-runtime.jar (libwebrtc.aar)
Duplicate class org.webrtc.BitrateAdjuster found in modules jetified-libwebrtc-runtime.jar (libwebrtc.aar)
Duplicate class org.webrtc.CalledByNative found in modules jetified-libwebrtc-runtime.jar (libwebrtc.aar)
Duplicate class org.webrtc.Camera1Capturer found in modules jetified-libwebrtc-runtime.jar (libwebrtc.aar)
Duplicate class org.webrtc.Camera1Enumerator found in modules jetified-libwebrtc-runtime.jar (libwebrtc.aar)
Duplicate class org.webrtc.Camera1Session found in modules jetified-libwebrtc-runtime.jar (libwebrtc.aar)
Duplicate class org.webrtc.Camera1Session$1 found in modules jetified-libwebrtc-runtime.jar (libwebrtc.aar)
Duplicate class org.webrtc.Camera1Session$2 found in modules jetified-libwebrtc-runtime.jar (libwebrtc.aar)
Duplicate class org.webrtc.Camera1Session$SessionState found in modules jetified-libwebrtc-runtime.jar (libwebrtc.aar)
...
```

Our first thought was: most likely our VoIP library just copied the original WebRTC classes, so we could try to remove them
from its jar file, and resolve build errors. We did it and the compilation was successful =)
However, the following crashes appeared at runtime:

```
java.lang.ClassNotFoundException: Didn't find class "org.webrtc.MediaCodecVideoDecoder" on path: DexPathList[[zip file "/data/app/<app_package_name>==/base.apk"],nativeLibraryDirectories=[/data/app/<app_package_name>==/lib/arm64, /data/app/<app_package_name>==/base.apk!/lib/arm64-v8a, /system/lib64]]
     at dalvik.system.BaseDexClassLoader.findClass(BaseDexClassLoader.java:196)
     at java.lang.ClassLoader.loadClass(ClassLoader.java:379)
     at java.lang.ClassLoader.loadClass(ClassLoader.java:312)
     at java.lang.Runtime.nativeLoad(Native Method)
     at java.lang.Runtime.nativeLoad(Runtime.java:1115)
     at java.lang.Runtime.loadLibrary0(Runtime.java:1069)
     at java.lang.Runtime.loadLibrary0(Runtime.java:1007)
     at java.lang.System.loadLibrary(System.java:1667)
     ...
	#
    # Fatal error in ../../sdk/android/src/jni/classreferenceholder.cc, line 132
    # last system error: 11
    # Check failed: !jni->ExceptionCheck()
    # error during FindClass: org/webrtc/MediaCodecVideoDecoder
    #
``` 

Another unsuccessful attempt. Obviously, our app module and VoIP library are using different
versions of WebRTC. The ideal solution would be if VoIP SDK vendor in
the implementation of their library will completely rename classes copied
from WebRTC since they were using them as part of the library sources rather than a depenedency. For example, the package name would change
from `org.webrtc` to `com.voipsdk`. But this way is not possible, since we don't have access to library sources and waiting for the requested update from the vendor side could take too much time.

Therefore, we will have to rename the package name when adding the original
WebRTC library from `org.webrtc` to something else, for example, `unite.webrtc`.
Then the absence of conflicts during compilation and in runtime is guaranteed!

There are two ways to rename the package name of the original WebRTC:
- rename the package name in the sources and build a new library
- rename the package name in the `libwebrtc.aar`'s classes and binaries

It seems that the first way is easier, since renaming packages of Java classes
in the `libwebrtc.aar` binary looks non-trivial. So,
[download the webrtc sources][webrtc-android-readme] and start looking for
what needs to be renamed there.

## Rename `org.webrtc` occurrences
It's clear that the first thing that needs to be renamed in the source is
all occurrences of the substring `org.webrtc` to `unite.webrtc`.
Examples of such occurrences:
```java
package org.webrtc;
...
```

```java
...

import org.webrtc.ThreadUtils.ThreadChecker;
...
```
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="org.webrtc">
  <uses-sdk android:minSdkVersion="21" android:targetSdkVersion="23" />
</manifest>
```

[_Replace in Files..._][jetbrains-replace-in-files] and you're done! However,
it is not enough, except that we renamed the 
`package org.webrtc`{:.language-java}{:.highlight} to 
`package unite.webrtc`{:.language-java}{:.highlight}, we also need to rename 
the path to such files, for example:
`./sdk/android/api/org/webrtc/AddIceObserver.java` ->
`./sdk/android/api/unite/webrtc/AddIceObserver.java`.
We have to use our hands. Don't worry, we will automate it later.

## Rename `org/webrtc` occurrences
There is a reference to java files by their relative path, in the configuration
files and the C source codes. So we need to rename all the substrings of
`org / webrtc` to `unite / webrtc`:

`BUILD.gn`:
```
rtc_android_library("base_java") {
    sources = [
      "api/org/webrtc/Predicate.java",
      "api/org/webrtc/RefCounted.java",
      "src/java/org/webrtc/CalledByNative.java",
      "src/java/org/webrtc/CalledByNativeUnchecked.java",
```

`simple_peer_connection.cc`:
```c
jmethodID link_camera_method = webrtc::GetStaticMethodID(
        env, pc_factory_class, "LinkCamera",
        "(JLorg/webrtc/SurfaceTextureHelper;)Lorg/webrtc/VideoCapturer;");
```

## Rename `org_webrtc` occurrences
To access Java classes through the JNI interface, the class name description
is used through `org_webrtc` prefixes. Need to rename it to `unite_webrtc` too:

`ice_candidate.cc`:
```c
return NativeToJavaObjectArray(jni, candidates,
                               org_webrtc_IceCandidate_clazz(jni),
                               &NativeToJavaCandidate);
```

In general, that's all, if you do the correct renaming of these entries
(there will be about 250 changes in the code, including files moving), then
it will be possible to build a correctly working `libwebrtc.aar`, which will
have all java classes with the `unite.webrtc` package.

It would be possible to commit these changes to a WebRTC fork, and all builds
of this fork will have a different package name. However, there will a lot of
maintainability issues in that case. For example, if a new Java files will be
added to the original repository, then in our repository we will have to rename
them again. There will also be a risk of complex merge conflicts when updating
a fork.
Therefore, to simplify Maintainability we will not commit so many changes.

It seems that the changes made to the code are mechanical enough to be automated
with some simple Python script.

So, here is the plan:
- Make a script that will rename these entries `org.webrtc`/`org/webrtc`/`org_webrtc`
- Run this script every time before building `webrtc`
- Profit!

Thus, formally, we will not make any changes at all to the source code
of WebRTC, but simply run a special script for preparing the project before
building!

The script turned out to be universal, it can be found in
[intermedia-net/jpkgchanger][jpkgchanger], and it is most likely may be suitable
not only for assembling WebRTC with custom package name:
```bash
jpkgchanger --current org.webrtc --target unite.webrtc
```


[voip-sdk-jar-internals]: {{ site.url }}/assets/webrtc-buildscript/portsip-voip-sdk-jar-internals.png
[portsip-voip-sdk]: https://www.portsip.com/portsip-voip-sdk/
[portsipvoipsdk-jar]: https://www.portsip.com/downloads/sdk/AndroidSample.zip
[webrtc-android-readme]: https://webrtc.github.io/webrtc-org/native-code/android/
[jetbrains-replace-in-files]: https://www.jetbrains.com/help/idea/finding-and-replacing-text-in-project.html#replace_search_string_in_project
[jpkgchanger]: https://github.com/intermedia-net/jpkgchanger
