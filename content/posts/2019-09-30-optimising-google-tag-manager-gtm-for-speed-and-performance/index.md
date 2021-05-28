---
title: "8 ways to optimise Google Tag Manager (GTM) for speed and performance"
date: "2019-09-30"
tags: 
  - "analytics"
  - "gtm"
  - "page-speed"
  - "performance"
description: "Google Tag Manager makes it incredibly easy to add marketing tags to your site. From registering ads conversions and transactions to sophisticated tags that segment users based on the weather in their current location, you can go crazy without having to go back to your development team every time. But that doesn't mean you should do it all. While your dev and SEO teams are working hard to reach their pagespeed goals all the marketeers are having a proverbial party in their yard. Here you'll find a few tips to keep your GTM container lean and fast."
---

_UPDATE: with the [GTM server-side release](https://www.dumkydewilde.nl/2020/08/why-googles-new-gtm-server-side-tagging-solution-is-a-big-win-win-for-both-your-website-and-google/) I've added a [section](#8-server-side-tagging) on server-side tagging_

_UPDATE 2: There's a [number 9](#9-bonus-using-a-caching-proxy-to-load-gtm) that shows you how to serve GTM through a caching proxy to increase performance_

Google Tag Manager makes it incredibly easy to add marketing tags to your site. From registering ads conversions and transactions to sophisticated tags that segment users based on the weather in their current location, you can go crazy without having to go back to your development team every time. But that doesn't mean you should do it all. While your dev and SEO teams are working hard to reach their [pagespeed goals](https://developers.google.com/speed/pagespeed/insights/), all the marketeers are having a proverbial party in their yard. Here you'll find a few tips to increase performance, clean up tags and variables, remove unused javascript and keep your Google Tag Manager container lean.

## Best Practices

### 1\. Use a (custom) template

Tag manager provides a lot of standard tag templates like the Google Analytics tag or the Hotjar tag, but recently they've enabled you to [create your own template](https://www.dumkydewilde.nl/2019/06/gtm-custom-templates-how-to-think-about-building-your-own/). This not only helps in terms of a more convenient interface for working with tags and added security benefits, but there is also a performance benefit here. Every time you use a custom HTML tag instead of a template tag the GTM script will have to go through your page again insert the script and evaluate the actual script. A template on the other hand is inserted at the same time as the GTM script itself.

### 2\. Asynchronous everything

Synchronous scripts are executed linearly in the page, meaning that the loading of other scripts is halted until this part of the script is completed. This is good if that part of the script is crucial for loading the rest of the page, it's not so good if it's nothing more than dead weight blocking the rendering of your page.

If something is so crucial that other scripts have to wait for it, it shouldn't be in your Google Tag Manager. In other words: all your GTM script should run a-synchronously. This is the default for GTM itself and for the default and custom templates it runs. You can however screw up in your custom HTML tags and variables.

One use case I often see that is a great example of this is getting a visitor's IP address using a third party service and mapping that to a list of know (business) IP addresses so you can exclude your own traffic easily. When you call a third party service like that you cannot just request the information as it'll wait for the information to return and then continue loading the rest of the page. Instead you'll have to write your request to behave asynchronously.

```javascript
// DON'T DO THIS
<script src='random-ip-service.io/get-ip'></script>

// DO THIS INSTEAD
<script>
(function() {
    if (sessionStorage.getItem('ipAddress') === "string") {
        dataLayer.push({
            ip: sessionStorage.getItem('ipAddress')
        }); 
    }
    else {
        var xhr = new XMLHttpRequest();
        xhr.open("GET", "random-ip-service.io/get-ip", true);
        xhr.onload = function (e) {
        if (xhr.readyState === 4) {
            if (xhr.status === 200) {
            dataLayer.push({
                ip: xhr.responseText
            });
            sessionStorage.setItem('ipAddress',xhr.responseText);
            } else {
            console.error(xhr.statusText);
            }
        }
        };
        xhr.onerror = function (e) {
        console.error(xhr.statusText);
        };
        xhr.send(null);
    }
})()
</script>
```

There are two things going on here. First of all, yes, asynchronous JavaScript is stupidly long, but use it anyway. Secondly, in the first example a request is made on every page again and again. The likelihood of someone's IP address changing mid-session is near zero as is the business impact of not registering that change. So just save that address in the sessionStorage and don't call the third party service if you don't have to.

### 3\. Minimise the use of variables (and tags)

Here's another easy optimisation trick for you: only fire tags when necessary. This sounds simple enough, but it's so easy to pollute your tag space and fire all those tags on every page and event. But even when you have set them to fire on specific pages, do you fire them at page view? Why not load them on 'Window Loaded'? Unless it's crucial to track visitors that spend less than three seconds on your page, consider using 'Window Loaded' instead of 'Page View'

It's easy to not fire tags on every page, but did you know that variables are calculated on _every page_? Especially if you use complex calculations in your Custom JavaScript variables this can become a cumbersome process. A few things you can consider:

- Store calculations in local or session storage for quick retrieval
- Use if statements on e.g. document.location to only do the complex calculation when necessary
- Ask your developer to provide the variable server-side — Yes, I know you don't like talking to your developer, but do it anyway!

### 4\. Have a process in place for adding and removing tags

Yes, here's captain obvious again, but tell me, for how many containers do you have a process in place for adding and removing tags? Over time tags will accumulate and leave their waste (unused variables) behind when they're removed again. Proper documentation and a decision process for adding and removing tags will help you remove that dead weight. For example set a time limit on campaign tags by using the _'custom tag firing schedule'_ in the advanced settings of your tag.

If you do have a lot of tags in your container and you're unsure which ones are actually still necessary and which ones are dead weight, consider using the free tool that I built specifically for this purpose called [GTM Export Tools](https://gtm-export-tools.web.app/). It allows you to take an export of your GTM container and with one click remove all triggers and variables that are not actually in use anymore. If you're interested to see which tags are actually not in use anymore or are slow to run, consider building a [GTM Tag Monitoring solution](https://www.dumkydewilde.nl/2020/07/building-a-complete-tag-monitoring-solution-for-google-tag-manager/).

## Learn from the best: what your development team does to keep things running smooth

Though broadband connections have allowed us to care less about page speed, mobile use and the ominous Google algorithm have had site builders think about page speed (and thus user experience) for quite some time. And they have a few tricks we can use.

### 5\. Caching: Don't use what's already there

Page load time is taken up by two things: waiting for the server to respond and waiting for the browser to render and execute everything on the page. While there's a lot in your code you can optimise for rendering faster, sometimes the easiest step is to not request anything from the server. How's that possible? The magic word is 'cache'. Files are stored in the browser's memory and if they haven't changed since the last time they were requested from the server, they'll be served from the browser cache.

Now you might think that only works for returning visitors, but here's the trick. A lot of sites use the exact same scripts. They use the same files and javascript libraries, and over the years these are no longer served by individual websites, but by high performance data centers from Google and the like. That means that if you use something like jQuery or Bootstrap, the likelihood of a user already having the latest version of that library on their computer hiding out in their browser cache is very high. To access that file from cache, all you need to do is make a request to the same place it was accessed from before: [the Google server](https://developers.google.com/speed/libraries). In other words: don't request from your own server what can be requested from a common server.

### 6\. Minifying and bundling

In the battle for speed every byte counts. That is why developers have something called a _build process_ where certain actions are performed before moving an app or website to production. This can include certain tests to check if code is working properly, but it also includes two actions called bundling and minifying.

Minifying means making your files as small as possible so they'll load faster. That means not only removing comments out of your code, but also changing variable names from say, _productsInCart_, to just _p_, that's 13 bytes saved right there. The same goes for all those spaces, tabs and blank lines you don't need. It makes your code unreadable for humans, but hey, the machines don't care about humans.

Some vendors also allow you to select different versions of their libraries. For example, if you [load Facebook](https://developers.facebook.com/docs/javascript/advanced-setup/) with the parameter `xfbml` set to false like so `https://connect.facebook.net/en_US/sdk.js?xfbml=false` you leave out the social buttons saving ~140kb in the process.

Minification is often combined with bundling. Every separate script means a new call to the server. And every call has some amount of overhead. So by bundling all scripts together you can reduce the number of requests to your server and the overhead and time increase that comes with it.

## 8\. Server-side tagging

### Google Tag Manager Server-side

With the release of [GTM Server-side](https://www.dumkydewilde.nl/2020/08/why-googles-new-gtm-server-side-tagging-solution-is-a-big-win-win-for-both-your-website-and-google/) you now have the option of off-loading a lot of your tags to your server instead of sending it through the user's browser. Tools like Segment have been doing this for a long time, but the basic principle is this. Instead of sending hits to every advertising and analytics platform with every user interaction, you only send that hit once to your "own" tagging server and on that server you have your own logic to decide how you want to distribute the information from that hit.

So you send a button click event to your server with both a Google and a Facebook identifier and on your server you send a `buttonClick` event to Google with the relevant identifier as well as a `buttonClick` event to Facebook with the Facebook identifier. The performance benefit here is two-fold: you don't have to load the Facebook and Google libraries on your site _and_ your sending only one request from the user's browser.

## 9\. BONUS! Using a caching proxy to load GTM

I just keep adding stuff to the list, but this one is very nice to shave those final milliseconds of your page load. If you're using a CDN or service like Cloudflare you might be able to cache the GTM container there and load it through your own CDN. I wrote a piece on how to do exactly that [with Cloudflare Workers](/posts/speeding-up-gtm-with-a-caching-proxy-using-cloudflare-workers/).