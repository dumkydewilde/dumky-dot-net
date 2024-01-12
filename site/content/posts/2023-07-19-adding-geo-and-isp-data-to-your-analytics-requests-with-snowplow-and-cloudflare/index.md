---
title: "Adding Geo and ISP data to your analytics hits with Snowplow and Cloudflare Workers"
date: "2023-06-18"
tags: 
  - Snowplow
  - Cloudflare
  - Workers
  - GA4
  - geo-location
  - ISP
description: "In this post we'll look at how to add geo and ISP data to your analytics hits with Snowplow and Cloudflare Workers, an approach that you can also re-use for GA4." 
---

Have you ever seen certain small towns in your analytics reports with thousands of visitors? Or maybe a lot of traffic from a country where your products don't even ship to? Google Analytics used to have a very nice feature that would allow you to see the Internet Service Provider (ISP) of the originating request. This made it easy to identify bot traffic and spam in your analytics account. That feature is no longer there, but we can still leverage that data and recreate the feature for ourselves in Snowplow. As you may know, I'm a [big fan](/posts/analytics-on-the-edge-server-side-request-tracking-and-cookie-setting-using-cloudflare-workers/) of [Cloudflare](/posts/fetching-ipv4-cidr-ranges-from-aws-gcp-azure-and-cloudflare-for-bot-detection-with-python/), especially their Workers. Cloudflare Workers are small functions that allow you to enrich or adjust an HTTP request. Using Cloudflare Workers we can enhance our request

{{< box important >}}
Storing location and IP/ISP data from users may not be allowed without proper consent. Don't store data you don't need and remember that you are responsible for your own compliance.
{{< /box >}}


## Setting up the Cloudflare Worker
We won't go over the exact details of setting up the worker as the [Cloudflare Docs](https://developers.cloudflare.com/workers/get-started/guide/) do a much better job. Once you have a worker set up we can add an event listener for requests to our Snowplow tracker, add the additional headers and forward them to our actual collector. Cloudflare provides a default `rf` object on the `request` variable. This object contains all the geographical information we need, like the country, region and city, but also information on the ASN. This is not the exact same as an ISP, as the AS stands for an 'Autonomous System' on the internet which multiple ISPs could share. However, in practice this can tell you for example if a request came from Google Cloud or AWS instead of a residential ISP.

```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url)
  const { pathname } = url

  // Change the path to what you want your collector to be.
  if (pathname.startsWith('/com.snowplowanalytics.snowplow/tp2')) {

    const forwardUrl = '{{ YOUR COLLECTOR URL HERE }}' 
    let headers = new Headers(request.headers)
    const cfHeaders = request.cf || {};
    headers.append("cf-asn", cfHeaders.asn)
    headers.append("cf-as-organization", cfHeaders.asOrganization)
    headers.append("cf-colo", cfHeaders.colo)
    headers.append("cf-country", cfHeaders.country)
    headers.append("cf-is-eu-country", cfHeaders.isEUCountry)
    headers.append("cf-city", cfHeaders.city)
    headers.append("cf-region", cfHeaders.region)
    headers.append("cf-region-code", cfHeaders.regionCode)
    headers.append("cf-continent", cfHeaders.continent)
    headers.append("cf-postal-code", cfHeaders.postalCode)
    headers.append("cf-metro-code", cfHeaders.metroCode)
    headers.append("cf-bot-management-verified-bot", cfHeaders.botManagement.verifiedBot)
    headers.append("cf-bot-management-score", cfHeaders.botManagement.score)

    const forwardRequest = new Request(forwardUrl, {
      method: request.method,
      headers: headers,
      body: request.body,
      redirect: 'manual',
    })

    const forwardResponse = await fetch(forwardRequest)

    // Copy the response headers from the forwarded response
    const responseHeaders = new Headers(forwardResponse.headers)

    // Modify the Cache-Control header to add max-age=0
    responseHeaders.set('Cache-Control', 'max-age=0, ' + headers.get('Cache-Control'))
 
    // Return the forwarded response with its original headers
    return new Response(forwardResponse.body, {
      status: forwardResponse.status,
      statusText: forwardResponse.statusText,
      headers: responseHeaders,
    })
  } else {
    const response = await fetch(request)
    return response
  }
}
```


## Processing the Geo and ISP data in Snowplow
The Snowplow enricher allows us to [add custom JavaScript enrichments](https://docs.snowplow.io/docs/enriching-your-data/available-enrichments/custom-javascript-enrichment/writing/). In this case we will use the `getDerived_contexts` function to fetch our existing headers context and add them to a header object. If the Cloudflare headers are present, we can then overwrite the geo and ISP fields in Snowplow. 

```javascript
function process(event) {
    let contexts = []
    const entities = JSON.parse(event.getDerived_contexts());

    if (entities) {
      let headers = {};
      entities.data.forEach((entity) => { 
        if (entity.schema.startsWith('iglu:org.ietf/http_header/jsonschema/1-0-0')) {
          headers[entity.data.name] = entity.data.value;
        }
      })

      if (headers['cf-country']) { 
        event.setGeo_country(headers['cf-country'] || '')
        event.setGeo_region(headers['cf-region-code'] || '')
        event.setGeo_region_name(headers['cf-region'] || '')
        event.setGeo_city(headers['cf-city'] || '')
        event.setIp_isp(headers['cf-as-organization'] || '')
      }
    }

    // Process additional contexts

    return contexts;
  }
```

## Final thoughts
I've shown you how to do this with Snowplow, but of course the same approach is valid for Google Analytics (GA4) or any other analytics tool that allows you to add in custom event data. Cloudflare makes it easy to capture requests on your own site and send them to wherever you need adding valuable data in the process.

