---
layout: single
title:  "Dealing with Crashes Caused by iOS 14 Local Network Permission"
date:   2021-06-17 15:09:44 -0700
categories: iOS Mobile
---
Hello from the Intermedia Unite Mobile team! Let's start with a short introduction: our team works on Unite mobile applications for iOS and Android. These apps are a key component of Intermedia Unite – Intermedia’s all-in-one voice, video, chat, file sharing and more unified communications as a service (UCaaS) offering.

In this article we wanted to share with you our experiences dealing with some weird iOS app crashes and how we fixed them. Hopefully what we learned can assist you should you run into any of the same issues.

It all started after iOS 14 had gone gold. Since its public release, we started receiving a number of complaints from customers that the Unite app began crashing at startup. We soon discovered that there was one common issue among all of the reports: the app was crashing only when connected to a wifi network. Also, in most cases, the app was crashing right after launch if the user was previously logged in or right after the login process. Interestingly enough, we weren’t able to reproduce the issue on any of our test or personal devices, nor on any of the wifi networks. In addition, we didn't receive any complaints from our beta testers. And on top of that, these crashes weren't detected by Crashlytics so we neither had any crash logs nor actually knew how much of our customer base was affected.

Having almost nothing to start with rather than just a strong desire to fix this problem we started by gathering direct information from affected users. And, thankfully, after taking a look at the adoption rate of iOS 14 among our customers (which was low) we realized we had a narrow window to try and figure this out and get it resolved.

By taking a closer look at what was added or changed in iOS 14, we detected that the most likely cause of these crashes was a new local network permission that Apple introduced as a security and privacy enhancement: [https://developer.apple.com/videos/play/wwdc2020/10110/][apple-developer-forum].

But since Crashlytics didn't give us any useful data to determine if this permission was indeed the source of the issue, we hoped that we could get some insights from TestFlight or at least would be able to obtain system logs from affected users (Settings → Privacy → Analytics & Improvements → Analytics Data). 
But we were out of luck - no crashes recorded in the system logs. 
This then led us to think that the system was probably killing the app due to something else that was introduced in the iOS 14 release.

Current iOS 14 behavior doesn't allow developers to directly control when the local network permission request dialog should be presented. Instead, the system shows this dialog when the application tries to access local network resources for the first time.

![image1]({{"/assets/images/image2021-4-7_9-48-24.png" | absolute_url}})![image2]({{"/assets/images/image2021-4-7_9-48-35.png" | absolute_url}})

Based on what Apple described in a corresponding WWDC talk, and where our application logs were cut off. we managed to narrow down the possible root cause of the issue to the following:

- System was killing the app when it was attempting to register with our SIP server for placing and receiving VoIP calls
- Attempt to access local network resources could be possible if app was trying to contact local network DNS servers

Armed with these possibilities, we next worked on methods to test these assumptions on affected users.

Solution 1: Forcing local network permission prompt to be shown

First, we started looking for ways to force the system to show the local network permission dialog so that the user could accept it at app startup. After searching for available options, the following was found:

- We could trigger the permission by sending dummy traffic to the local network as described here: [link](https://developer.apple.com/forums/thread/663768 "https://developer.apple.com/forums/thread/663768") - this option however only worked for the clean installation of the app but didn't trigger the permission dialog if the app was updated from a previously installed version;
- We could trigger the permission by broadcasting a fake Bonjour service as described here: [link](https://stackoverflow.com/questions/63940427/ios-14-how-to-trigger-local-network-dialog-and-check-user-answer "https://stackoverflow.com/questions/63940427/ios-14-how-to-trigger-local-network-dialog-and-check-user-answer") - this option seemed to work well but the concerns were that this approach required the Bonjour service to be defined in the application configuration and additional entitlements might also be needed. Taken together, this most likely would lead to Apple rejecting our app submission as we weren't actually using a real local network service;
- We could trigger the permission by accessing ProcessInfo hostName property [link](https://developer.apple.com/documentation/foundation/processinfo "https://developer.apple.com/documentation/foundation/processinfo") - this option suited us best as it was simple to implement (just call _ = ProcessInfo.hostName) and didn't require any additional services/features to be used in the app. An important thing to note - call to ProcessInfo.hostName should be made off the main thread otherwise it might hang the thread for a significant amount of time and doing so at app launch will lead to the system killing the app due to exceeding the allowed time to start.

We implemented the third solution and it worked well for us. By testing it with users who experienced stable crashes on wifi we confirmed that accepting local network permission solves the crash issue. The only issue that remained was that after updating to the version with this fix, users who had experienced crashes on wifi would most likely still experience one more crash upon first launch of the app since, at this point, the permission would still not have been accepted.  So the app will crash for them although the permission dialog will be shown and after accepting it all subsequent app launches will work well.

But what about avoiding this first crash all together?  This led us to a further solution.

Solution 2: Keeping the system from killing the app

We came across a tweet from Sber engineers [link in Russian](https://twitter.com/katleta3000/status/1374697785405112321 "https://twitter.com/katleta3000/status/1374697785405112321") describing a similar issue with an app crash on wifi, in this case with Yandex.MapKit. They determined that the crash was not actually a crash but that the app was instead being forcibly terminated by the system. This was happening when the application was trying to access DNS, but the local network permission was not being accepted. This actually proved our assumptions. In this situation the outgoing DNS requests were somehow being modified leading to a force closing of the app. To solve, they came up with a fix that made the app ignore a specific signal being sent to it by the system so basically the app's process started to ignore some force close signals from the system. We adapted this fix to our application and it seemed to work well without breaking anything:

{% highlight swift %}
// MARK: - UIApplicationDelegate
    override init() {
 
        var newAction = sigaction(__sigaction_u: __sigaction_u(__sa_handler: SIG_IGN), sa_mask: 0, sa_flags: 0)
        sigemptyset(&newAction.sa_mask)
        sigaction(SIGPIPE, &newAction, nil)
        super.init()
    }
{% endhighlight %}

So, by combining the 1st and 2nd solutions we arrived at a pretty robust fix for the wifi crash issue. Users without crashes will not notice any changes in app behavior, users with crashes will be able to accept the local network permission right after the app update without experiencing an additional app crash.


[apple-developer-forum]: https://developer.apple.com/videos/play/wwdc2020/10110/