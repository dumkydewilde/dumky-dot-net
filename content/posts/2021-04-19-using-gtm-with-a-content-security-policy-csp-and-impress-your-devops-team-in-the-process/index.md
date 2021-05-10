---
title: "Using GTM with a Content Security Policy (CSP) and impress your DevOps team in the process"
date: "2021-04-19"
tags: 
  - "gtm"
  - "security"
description: "The internet is a beautiful place. If you think chaos is beautiful, that is, because it is also a place where everyone and everything is hacked, abused, and manipulated for money, status or just the lolz. To prevent your precious Google Tag Manager implementation —and your entire site for that matter— from falling victim to malicious code taking over checkout funnels or secretly listening to form input from visitors it's time to implement a Content Security Policy (CSP)."
---

The internet is a beautiful place. If you think chaos is beautiful, that is, because it is also a place where everyone and everything is hacked, abused, and manipulated for money, status or just the lolz. To prevent your precious Google Tag Manager implementation —and your entire site for that matter— from falling victim to [malicious code taking over checkout funnels](https://en.wikipedia.org/wiki/Web_skimming) or secretly listening to form input from visitors it's time to implement a Content Security Policy (CSP). At the end of this you should understand:

- What a CSP is and how it helps to protect your site's visitors
- How a CSP impacts images, stylesheets, and of course JavaScript.
- The security risk that Google Tag Manager poses to your site, and how using GTM with a Content Security Policy (CSP) can prevent your visitors from malicious code
- How to prevent Custom JavaScript variables from returning `undefined` in GTM
- What a 'nonce' is, what a 'hash' is and how nonces and hashes can help you implement GTM and other scripts securely.
- How to take away any pain points from your developers when implementing scripts alongside a CSP.

## To serve and protect

Google Tag Manager is a great way to serve a random collection of scripts and images (tags) that help you do all sorts of things like tracking purchase conversions for Facebook Ads, tracking funnels with Google Analytics or even [send Slack messages when a user sees a 404 page](https://www.youtube.com/watch?v=xCcZwpnGQZY). Your browser, the beacon of light trying to protect you from all the bad out there in the digital world, has no way to distinguish those 'good' scripts from the 'bad' scripts trying to steal your credit card information.

As site owners we can help the browser protect you from malicious code by telling it how to deal with code that connects to anything other than the current document being loaded. That 'other' thing could be your self-hosted `cats.gif` image you'd like to show the world or the Google Analytics tracking code to see how many people have looked at your cat GIF. A Content Security Policy to deal with this highly advanced Cat GIF page could look something like this:

```xml
default-src 'self'; img-src https://www.google-analytics.com; script-src https://www.google-analytics.com;
```

You will find the CSP:

- In your browser's network tab, under the headers for the request to load the page.
- In a meta tag in the actual HTML of the page `<meta http-equiv="Content-Security-Policy" content="" >`

When we use the above CSP, we'll find that it blocks everything except:

- Any resource loaded from the same domain
- Images (`<img src="" />`) loaded from the domain `https://www.google-analytics.com`
- Scripts loaded from `https://www.google-analytics.com`

This works quite well if you want to do basic tracking with Google Analytics. You might know however, that GA can also send POST requests and use the browser's beacon function to send information more reliably. To do that we'll also have to add

`connect-src https://www.google-analytics.com`

That way also so-called XHR requests are allowed.

That's still not enough to fully run Google Analytics though, because, as you might know, GA actually asks you to add a piece of code to your site:

```javascript
<!-- Global site tag (gtag.js) - Google Analytics -->
 <script async src="https://www.googletagmanager.com/gtag/js?id=UA-000000-1"></script>
 <script>
   window.dataLayer = window.dataLayer || [];
   function gtag(){dataLayer.push(arguments);}
   gtag('js', new Date());
   gtag('config', 'UA-000000-1');
 </script>  
```

By default, using `default-src 'self'` also means that nothing between `<script></script>` tags is executed. This is a good thing because if malicious code was somehow injected into our page it wouldn't run.

You will sometimes see the argument `'unsafe-inline'` added to a CSP. For example `script-src 'unsafe-inline' https://www.google-analytics.com;` will allow you to run the Google Analytics script above. It will also allow you to run [code to start a crypto-miner on the visitors computer](https://www.bbc.com/news/technology-43025788). In other words, though slightly better than nothing, it's still bad.

## How to implement Google Tag Manager (GTM) with a Content Security Policy (CSP)

As we could see in the example above, implementing a CSP and making sure all your tags are still firing is not as easy as adding `script-src https://www.google-analytics.com https://www.googletagmanager.com;`. Even if you use the `'unsafe-inline'` argument, you'll still find that although GTM seems to work, something annoying is happening: custom JavaScript variables are not working and returning `undefined`.

This has to do with how custom JavaScript variables are executed by GTM. To make it work we would have to add the following argument to your CSP (DON'T DO THIS, IT IS BAD AND WILL NEGATE YOUR CSP): `unsafe-eval`. This will allow any script to dynamically execute any other script and basically render your CSP useless. Instead there are other ways to execute inline scripts on the page.

### Execute inline script with a CSP using a hash

One way to make sure that a third party script is safe to execute is when you know the origin is safe (for example it comes from a trusted content delivery network like Google's or Cloudflare's) and you are absolutely sure it was not modified on the way to your site. You can make sure of the latter by verifying the integrity of the script with a hash. Any modification, even adding a space, will alter the hash and render the script invalid.

This process, called SubResource Integrity (SRI) is great for commonly used scripts like jQuery, Font Awesome, or Bootstrap. For example if we want to add Font Awesome to our site, we can just add it with the integrity attribute and be done.

`<link href="[//cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css](https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css)" rel="stylesheet" integrity="sha384-wvfXpqpZZVQGK6TAh5PVlGOfQNHSoD2xbE+QkPxCAFlNEevoEH3Sl0sibVcOQVnN" crossorigin="anonymous" />`

As you can see the resource is also versioned (4.7.0), so we know that as long as we use the same version, we will have the same script and thus the same hash.

### Execute inline script with a CSP using a nonce

Think about GTM and what it does for a moment. It allows us to create a custom set of tags and variables, adjust that as we go and load that on our site. By definition GTM will be a different script every time we hit the publish button, so using hashes is out of the question. We can however use a so-called 'nonce' a number-used-once. By adding this 'nonce' to our script tag and then add the same nonce to our CSP header we can tell the browser that the contents of that script tag are safe and can be executed.

`<script nonce="as3d54h3sdf13a4f">`

`script-src nonce-as3d54h3sdf13a4f`;

This implementation is the recommended implementation of using Google Tag Manager with a Content Security Policy. GTM even has [a 'nonce aware' script version](http://* https://developers.google.com/tag-manager/web/csp) that you'll have to add to your page instead of the default code, so that also custom JavaScript variables can be executed.

It is good to know however that once you go nonce there's no way back and [the `'unsafe-inline'` argument will be invalidated](http://* https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src). That means you'll have to be prepared for a discussion with your developer as they might be using inline scripts and will say that _"your update made our site crash"._

## Analysts ❤️ Developers: How to help your developer set up a CSP for the entire site

When removing or invalidating the `'unsafe-inline'` argument from your CSP you might get some push back from your developers. First of all it's good for them to know that their site is getting more secure since, as they themselves have noted, it is no longer possible to execute random code.

The problem is that most of the time they will use a framework that doesn't work with a CSP out of the box. For example [for React you'll have to set a build variable called `INLINE_RUNTIME_CHUNK` to `false`.](https://create-react-app.dev/docs/advanced-configuration/)

> By default, Create React App will embed the runtime script into `index.html` during the production build. When set to `false`, the script will not be embedded and will be imported as usual. This is normally required when dealing with CSP.

AngularJS [has similar options](https://docs.angularjs.org/api/ng/directive/ngCsp).

If browser compatibility is an issue, you can add the `'unsafe-inline'` option as a fallback: newer browsers will ignore it when using nonces, older browsers will still have some protection and functionality when they do not recognise nonces.

Another problem you might run into is that developers love to cache as much as possible. In other words, they want to serve the same version of the site to everyone, but using a nonce (a unique number) implies serving a _different_ page to everyone. One way to solve this is by using [something like a Cloudflare service worker](https://scotthelme.co.uk/csp-nonces-the-easy-way-with-cloudflare-workers/), which stands between the user and the origin server to replace anything that has a 'nonce' placeholder, with the actual nonce.

## Resources and Final Thoughts

Website security is extremely complex and new exploits are uncovered frequently. CSP's are one way to mitigate those exploits. However, when you allow GTM to run freely on your site, anyone with access to GTM can do anything with your site. In other words, GTM becomes part of a potential [supply chain attack](https://en.wikipedia.org/wiki/2020_United_States_federal_government_data_breach). Too often I've seen user's like `nameless.agency@gmail.com` with unfeathered access to GTM. There's no way to know who's behind that or if the email address has been compromised.

- Security researcher Troy Hunt has [a great explainer on CSP's](https://www.troyhunt.com/locking-down-your-website-scripts-with-csp-hashes-nonces-and-report-uri/) and is in general a good guy to follow for security content
- Google's developer documentation on [implementing GTM with a CSP](https://developers.google.com/tag-manager/web/csp)
- Bounteous on [CSP's and GTM with some extra examples](https://www.bounteous.com/insights/2017/07/20/using-google-analytics-and-google-tag-manager-content-security-policy/).
- Square on [CSP's for single page applications](https://developer.squareup.com/blog/content-security-policy-for-single-page-web-apps/)
- [Generating nonces with Cloudflare workers](https://scotthelme.co.uk/csp-nonces-the-easy-way-with-cloudflare-workers/)
- A '[thought experiment' on dependency confusion and supply-chain attacks](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610)
- Some examples of [how GTM can be used in XSS attacks](https://blog.deteact.com/csp-bypass/). Be aware that whitelisting the GTM domain without your GTM ID and e.g. `unsafe-inline` means attackers can also use their own GTM container to inject and run code.
