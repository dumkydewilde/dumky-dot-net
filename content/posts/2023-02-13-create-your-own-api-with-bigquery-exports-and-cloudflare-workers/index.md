---
title: "Create your own API from BigQuery data in minutes with SQL exports and Cloudflare Workers"
date: "2023-02-07"
tags: 
  - "SQL"
  - "Cloudflare Workers"
  - "BigQuery"
  - "Google Cloud Storage"
  - "Javascript"
description: "Want to have data from BigQuery publicly available? Create a simple API with BigQuery scheduled queries, JSON exports 
and a Cloudflare Worker to map the right URL to the right data." 
---
How do you expose data in BigQuery with an API? I was recently looking for an easy way to do just that and make data from BigQuery publically available with little effort but still secure. The result of this effort is now available on my [analytics](https://www.dumky.net/analytics/) page and if I want a JSON response with my BigQuery data, for example pageviews in the last 90 days all I have to do is call `/analytics/json/pageviews`. Is it magic? Not really, as I'll show you. But the services supporting this are close to magic. 

The whole process works as follows.
- I have a [dataset in BigQuery from my Snowplow tracker](https://www.dumky.net/posts/dbt-in-a-box-using-google-cloud-run-and-bigquery-to-run-your-dbt-sql-models-from-a-docker-container/).
- I run a **scheduled query** in BigQuery everyday to export the data for the API
- The export is stored as **JSON files in Google Cloud Storage**
- A **[Cloudflare Worker](https://workers.cloudflare.com/)** retrieves the relevant data for every endpoint (e.g. pageviews, referrers, etc.)
- The Cloudflare Worker is mapped to my own domain at the path `/analytics/json/*` . 

And voila! We have ourselves an API (You can see for yourself at [/analytics/json/pageviews](https://dumky.net/analytics/json/pageviews)). But alright, now in a little bit more detail.

## BigQuery Exports
BigQuery makes it super easy to export any query. For example, exporting the last 90 days of pageviews to JSON is as simple as:
```SQL
EXPORT DATA
  OPTIONS (
    uri = 'gs://<my-storage-bucket>/json/pageviews/*.json',
    format = 'JSON',
    overwrite = true
    )
AS (
  SELECT 
    DATE(start_tstamp) date,
    COUNT(*) pageviews
  FROM `snowplow_dbt.snowplow_web_page_views` 
  WHERE DATE_DIFF(CURRENT_DATE(),DATE(start_tstamp),DAY) =< 90
  GROUP BY date
  ORDER BY date DESC
);
```

After creating a new bucket in Google Cloud Storage with public access enabled, from the BigQuery UI we can just take that query —or any query— and click "Schedule" to create a new scheduled query. Depending on how frequently you want to update and access the data you may have to adjust the [caching settings](https://cloud.google.com/storage/docs/caching#built-in_caching_for) on Cloud Storage (default is 1 hour).

## Building a static file API with Cloudflare Workers
Building an API with endpoints is a little bit more complex, but nothing you can't handle. I'm using Cloudflare Workers, because it's fast, does caching, is cheap (as in  free for my purposes) and integrates very well with the rest of my site and domain management. In theory you could also use Google Cloud Functions to get similar functionality. 

The function really needs to do only two things: 
- Fetch a JSON file
- Return the right JSON response for the right endpoint

As you can see our `gatherResponse` function handles the retrieval of the JSON data. The same JSON data we just exported with our query above. Next, the `handleRequest` function will check the path name of our request and depending on the path we fetch a different JSON file. Finally we can respond with a slight transformation as we turn our ND-JSON (one object per line in the file) into a proper JSON response by wrapping it in an array (`[...]`) and changing the line returns into commas. 

```javascript
// Retrieve file function
async function gatherResponse(response) {
    console.log(response)
    const { headers } = response;
    const contentType = headers.get('content-type') || '';
    if (contentType.includes('application/json')) {
      return JSON.stringify(await response.json());
    }
    return response.text();
  }
  
async function handleRequest(request, env) {
	const baseURL = `https://storage.googleapis.com/${env.STORAGE_BUCKET}`
    const { pathname } = new URL(request.url);
    const init = {
      headers: {
        'content-type': 'application/json',
        'Access-Control-Allow-Origin': '*',
        },
      };
      
  
    if (pathname.startsWith("/analytics/json/pageviews")) {
      const url = `/${baseURL}/json/pageviews/000000000000.json`
      const response = await fetch(url, init);
      const results = await gatherResponse(response);
      return new Response("["+results.trim().split(/\r?\n/)+"]", init);
    
    } else if (pathname.startsWith("/analytics/json/referrers")) {
      const url = `/${baseURL}/json/referrers/000000000000.json`
      const response = await fetch(url, init);
      const results = await gatherResponse(response);
      return new Response("["+results.trim().split(/\r?\n/)+"]", init);
    
    } else if (pathname.startsWith("/analytics/json/topposts")) {
      const url = `/${baseURL}/json/topposts/000000000000.json`
      const response = await fetch(url, init);
      const results = await gatherResponse(response);
      return new Response("["+results.trim().split(/\r?\n/)+"]", init);
    
    } else {
      return new Response("Not found", {status: 404})
    }
  }
  
export default {
    async fetch(request, env) {
      return await handleRequest(request, env).catch(
        (err) => new Response(err.stack, { status: 500 })
      )
    }
  }
```