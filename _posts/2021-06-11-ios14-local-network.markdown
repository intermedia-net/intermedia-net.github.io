---
layout: single
title:  "Dealing with Crashes Caused by iOS 14 Local Network Permission"
date:   2021-06-17 15:09:44 -0700
categories: iOS Mobile
---
Hello from Unite Mobile team! Let's start with a short introduction: our team works on Unite mobile applications for iOS and Android. These apps are a key component of Intermedia's UCaaS offering providing essential calling and messaging features.

In this article we want to share with you a story of weird iOS app crashes and how we've fixed them.

It all started after iOS 14 had gone gold. Since its public release we've started getting a number of complaints from our customers that Unite app began crashing for them at startup. One thing was common among all of the reported issues: app was crashing only when connected to wifi network. Also in most cases app was crashing right after launch if user was previously logged in or right after the login process. Interesting enough that this issue wasn't reproduced on any of our test or personal devices as well as on any of the wifi networks and we also didn't get any complaints from our beta testers as well. What was even more weird is that these crashes weren't detected by Crashlytics so we neither had any crash logs nor actually knew how much of our customer base was affected.

Having almost nothing to start with rather than just a strong desire to resolve this problem we've started with gathering direct information from affected users. Also we were monitoring the adoption rate of iOS 14 among our customers and according to what we've got only a small subset of users was affected. Few:)

By taking a closer look at what was added or changed in iOS 14 we've detected that the most likely cause of these crashes was local network permission that Apple introduced as a new security and privacy enhancement: [https://developer.apple.com/videos/play/wwdc2020/10110/][apple-developer-forum].

We actually weren't able to gather exact evidences that this permission caused app to crash. Since Crashlytics didn't give us any useful data we hoped that we can get some insights from TestFlight or at least will be able to obtain system logs from affected users (Settings → Privacy → Analytics & Improvements → Analytics Data). But we were out of luck-no crashes in system logs as well. This led us to think that probably system was killing the app for some very specific reason and this must be something new introduced in iOS 14.

Current iOS 14 behavior doesn't allow developers to directly control when local network permission request dialog should be presented. Instead system shows this dialog when application tries to access some local network resources for the first time.

![image1]({{"/assets/images/image2021-4-7_9-48-24.png" | absolute_url}})![image2]({{"/assets/images/image2021-4-7_9-48-35.png" | absolute_url}})

Based on what Apple described on corresponding WWDC talk and based on where our application logs were cut off we've managed to narrow down the possible root cause of the issue to the following assumptions:

- System was killing the app when it was attempting to register with our SIP server for placing and receiving VoIP calls
- Attempt to access local network resources could be possible if app was trying to contact local network DNS servers

Armed with the above assumptions we've started working on possible solutions to try them out on affected users.

Solution 1: Forcing local network permission prompt to be shown

First we started looking for ways to force the system to show local network permission dialog so that user can accept it at the app startup. After searching for available options the following was found:

- We can trigger the permission by sending a dummy traffic to local network like suggested here: [link](https://developer.apple.com/forums/thread/663768 "https://developer.apple.com/forums/thread/663768") - this option however only worked for the clean installation of the app but didn't trigger permission dialog if the app was updated from previously installed version
- We can trigger the permission by broadcasting fake Bonjour service like suggested here: [link](https://stackoverflow.com/questions/63940427/ios-14-how-to-trigger-local-network-dialog-and-check-user-answer "https://stackoverflow.com/questions/63940427/ios-14-how-to-trigger-local-network-dialog-and-check-user-answer") - this option seemed to work well but the concerns were that this approach required Bonjour service to be defined in application configuration as well as additional entitlements might be needed. All of that most likely would lead to Apple rejecting our app submission as we weren't actually using a real local network service
- We can trigger the permission by accessing ProcessInfo hostName property [link](https://developer.apple.com/documentation/foundation/processinfo "https://developer.apple.com/documentation/foundation/processinfo") - this option suited us best as it was simple to implement (`just call _ = ProcessInfo.hostName`) and didn't require any additional services/features to be used in the app. Important thing to note here is that call to `ProcessInfo.hostName` should be made off the main thread otherwise it might hang the thread for a significant amount of time and doing so at app launch will lead to system killing the app due to exceeding allowed time to start

We've implemented the third solution and it worked well for us. By testing it with users who experienced stable crashes on wifi we've confirmed that accepting local network permission solves crashes issue. The only problem that remained with this approach was that after updating to the version with this fix users with crashes on wifi most likely would still experience another one crash upon first launch of the app because at this point permission will still not be accepted. So the app will crash for them although the permission dialog will be shown and after accepting it all subsequent app launches will work well.

With all of the above knowledge we've started looking into how this first crash could be avoided which led us to the second solution.

Solution 2: Avoiding system killing the app

After further investigation we've came across Sber engineers tweet [link in Russian](https://twitter.com/katleta3000/status/1374697785405112321 "https://twitter.com/katleta3000/status/1374697785405112321") describing pretty similar issue with app crashing on wifi but with Yandex.MapKit. They've determined that the crash was not actually a crash but app was forcibly terminated by the system. That was happening when application was trying to access DNS with local network permission not being accepted which actually proved our assumptions. In this situation the outgoing DNS requests were somehow modified leading to force closing the app. They've also came up with the fix that made the app to ignore a specific signal being sent to it by the system so basically app's process started to ignore some force close signals from system. We've adapted this fix to our application and it seemed to work pretty well without breaking things:

{% highlight swift %}
// MARK: - UIApplicationDelegate
    override init() {
 
        var newAction = sigaction(__sigaction_u: __sigaction_u(__sa_handler: SIG_IGN), sa_mask: 0, sa_flags: 0)
        sigemptyset(&newAction.sa_mask)
        sigaction(SIGPIPE, &newAction, nil)
        super.init()
    }
{% endhighlight %}

So by combining 1st and 2nd solutions we've got pretty robust fix for wifi crashes issue. Users without crashes will not notice any changes in app behavior, users with crashes will be able to accept local network permission right after app update without additional app crash.


[apple-developer-forum]: [https://developer.apple.com/videos/play/wwdc2020/10110/]