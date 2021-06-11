---
layout: single
title:  "Dealing with iOS 14 Local Network Permission"
date:   2021-06-10 15:09:44 -0700
categories: iOS Mobile
---
Since the public release of iOS 14 we started getting a number of complaints from customers that apps were crashing for them on wifi networks. In most cases apps were crashing right after launch if users were previously logged in or right after they completed login process. The issue wasn't reproducible on our test devices and in our own wifi networks.

By studying what was added or changed in iOS 14 we've detected that the most likely cause of these crashes was local network permission that Apple introduced as a new security and privacy enhancement in iOS 14. More information about permission can be found here: [https://developer.apple.com/videos/play/wwdc2020/10110/][apple-developer-forum]

We weren't able to gather exact evidences that this permission actually caused app to crash. Mainly due to missing crash logs either in Crashlytics or even in TestFlight and system logs on device (Settings → Privacy → Analytics & Improvements →Analytics Data). 

Current iOS 14 behavior doesn't allow developers to directly control when local network permission dialog should be presented. Instead system itself shows this dialog when application first tries to access some local network resources.

![image1]({{"/assets/images/image2021-4-7_9-48-24.png" | absolute_url}})![image2]({{"/assets/images/image2021-4-7_9-48-35.png" | absolute_url}})

Solution 1: Forcing local network permission prompt to be shown

After searching for available options the following was found:

- We can trigger the permission by sending a dummy traffic to local network like suggested here: [link](https://developer.apple.com/forums/thread/663768 "https://developer.apple.com/forums/thread/663768") - this options for some reason only worked for the clean installation of the app but didn't trigger permission dialog if the app was updated from previously installed version
- We can trigger the permission by broadcasting fake Bonjour service like suggested here: [link](https://stackoverflow.com/questions/63940427/ios-14-how-to-trigger-local-network-dialog-and-check-user-answer "https://stackoverflow.com/questions/63940427/ios-14-how-to-trigger-local-network-dialog-and-check-user-answer") - this options seemed to work well but the concerns were that this approach required Bonjour service to be defined in application configuration as well as additional entitlements might be needed. All of that most likely would lead to Apple rejecting our app submission as we were using functionality (Bonjour) in an inappropriate way (for triggering local network permission instead of actually broadcasting information about service hosted by device)
- We can trigger the permission by accessing ProcessInfo hostName property [link](https://developer.apple.com/documentation/foundation/processinfo "https://developer.apple.com/documentation/foundation/processinfo") - this option suited us best as it was simple to implement (just call _ = ProcessInfo.hostName) and didn't require any additional services/features to be used in the app. The issue that we've discovered with this approach was that first call to ProcessInfo.hostName for some reason took about 20 seconds to complete and doing this on main thread just completely hanged the app at launch leading to a termination by the system due to exceeding allowed time for application launch. We did a simple workaround by placing call to ProcessInfo.hostName on the background queue which solved the issue. What's interesting is that local network permission dialog was triggered immediately without waiting for the ProcessInfo.hostName call to complete

We've implemented the third solution and it worked well for us. By testing it with users who experienced stable crashes on wifi we've confirmed that accepting local network permission solves crashes issue. The only problem that remained with this approach was that after updating to the version with this fix users with crashes on wifi most likely would still experience another one crash upon first launch of the app because at this point permission will still not be accepted. So the app will crash for them although the permission dialog will be shown and after accepting it all subsequent app launches will work well.

With all of the above knowledge we've started looking into how this first crash could be avoided which led us to the second solution.

After further investigation we've came across Sber engineers tweet describing pretty similar issue with app crashing on wifi but with Yandex.MapKit. They've determined that the crash was not actually a crash but app was forcibly terminated by the system. That was happening when application was trying to access DNS with local network permission not being accepted. In this situation the outgoing DNS requests were somehow modified leading to force closing the app. They've also came up with the fix that made the app to ignore a specific signal being sent to it by the system so basically app's process started to ignore some force close signals from system. We've adapted this fix to our application and it seemed to work pretty well without breaking things:

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