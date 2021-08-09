---
layout: single
title:  "Incorrect stacktrace in crashlytics"
date:   2021-06-09 15:09:44 +0700
categories: Android Mobile
---

We recently got a production crash whose call stack did not match the code 
of our application. In the stacktrace in crashlytics and play console, 
we saw something like this: 

```java
Fatal Exception: java.lang.IllegalArgumentException: 
        Parameter specified as non-null is null: 
        method com.example.feature.c.a, parameter $this$extensionB
    at com.example.extensions.ExtentionsA.extensionA(ExtentionsA.java:29)
    at com.example.service.FcmService.onMessageReceived(FcmService.java:92)
    ...
```

And in the source code:

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

That is, the stacktrace shows that `FcmService.onMessageReceived` calls 
`ExtensionsA.extensionA`, even though it was actually calling 
`ExtensionsB.extensionB`. How this happened, read on!

Having carefully studied the stacktrace, another question arises: how did it 
happen that `ExtensionsA` and` ExtensionsB` have a package called 
`com.example.extensions`, and there is another package `com.example.feature.c` in
the crash message? We will find the answer, but later.

## R8

So, our mobile Android application was minified and obfuscated with R8 like 
all others before publishing. 
It means that the stacktrace of the production crash contains the modified class names. That
there used to be a class called `class ExtensionA`{:.language-java}{:.highlight},
and after R8 was applied, it began to be called `class c`{:.language-java}{:.highlight}, and
in the stacktrace, accordingly, it will be called `com.example.extensions.c`.
We load a special `mapping.txt` file along with the newest version of the 
application to deobfuscate such stacktraces. `mapping.txt`
contains one-to-one matching the names of classes/methods of the source 
code and names of classes/methods of release binaries. 
Then crashlytics run the script [./retrace] [retrace] like this after 
receiving a crash report:
```bash
$ ./retrace mapping.txt stacktrace.txt > result_stacktrace.txt
```
and turns an unreadable stacktrace into a readable one. That is how
`com.example.extensions.c` become `com.example.extensions.ExtensionsA`.

Somewhere in this process, a mistake was made, and incorrect class and method 
names were used in the resulting stacktrace.

Let's take a look at our `mapping.txt` and find there 
`class ExtensionsA`{:.Language-java}{:.highlight} and his methods:

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

Almost all the methods of the resulting class `com.example.feature.c` have the 
same name - `a`, the difference is in the arguments and return types, so their
signatures are different. To confirm this fact, we can look at the bytecode
of `com.example.feature.c` class of the release binary. This can be done, for example,
in the following way:

Android Studio -> Build -> Analyse APK... -> <select apk/aab> -> Select .dex file which contains the *.class file -> Found and open it!

![Analyse release][analyse-release]

28 methods of this class are named `a`, and one method is named` b`.

I also wonder why the `R$style` class suddenly began to contain methods from
`ExtensionsA` and other utility classes? The answer lies in the R8 optimizations,
specifically in the `class/merging` optimizations. R8 can combine multiple classes
into one under special conditions. This is what happened: the class
`com.example.feature.R$style` was empty and other static methods were "moved"
to this class to minimize the number of classes in `*.dex` files.

## Kotlin std-lib

Let us remember these facts and begin to deal with another question: 
why the crash message contains a strange package `com.example.feature.c`?
```
Parameter specified as non-null is null: 
    method com.example.feature.c.a, parameter $this$extensionB
```
Why is this strange? Because in the crash in crashlytics everything is deobfuscated
using [./retrace][retrace], but this package remained obfuscated. As
this exception was thrown by kotlin-specific code, then let's take a look at
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
can't change it. Next, we need to look at the implementation
[`Intrinsics.checkParameterIsNotNull`][intrinisics-implementation] in the Kotlin 
standard library to understand how the exception message is generated:
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
That is, the full class name along with the package is taken from the thread 
stacktrace and placed in the exception message. Of course, [./retrace][retrace] 
leaves error messages unchanged.

Before taking the final step, let's collect all our research into a single
schema:

![What happened][pre-finish-scheme]

## Final step

To understand what happened, we are missing one element -
real stacktrace crash. Unfortunately, neither firebase crashlytics nor play
console does not provide an opportunity to look at the stacktrace before it
deobfuscating. Therefore, we can simulate the crash by ourselves. In our case,
it turned out to be quite simple. But in other cases, it is necessary to make 
sure that the `mapping.txt` file will be similar to the `mapping.txt` 
from the original crash version.

So, here is the real stacktrace of the emulated crash:
```java
java.lang.IllegalArgumentException: 
        Parameter specified as non-null is null: 
        method com.example.feature.c.a, parameter $this$extensionB
    at com.example.feature.c.a(Unknown Source:2)
    at com.example.service.FcmListenerService.onMessageReceived(SourceFile:4)
    ...
```

The problem is `Unknown Source: 2`. [./retrace][retrace] chose the first available method
named `com.example.feature.c.a` because it didn't have enough debug
information in the original stacktrace! Probably this information was not there
because the static method `ExtensionsB.extensionB` has become part of the class
`R$style` due to R8 optimizations. Class `R$style` does not compile
along with the entire project. AGP since version `3.3.0`, [packs it immediately
in compiled state] [AGP-3-3-0]:
> Previously, the Android Gradle plugin would generate an R.java file for each 
> of your project's dependencies and then compile those R classes alongside 
> your app's other classes. The plugin now generates a JAR containing your 
> app's compiled R class directly, without first building intermediate 
> R.java classes.

## What we can do with this?

It would seem that we just need to turn off the R8 optimization, which merges small
classes into one. The proguard syntax even allows us to do this:
```text
-optimizations !class/merging/*
```
However, [R8 ignores][R8-optimizations] all rules associated with the discrete 
optimizations, and this rule will not work.
It is possible to disable all optimizations together using the `-dontoptimize` 
rule, but then we risk losing productivity.

At the time of receiving this strange crash, our application was using AGP
version `3.6.2`, which uses R8 version` 1.6.82`. After upgrading the AGP
version to `4.2.2` (which started using R8 version` 2.2.71`) the behavior of the 
optimizations have changed, and now if the methods had a different name in the source code,
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


[retrace]: https://developer.android.com/studio/command-line/retrace
[analyse-release]: {{ site.url }}/assets/incorrect-stacktrace/analyse-release.jpg
[intrinisics-implementation]: https://github.com/JetBrains/kotlin/blob/1.3.50/libraries/stdlib/jvm/runtime/kotlin/jvm/internal/Intrinsics.java#L126-L142
[pre-finish-scheme]: {{ site.url }}/assets/incorrect-stacktrace/pre-finish-scheme.png
[AGP-3-3-0]: https://developer.android.com/studio/releases/gradle-plugin#3-3-0
[R8-optimizations]: https://developer.android.com/studio/build/shrink-code#optimization