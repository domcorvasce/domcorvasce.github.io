---
title: "On Apple Screen Time API"
date: 2023-07-12T08:44:51+02:00
draft: false
---

Recently I've been working on integrating [Apple Screen Time](https://developer.apple.com/documentation/screentime) into [Brave iOS](https://github.com/brave/brave-ios/pull/7639) in order to:

* track web pages usage to get a detailed breakdown in the Screen Time app
  ![Screen Time app](https://user-images.githubusercontent.com/61356846/248568291-4c07b189-607f-421d-bc80-cab7bf09ae39.jpeg)
* enforce app limits on web pages
  ![Screen Time limit reached view](/images/apple-screen-time-limit-view.png)

The implementation is fairly simple but I've learned a lot about Screen Time API and Safari's behaviour along the way.

### Screen Time API

Screen Time API actually consists of [three frameworks](https://developer.apple.com/videos/play/wwdc2021/10123/):

* [ManagedSettings](https://developer.apple.com/documentation/ManagedSettings) for access restriction
* [FamilyControls](https://developer.apple.com/documentation/FamilyControls) for parental control
* [DeviceActivity](https://developer.apple.com/documentation/DeviceActivity) for activity monitoring

It also includes [STWebpageController](https://developer.apple.com/documentation/screentime/stwebpagecontroller?language=swift) which allows you to implement the infamous *You've reached your limit* view you see in Safari when web content limits are set. Unlike the frameworks above, it does not require any additional [permission](https://developer.apple.com/documentation/uikit/protecting_the_user_s_privacy/requesting_access_to_protected_resources) to work.

```swift
import ScreenTime

let screenTimeController = STWebpageController()

let url: URL? {
    willSet {
        screenTimeController.url = newValue
    }
}
```

You add the controller as a subview, set `STWebpageController#url` whenever a user switches tabs, and you are ready to go!

To emulate Safari's behaviour, `STWebpageController#suppressUsageRecording` should be set to `true`
when the user is in incognito mode (see [Web content limits in Safari](#web-content-limits-in-safari)).

### Testing `STWebpageController` via the Simulator

The XCode Simulator does not fully implement the Screen Time API.
You can set app limits in the device settings app but the controller won't work correctly.

### Content & Privacy restrictions

iOS allows you to [restrict access to websites](https://support.apple.com/en-us/HT201304#web-content).
It will display a *You cannot browse this page* view whenever you visit a restricted website.
You might think you have to implement this feature yourself but you don't:
this functionality is built into WebKit, therefore it's already implemented on [all iOS web browsers](https://developer.apple.com/app-store/review/guidelines/#2.5.6).

### Web content limits in Safari

Although Safari displays the *You've reached your limit* view even for private tabs,
it does it only if you ran out of time while browsing the page in a public tab. It does not track web page usage in incognito mode.

Each Safari tab has its own `STWebpageController`
hence the "limit reached" view does not prevent you from switching between tabs.
