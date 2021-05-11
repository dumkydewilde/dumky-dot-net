---
title: "Cookie Consent: A Simple Checklist for Compliance"
date: "2021-04-07"
tags: 
  - "avg"
  - "consent"
  - "cookies"
  - "gdpr"
  - "itp"
description: "With terms like GDPR, Cookies, ePrivacy Directive, ePrivacy Regulation, ITP, ETP, most people's heads will start spinning. Nonetheless, not complying with privacy regulation can be a costly business in multiple ways. Find out what's what and how to check if your website is compliant."
---

With terms like GDPR, Cookies, ePrivacy Directive, ePrivacy Regulation, ITP, ETP, most people's heads will start spinning. Nonetheless, not complying with privacy regulation can be a costly business in three ways.

- You can be fined for non-compliance. And, yes, regulators are actually handing out fines, despite anything your local 'marketeer' might say. And these fines can range anywhere from €15.000-€15mln (and more if you're a platform or big player)
- You might disappoint your customers for not being honest with them, appearing dishonest because of a faulty implementation, or unnecessarily tracking them, or too much information about them without their _actual_ consent (more than just an absentminded click on 'proceed')
- It can cause downtime for you (or your app) now that other players like Apple are also cracking down on organisations and apps that mislead customers in their tracking capabilities.

So time to assess your cookie consent implementation with a simple glossary of definitions, a simple checklist, some best practices and actual examples of what's good and what's bad. I'll go over some of the implementation steps to check cookie consent for commonly used tools like Google Tag Manager and Google Analytics.

_Do note that I'm (luckily ;-) ) not a lawyer and this is intended as a rough guideline only. Always discuss the finer details with a legal counsel._

## Definitions

First, the boring stuff. I find that a lot of the confusion comes down to misunderstanding of the various applicable legislative documents combined with a misunderstanding of what role browsers play.

### Cookie

The term 'cookie' is explicitly used in the European ePrivacy directive from 2009, therefore called the 'cookie-law'. It was the origin of all the consent banners we now click away blindly. The term 'cookie' however, doesn't _only_ mean cookies that are set in the browser. It also designates other means of browser storage, like `localStorage` or tracking scripts in a broader sense.

Nowadays we can distinguish 'cookies' in three main categories:

- functional/preferences cookies (always allowed): those that are used for preferences like language settings, keeping items in a shopping cart, or for logging in.
- analytics cookies (sometimes allowed): those used for collecting data on user behaviour. Note that anonymised, aggregated analytics data is allowed in some EU countries (e.g. The Netherlands) but not in others (e.g. Germany).
- marketing/cross-site tracking cookies (never allowed without consent):

### GDPR (General Data Protection Regulation)

The GDPR is the European legislation around capturing consent for data collection and storage in the broadest possible way. According to the GDPR capturing consent should be non-ambiguous and applies to any party in their dealing with European end-users.

### AVG / DSGVO

The AVG and DSGVO are the Dutch and German 'implementations' of the GDPR because the GDPR itself is not centrally enforced by the EU. There is some room for 'interpretation' for each country under the GDPR, which is why a 'child' in Belgium is younger than 13 years old and younger than 16 years old in the Netherlands. In The Netherlands, in contrast to Germany, anonymised, aggregated analytics data can be collected without consent.

### ePrivacy Directive (ePD)

The 2009 ePD is the more specific (non-binding) legislation around (online) data processing. It explicitly calls for the duty to identify the purpose of a 'cookie' and giving users the option to opt-out.

### ePrivacy Regulation (ePR)

The ePR was originally intended to come into action along with the GDPR in 2018, but it looks like we'll have to wait until at least 2021. The ePR is the centralisation and update of ePD and includes the possibility of a central EU legislative body to be able hand out fines. The ePD will also include other means of online communication like VoIP (e.g. Skype) and messaging (e.g. WhatsApp). On top of that there will be a possibility for software providers (i.e. browsers) to streamline cookie consent implementations with a single opt-in, though it remains to be seen if these 'software providers' are willing to play along.

## Checklist: Cookie Consent Compliance

The checklist below allows you to do a quick check on both how you _capture_ a user's consent as well as how it's _implemented_. For the implementation I'm assuming you're using some commonly used tools like Google Tag Manager and Google Analytics. I'll use and refer to the Dutch context mostly, as I'm most familiar with that.

### Consent Capture

To comply with legal requirements you'll have to check for the following.

- There is an option to decline cookies that do not fall under the category of 'necessary' cookies.
- There are no pre-ticked checkboxes for consent (i.e. there will only be _explicit_ consent)
- There is no 'extra' step to opt-out, e.g. by going through a 'settings' option
- Content is available without explicit consent. (E.g. a cookie wall is not allowed)
- Content is available _independent_ of consent level. E.g. "opt-in to receive our free e-book" is not allowed.
- It is possible to adjust the level of consent afterwards. A 7-step how-to on clearing cookies from your browser is _not_ the same as being able to adjust your consent level.
- There is a comprehensive list of which cookies will be placed with which consent level.

### Consent Implementation

- Only functional cookies or cookies for anonymised analytics are placed before consent is captured.
- No requests to third parties are sent (which can potentially place cookies) without the proper consent
- When using third parties, like Google Analytics, consent levels are not mixed without proper consent. E.g. Google Analytics hits are not forwarded to Google Ads without marketing consent. Check that your Google Analytics Settings in GTM contain the `allowAdFeatures` field to determine marketing consent.
- Analytics hits before consent are properly anonymised. E.g. use the `anonymizeIp` field in your Google Analytics settings in GTM.
- If you're using GTM for managing consent levels make sure that there are no non-functional, third-party scripts in the source code. E.g. look for HTML tags like `<script src=”3rdpartytool.com/tracker.js”>` or `<img src=”socialmediaplatform.com/insights.gif” />`.
- Consent levels are clearly indicated for everyone who uses the tag management tool (e.g. GTM), to prevent future confusion. In GTM, for example, this can be done by using exception triggers to prevent marketing or analytics tags from firing. In GTM360 you can for example block entire zones without proper consent.

## Best Practices

- If you're allowed to use anonymised analytics with implicit consent (e.g. in the Netherlands), use this to better understand your opt-in rates for consent
- Make your opt-in as easy as your opt-out. That's a good general guideline for complying with the law. A little nudge is alright, like a button versus a text-link, but it should not limit a user's ability to opt-out.
- Store the consent level of your user (with the proper consent of course) in your analytics, CDP or CRM tool, so you can segment your users based on the proper consent and exclude users without proper consent from marketing campaigns for example.

## Resources

- \[Dutch\] The ACM on when and [how to legally use cookies](https://www.acm.nl/nl/onderwerpen/telecommunicatie/internet/cookies)
- \[Dutch\] The AP on [regulations for placing cookies, types of cookie consent banners that are allowed and the types of (GDPR) consent that are allowed](https://autoriteitpersoonsgegevens.nl/nl/onderwerpen/internet-telefoon-tv-en-post/cookies#wat-zijn-de-belangrijkste-regels-voor-het-plaatsen-van-cookies-2131)
- The actual GDPR legislation is [quite readable, especially the section on principles](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32016R0679&from=EN#d1e1797-1-1).
