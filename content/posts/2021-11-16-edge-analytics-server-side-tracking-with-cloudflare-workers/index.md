---
title: "Analytics on the edge: server-side request tracking and cookie setting using Cloudflare Workers"
date: "2021-10-25"
tags: 
  - "google analytics"
  - "measurement protocol"
  - "cloudflare"
  - "serverless"
  - "cloudflare workers"
  - "dns"
  - "edge analytics"
  - "server-side tracking"
description: "" 
---
Server-side tracking is all the rage these days, but let me tell you about the uber-coolest kid on the blockchain: edge analytics. I'm kidding, there's no such thing as edge analytics (except maybe for IoT devices —story for another time—), but there is the possibility to intercept requests on the 'edge' of the network. Using Cloudflare Workers, you can send data to Google Analytics for all kinds of scenarios, even for users visiting pages THAT DON'T EVEN EXIST! 

Ok, so what kind of black magic is this? First of all we need to understand what it means to load a page in your browser and have Google Analytics track it. Simo Ahava has done a great job explaining how that entire chain works in [one of his recent podcasts](https://www.teamsimmer.com/2021/09/07/web-browsers-with-simo-ahava/): from requesting a page to resolving the DNS and eventually loading the page in the browser. What you need to know for now is that everytime you request a URL like `example.com` that piece of text will be translated or resolved to an IP address in the form of `1.2.3.4`. The service that translates the name for this site —`dumky.net`— to an actual IP address is called Cloudflare and Cloudflare provides this service for over 15% of the internet. Because they handle this request and have data centers all over the world —on the so-called edge of the network— to serve your website faster from a cache, they can also amend that request with a 'worker', a serverless function similar to AWS Lambda or Google Cloud Functions that allows you to run arbitrary scripts on the request. 

Workers can do things like combine multiple requests into one or translate pages on the fly, but in this case we'll use a worker to send a hit to Google Analytics via the measurement protocol as your page is being served. Normally the hit to Google Analytics is sent via JavaScript loaded on the page (client-side), but with the measurement protocol we can send hits server to server. Intercepting the request allows us to do a few interesting things. First of all not every request is to an actual page on our website. By tracking all requests we can get an understanding of whether something is wrong with our pages or if users expecting something that isn't there. Secondly, we are not even limited to tracking just pages, we can also track requests to images or other files that we find interesting, even when there's no analytics enabled for them. Thirdly, when we combine the server-side request logging with client-side tracking, we can get an understanding of how many users (and by users I mean people *and* bots) are actually loading the GA JavaScript library. In other words: how much of our traffic do we actually cover in our analytics tool?

## Building our first edge analytics worker
Let's get cracking. Here's the barebones function to both handle our request correctly for visitors as well as send a measurement protocol hit to GA. As you'll see it's NodeJS with some Cloudflare flavor. We listen for new requests and then run our `handleRequest()` function. This function will fetch the requested item and make a response to the requesting visitor, but it will also run some behind the scenes logic and send a payload to Google Analytics.

```javascript
const trackingId = "UA-ABCDE-FG";

async function handleRequest(event) {
  const res = await fetch(request); // Fetch the actual page the request was for
  newResponse = new Response(res.body, res);

  let uaData = {} // Object with data for GA Universal Analytics

  // ... here we'll combine all the data we need into our uaData object

  const payload = (Object.keys(uaData).map(k => {
    return `${k}=${encodeURI(uaData[k])}`
  }).join('&'));

  // Use 'waitUntil' to extend the FetchEvent lifetime
  event.waitUntil(fetch("https://www.google-analytics.com/collect?" + payload));       
  
  return newResponse
}

addEventListener("fetch", (event) => {
    // If our worker fails, pass the request through like normal
    event.passThroughOnException();
    return event.respondWith(handleRequest(event))
})
```

There are two important things to note. Firstly, if our script ever fails the request from the user will not be impacted as the `event.passThroughOnException()` line will make it seem like the worker was never there. Secondly, we don't want to negatively impact our response time. This is why we run our `handleRequest()` function asynchronously and respond as soon as possible. After the response the worker isn't immediately shut down, but instead it will keep running until the actions in`event.waitUntil(// Promise based actions));` are completed. In theory this allows us to even fetch the full page the user wants to see in a sort of side channel and read whatever we want from that page. For example, it allows you to read the title of a page from the `<title>` attribute, something that would be impossible with normal request logging. 


## Gathering data
So now that we understand a little bit about how we can hook onto a request to our site and how to use Cloudflare Workers to evaluate that request and simultaneously send a hit to Google Analytics, we can start to thing about what we actually can and want to send. Of course every request has the basics we need: URL, referrer, user agent. We can then combine that with some static information like the tracking ID, non-interaction status (since there's no actual user interaction yet), an event name and a source name —which I've added as custom dimension 4 in the example below. Then we get to some interesting stuff, because Cloudflare has a lot of extra information. I've added the country (instead of say, the full IP address) from the Cloudflare `cf-ipcountry` header. 

If you're on a Cloudflare plan that has bot management (not available in the free plan unfortunately) you can also add a bot score to the payload. And good news for [those who miss the 'network domain' and 'service provider' reports](https://www.seerinteractive.com/blog/deprecating-network-domain-service-provider/)! Cloudflare allows you to access the ASN (autonomous system number, the identifier for e.g. an ISP or internet exchange), which allows you to get that service provider information. Finally one of the most important pieces of information that you would never be able to grab with client-side analytics is the requests (and request statuses) that either fail or are for something other than an HTML page, like a PDF file or an image. In the example below we'll only look at html pages with status 200 (OK).

```javascript
        if(res.headers.get('content-type').indexOf("text/html") > -1 && res.status === 200)  {        
            const lang = request.headers.get('accept-language') ? request.headers.get('accept-language').split(",")[0] : null;
            const cfHeaders = request.cf || {};
            const botManagement = cfHeaders.botManagement || {};

            let uaData = {
                v: 1,
                tid: trackingId,
                ni: 1,
                t: 'event',
                ec: 'cf_worker',
                ea: 'request',
                el: res.status + ': ' + request.url,
                dl: request.url,
                dr: request.headers.get('referer'),
                geoid: request.headers.get('cf-ipcountry'),
                ul: lang,
                ua: request.headers.get('user-agent'),
                cd2: request.url,
                cd1: request.headers.get('user-agent'),
                cd4: 'cf_worker',
                cd5: botManagement.verifiedBot+'|'+botManagement.score+'|'+ cfHeaders.asOrganization,
                z: Math.random()
            }

            // Get the GA cookie if available, or create a new one
            const cookies = request.headers.get("Cookie") ? request.headers.get("Cookie").split(";") : [];
            const gaCookie = cookies.filter((c) => { return c.indexOf('_ga') > -1 });
            if (gaCookie.length > 0) {
                uaData['cid'] = gaCookie[0].match(/_ga=GA[0-9]\.[0-9].(.+)/)[1];
            } else {
                const rando = btoa(crypto.getRandomValues(new Uint32Array(1)));
                uaData.cid = `${rando}.${Date.now()}`
                const _ga = `GA1.2.${uaData.cid}`
                
                // Set client ID cookie
                newResponse.headers.set('Set-Cookie', `_ga=${_ga}; SameSite=Strict; Secure; Max-Age=${60*60*24*90}`);
            }
            
            // Set client ID as custom dimension
            uaData.cd3 = uaData.cid;

            ... // Send the payload

        }
```

If you've looked closely at the script, you've noticed that we skipped over a part. The cookies! Since Safari's Intelligent Tracking Prevention (ITP) measures from 2018 treat client-side (javascript) initiated storage differently from server-side initiated storage, server-side cookies are *en vogue* again —contrary to corduroy pants, those never were and never will be *en vogue*. In our example above we look for an existing Google Analytics client ID in the cookie header of the request. The client ID is the unique identifier for a device in your GA data and is stored as the `_ga` cookie in the user's browser. If there's no `_ga` cookie present it'll be generated client-side by the Google Analytics script. In this case we want to be ahead of that and generate the client ID ourselves from a random number + a timestamp. We'll also send that client ID along as a custome dimension so we can match it to any client-side hits if necessary.

And that's it for now. If you're not using Cloudflare you can sign up for their service for free and point your domain name to their domain name servers to get started. You can then set up your own worker and route your traffic through it as you wish. 