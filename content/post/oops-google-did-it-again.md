---
title: "Oops!... Google Did It Again"
date: 2023-07-29T15:49:24+02:00
draft: false
---

{{< youtube CduA0TULnow >}}

Computer scientists always consider the **worst case** scenario because it allows us to edge against the risk&mdash;or, well, certainty&mdash;that something will go wrong.

The [Web Environment Integrity](https://github.com/RupertBenWiser/Web-Environment-Integrity/blob/main/explainer.md) proposal championed by a [small group](https://github.com/RupertBenWiser/Web-Environment-Integrity/blob/main/explainer.md#authors) of Google employees is an example of what happens when you don’t consider the worst case scenario. The proposal aims to create a new Web API which would allow websites to attest the trustworthiness of a client before sending back content&mdash;kinda like a DRM for the web.

The browser asks the **attester**&mdash;most likely the OS backed by dedicated hardware such as [TPM](https://en.wikipedia.org/wiki/Trusted_Platform_Module) on PCs or [T2](https://en.wikipedia.org/wiki/Apple_T2) on Apple devices&mdash;to check the "integrity" of the client's device and generate a **low-entropy token** signed using the **attester's private key**. The token is then sent to the website which validates it using the **attester’s public key**. If the token checks out, the website delivers its content.

![Web Environment Integrity proposal](/images/wei.png)

The idea is not something new. Back in 2022, Apple implemented [Private Access Tokens](https://developer.apple.com/news/?id=huqjyh7k). Cloudflare uses PATs to [eliminate the need for CAPTCHAs](https://blog.cloudflare.com/eliminating-captchas-on-iphones-and-macs-using-new-standard/) when the client runs on Apple devices. The mechanism behind PATs is almost identical to WEI's. The main difference being that in PATs the **Attester** (Apple) and **Issuer** (Cloudflare, Fastly) are two different entities.

![Apple PATs with Cloudflare](https://blog.cloudflare.com/content/images/2022/06/Screen-Shot-2022-06-07-at-2.01.38-PM.png)

Why are [people](https://www.fsf.org/blogs/community/web-environment-integrity-is-an-all-out-attack-on-the-free-internet) [complaining](https://arstechnica.com/gadgets/2023/07/googles-web-integrity-api-sounds-like-drm-for-the-web/) about WEI and not PATs then?

Well, Google is simultaneously the owner of the most popular web browser (Chrome) and the most popular mobile operating system (Android) on top of which Chrome runs. WEI is guaranteed to be a recipe for anti-competitive practices.

The WEI attesters&mdash;["a relatively small group"](https://github.com/RupertBenWiser/Web-Environment-Integrity/blob/main/explainer.md#attester-level-acceptable-browser-policy)&mdash;would most likely be picked by Google. It'll make it hard, time-expensive, and most probably costly for new OSes, device vendors, and web browsers to get their foot in the door as they will have to go through Google.

Let's consider the **best case** scenario for a moment: Google doesn't abuse their power. Instead, they go out of their way to ensure everyone has equal access to this functionality. As people stop being bombarded with CAPTCHAs, social media websites cut down on fake engagement, and advertisers stop fingerprinting to ensure ads delivery to real humans, we start asking ourselves why WEI took so long!

Now back to reality: no one really cares about the best scenario though. It’s not realistic. Once you have power it's very tempting to abuse it, and it’s very hard to take it away from you. Google gained their monopoly in web browsing and we let them. We had and still have bigger problems in our lives than letting a company control which piece of software we browse the web on.

They are now in the position to go ahead and implement this proposal even if W3C disagrees.
What is left to us is to ask: is this CAPTCHA-free bot-less garden-walled world worth it?
