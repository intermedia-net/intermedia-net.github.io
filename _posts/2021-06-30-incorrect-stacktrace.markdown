---
layout: single
title:  "Chasing the wrong Android app stacktrace in Crashlytics"
date:   2021-07-01 15:09:44 +0700
categories: Android Mobile
---

In this article we share the story of an interesting investigation we had to perform in order to track down a misleading stack trace we found in our Crashlytics reports.

We had received a crash report whose call stack did not match the source code of our application. Both in Crashlytics and the Google Play console we saw something like this:

```java
Fatal Exception: java.lang.IllegalArgumentException: 
        Parameter specified as non-null is null: 
        method com.example.feature.c.a, parameter $this$extensionB
    at com.example.extensions.ExtentionsA.extensionA(ExtentionsA.java:29)
    at com.example.service.FcmService.onMessageReceived(FcmService.java:92)
    ...
```

And in the actual source code of the project we had the following:

`FcmService.java`:
```java
package com.example.service;

class FcmService {
    ...
    void onMessageReceived() {
        ExtensionsB.extensionB(someNullableString, param);
    }
    ...
}
```
`ExtensionsA.kt`:
```kotlin
package com.example.extensions

fun String.extensionB(param: Int) {
    ...
}

```

As you can see the stacktrace indicates that `FcmService.onMessageReceived` calls 
`ExtensionsA.extensionA`, even though it was actually calling 
`ExtensionsB.extensionB`.

Having carefully studied the stacktrace we asked ourselves a question: how did it 
happen that `ExtensionsA` and` ExtensionsB` have a package called 
`com.example.extensions` but there is another package `com.example.feature.c` in
the crash message? In search of the answer we started to look into our application's obfuscation configuration.

## R8

Our Android application was minified and obfuscated with the R8 just like 
all other apps are before publishing, meaning the stacktrace of the production crash contained the modified class names. 
If there used to be a class called `class ExtensionA`{:.language-java}{:.highlight},
then after R8 was applied, it transformed into `class c`{:.language-java}{:.highlight}.
In the resulting stacktrace it would be called `com.example.extensions.c`.
We load a special `mapping.txt` file to Crashlytics along with the new release of our 
application in order to deobfuscate such stacktraces. `mapping.txt`
contains a one-to-one matching for names of classes/methods from the source 
code and names of classes/methods from release binaries. 
Then Crashlytics run the script [./retrace] [retrace] after 
receiving a crash report:
```bash
$ ./retrace mapping.txt stacktrace.txt > result_stacktrace.txt
```
and turns an unreadable stacktrace into a readable one. That is how
`com.example.extensions.c` became `com.example.extensions.ExtensionsA`.

There was an error somewhere in this process and incorrect class and method 
names were used in the resulting stacktrace.

Let's take a look at our `mapping.txt` and find 
`class ExtensionsA`{:.Language-java}{:.highlight} and its methods:

```text
com.example.feature.R$style -> com.example.feature.c:
    1:1:void com.example.ExtensionsA.extensionA(java.lang.String,java.lang.Integer,java.lang.Integer):29:30 -> a
    ...
    9:9:android.view.View com.example.feature.extensions.AndroidExtensionsKt.inflate(android.view.ViewGroup,int,boolean):0:0 -> a
    9:9:android.view.View com.example.feature.AndroidExtensionsKt.inflate$default(android.view.ViewGroup,int,boolean,int,java.lang.Object):23 -> a
    10:10:android.view.View com.example.feature.extensions.AndroidExtensionsKt.inflate(android.view.ViewGroup,int,boolean):24:24 -> a
    10:10:android.view.View com.example.feature.extensions.AndroidExtensionsKt.inflate$default(android.view.ViewGroup,int,boolean,int,java.lang.Object):23 -> a
    ...
    14:15:void com.example.ExtensionsA.extensionC(java.lang.CharSequence,java.lang.String):11:12 -> a
    16:17:void com.example.ExtensionsA.extensionC(java.lang.CharSequence,java.lang.String):15:16 -> a
    18:18:void com.example.ExtensionsA.extensionC(java.lang.CharSequence,java.lang.String) -> a
    19:19:void com.example.ExtensionsA.extensionC(java.lang.CharSequence,java.lang.String):17:17 -> a
    ...
    27:28:void com.example.ExtensionsB.extensionB(java.lang.String,java.lang.Integer):18:19 -> a
```

Almost all methods of the resulting class `com.example.feature.c` have the 
same obfuscated name - `a`, the difference is in the arguments and return types, so their
signatures are different. To confirm this fact, we can look at the bytecode
of `com.example.feature.c` class from the release binary. This can be done, for example,
from Android Studio:

Build -> Analyse APK... -> <select apk/aab> -> Select .dex file which contains the *.class file -> open it

![Analyse release][analyse-release]

28 methods of this class are named `a`, and one method is named` b`.

We also wondered why the `R$style` class suddenly contained methods from
`ExtensionsA` and other utility classes? The answer lies in the R8 optimizations,
specifically in the `class/merging` optimizations. R8 can combine multiple classes
into one under special conditions. This is what happened: the class
`com.example.feature.R$style` was empty and other static methods were "moved"
to this class to minimize the number of classes in `*.dex` files.

## Kotlin std-lib

Let's keep in mind the above mentioned facts and start dealing with another question: 
why the crash message contained the strange package `com.example.feature.c`?
```
Parameter specified as non-null is null: 
    method com.example.feature.c.a, parameter $this$extensionB
```
Why is this strange? Because in the crash in Crashlytics everything is deobfuscated
using [./retrace][retrace], but this package remained obfuscated. As
this exception was thrown by kotlin-specific code, we then took a look at
its bytecode (for simplicity we decompile it in Java) to understand the reason:

```java
public final class ExtensionsB {
    @NotNull
    public static void extensionB(
            @NotNull String $this$extensionB, 
            final int param
    ) {
        Intrinsics.checkParameterIsNotNull(
                $this$extensionB, 
                "$this$extensionB"
        );
    }
}
```
Here we can see that the argument name is stored as a string, and R8, of course,
can't change it. Next, we needed to look at the implementation
[`Intrinsics.checkParameterIsNotNull`][intrinisics-implementation] in the Kotlin 
standard library to understand how the exception message was generated:
```java
StackTraceElement[] stackTraceElements = Thread.currentThread().getStackTrace();
StackTraceElement caller = stackTraceElements[3];
String className = caller.getClassName();
String methodName = caller.getMethodName();

IllegalArgumentException exception = new IllegalArgumentException(
        "Parameter specified as non-null is null: " +
        "method " + className + "." + methodName +
        ", parameter " + paramName
);
throw sanitizeStackTrace(exception);
```
That is, the full class name along with the package was taken from the thread 
stacktrace and placed in the exception message. Of course, [./retrace][retrace] 
left error messages unchanged.

Before taking the final step, here is all research compiled into a single
schema:

![What happened][pre-finish-scheme]

## Final step

To understand what happened, we were missing one element -
a real stacktrace crash. Unfortunately, neither Firebase Crashlytics nor Google Play
console does not provide an opportunity to look at the stacktrace before its
deobfuscation. Therefore, we needed to simulate the crash by ourselves. In our case,
it turned out to be quite simple. But in other cases, it is necessary to make 
sure that the `mapping.txt` file is similar to the `mapping.txt` 
from the original crashed application version.

So, here is the real stacktrace of the emulated crash:
```java
java.lang.IllegalArgumentException: 
        Parameter specified as non-null is null: 
        method com.example.feature.c.a, parameter $this$extensionB
    at com.example.feature.c.a(Unknown Source:2)
    at com.example.service.FcmListenerService.onMessageReceived(SourceFile:4)
    ...
```

The problem was `Unknown Source: 2`. [./retrace][retrace] chose the first available method
named `com.example.feature.c.a` because it didn't have enough debug
information in the original stacktrace! This information was likely not there
because the static method `ExtensionsB.extensionB` had become a part of the class
`R$style` due to R8 optimizations. Class `R$style` does not compile
along with the entire project. Android Gradle Plugin since version `3.3.0`, [packs it immediately
in compiled state] [AGP-3-3-0]:
> Previously, the Android Gradle plugin would generate an R.java file for each 
> of your project's dependencies and then compile those R classes alongside 
> your app's other classes. The plugin now generates a JAR containing your 
> app's compiled R class directly, without first building intermediate 
> R.java classes.

## So what did we learn?

It would appear that we just need to turn off the R8 optimization, which merges small
classes into one. The proguard syntax even allows us to do this:
```text
-optimizations !class/merging/*
```
However, [R8 ignores][R8-optimizations] all rules associated with the discrete 
optimizations, and this rule will not work.
It is possible to disable all optimizations together using the `-dontoptimize` 
rule, but this might affect application performance which is not desirable.

At the time of receiving this strange crash, our application was using AGP
version `3.6.2`, which uses R8 version` 1.6.82`. After upgrading the AGP
version to `4.2.2` (which started using R8 version` 2.2.71`) the behavior of the 
optimizations changed, and now if the methods have a different name in the source code,
then they will also have a different name in the optimized one:
```text
com.example.feature.R$style -> com.example.feature.c:
    1:1:void com.example.ExtensionsA.extensionA(java.lang.String,java.lang.Integer,java.lang.Integer):29:30 -> A
    ...
    9:9:android.view.View com.example.feature.extensions.AndroidExtensionsKt.inflate(android.view.ViewGroup,int,boolean):0:0 -> B
    9:9:android.view.View com.example.feature.AndroidExtensionsKt.inflate$default(android.view.ViewGroup,int,boolean,int,java.lang.Object):23 -> B
    10:10:android.view.View com.example.feature.extensions.AndroidExtensionsKt.inflate(android.view.ViewGroup,int,boolean):24:24 -> B
    10:10:android.view.View com.example.feature.extensions.AndroidExtensionsKt.inflate$default(android.view.ViewGroup,int,boolean,int,java.lang.Object):23 -> B
    ...
    14:15:void com.example.ExtensionsA.extensionC(java.lang.CharSequence,java.lang.String):11:12 -> C
    16:17:void com.example.ExtensionsA.extensionC(java.lang.CharSequence,java.lang.String):15:16 -> C
    18:18:void com.example.ExtensionsA.extensionC(java.lang.CharSequence,java.lang.String) -> C
    19:19:void com.example.ExtensionsA.extensionC(java.lang.CharSequence,java.lang.String):17:17 -> C
    ...
    27:28:void com.example.ExtensionsB.extensionB(java.lang.String,java.lang.Integer):18:19 -> D
```
Accordingly, if `Unknown Source` is present in the stack trace, then 
[./retrace][retrace] will detect the correct name of the original method
and class by unique method name.

So, at the end of the day, we just had to upgrade our Gradle plugin to a newer version - but since it is not always a working solution for some legacy projects it is good to know the reason behind this wrong stacktrace issue. Hopefully this information will save you valuable troubleshooting time or help you plan in advance to prevent the issue from occurring.

[retrace]: https://developer.android.com/studio/command-line/retrace
[analyse-release]: {{ site.url }}/assets/incorrect-stacktrace/analyse-release.jpg
[intrinisics-implementation]: https://github.com/JetBrains/kotlin/blob/1.3.50/libraries/stdlib/jvm/runtime/kotlin/jvm/internal/Intrinsics.java#L126-L142
[pre-finish-scheme]: {{ site.url }}/assets/incorrect-stacktrace/pre-finish-scheme.png
[AGP-3-3-0]: https://developer.android.com/studio/releases/gradle-plugin#3-3-0
[R8-optimizations]: https://developer.android.com/studio/build/shrink-code#optimization