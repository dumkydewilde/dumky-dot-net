---
title: "Why web analytics is still a mess in 2023"
date: "2023-02-26"
tags: 
  - analytics
  - reflection
  - 2023
description: "Web analytics still feels 'messy' in 2023. Why is it so hard to solve the problem of web analytics? Let's dive into some of the misconceptions that fuel the mess, like the ideas that websites are easy,  are visited by people, that web analytics is about tracking poeple, that we have all the tools we need, and that web analytics is actually important." 
---

How hard can it be to measure how many people visited your site? I recently spent a few days at a [fantastic (web) analytics conference](https://www.superweek.hu). After talking to so many smart people it suddenly struck me: we are not *solving* the problem of web analytics, we are solving *how to work around the problems*. Sounds too philosophical? Let me start with a picture first.

![[Webalizer]](images/webalizer-mrunix.png)*

For a long time web analytics looked like the Webalizer image above. A few default charts indicating the amount of traffic to your site. We solved the problem of web analytics, right? People view pages on your site. You see how many pages have been viewed. Done. 
Then how come web analytics teams keep doubling in size? How come we have agencies, consultants, conferences and an entire industry still discussing solutions to problems that once seemed solved? How can web analytics be as simple as the chart above and so complex at the same time? How come everyone can understand how a website works, yet every conversation on the analytics of that website ends with: "But why are these two numbers not the same?"

Unfortunately I am here to tell you that you will have to keep having that conversation for the forseeable future. There are a few crucial underlying problems, so fundamental to web analytics, that, as an industry, we'll have to Marie-Kondo this mess as much as we can, but at the same time accept the things we can not change (there are [plenty of support groups](https://www.meetup.com/analytics-engineering/) if you are in need) . Here are a few common misconceptions that might give you the idea web analytics is a solved problem, but can make the life of every web analyst a living hell.

_*Unfortunately I lost all my Webalizer data, but this image from mrunix.net gives you an idea._

## 1. Websites are easy so analytics is easy
The key question of how many people visited your site seems deceptively easy. It seems easy, because for one we think websites are easy. However, they are not. Organisations spend millions and millions to make websites *feel easy* to use. And yes, there is a lot of wasted money, and yes sometimes teams get it right the first time. But when you look at organisations that consistently get the online experience right, you'll see that they have big budgets to spend time and money on getting it right. Think of making a good website like building a highway: it will get you to your destination faster than a dirt road. Now think of web analytics like cutting your own path through the jungle. There is no map, only machete salesmen.

Websites used to be a page that would load once. Now a page loads once after it has been pre-rendered on the server to improve load times, then lazily loads all other content like images, reviews, and product information from a CDN, recommendations specifically tailored to you through AJAX calls to micro-services and third party A/B-testing tools that will change the way the page looks for you. The page is tracked both on the server, in the browser and by the third party tool. You'll get different results for all of them.

## 2. Websites are visited by people
Yes, people visit websites. Unfortunately they are not the only ones. There is an increasing amount of bot traffic, sometimes or for some periods up to 50-80% that renders any analytics tool that doesn't deal with bots immediately ineffective. The reason it's ineffective is that for the most part we don't care about these bots. Yet telling bots and people can be extremely hard. In practice I've seen and/or built bots that:
- Read your pages for Google search results
- Read your pages for search results that are not Google
- Read your page because someone shared your link and the service wants a preview.
- Read your pages to get price changes for your competitor
- Create fake accounts to sign up for discounts/connections/information/tickets/anything-valuable
- Scan for vulnerable parts of widely used systems that can be exploited
- Perform automate testing from the developers of the site
- Are part of the pet project of a computer science undergrad who learned about web scraping.
- ...

Even if you manage to filter out all this bot traffic, you are still stuck with another problem: people. 

## 3. Web analytics is about tracking people
People behave in weird ways and a big problem of web analytics is that the things that are important to you (revenue) are not how people use your site. They will (true stories):
- Leave tabs open forever
- Share accounts with family
- Share accounts with friends
- Use both your website and app at the same time
- Use a VPN so the same user appears in different places at the same time.
- Use an adblocker because their trust has been abused so many times
- Use an adblocker because their nephew installed it for them
- Access your site 400 times a day in one long session because it is on a tablet in a physical store
- Have their 3-year old click 5000 times because it relaxes them
- Read and copy the text 100 times a day even though they are an employee, just because it's the easiest.
- Use Safari so their cookies reset either every 1 or 7 days.
- Drive in and out of range of cell towers, so the connection drops or page loads are incredibly slow
- Use your website 50 times a day because they are on your sales team. 

Which one of these do you want information about? Which ones do you want to filter out? And how does their behaviour influence your averages, rates and KPIs? Yes, you want to track people more than bots, but not all people equally. In the end what matters is how behaviour on sites as part of a customer journey maps to real business outcomes. Because of that I am a big fan of [the concept of "tries"](https://www.analytics-ninja.com/blog/2021/07/customer-journeys-and-tries.html), that is: people will interact with your company differently depending on how often they've been through the same flow.

## 4. Web analytics is a solved problem
Yes, you can set up a webshop with Shopify or a blog with Wordpress in a few minutes and even get standardised reporting for it. Analytics tools will tell you their implementation is "just one line of code" and you'll be up and running in less than a minute. 
The problem is that (web) analytics is a solved problem in so far as your business is a solved problem. That is, the output of your analytics should map to value drivers and pain points of your business. Yes, for e-commerce you can probably re-use existing patterns and thus existing reports, but every business will target a niche or have a specific edge that is unique and might therefore need a unique analytics implementation. 
The hardest part of web analytics is not the implementation, it is mapping data collection (event tracking) to a data model that matches your business on a continuous basis. Yes, the first part, creating a data model can be hard (session vs. event based anyone?), but the latter part —doing it not just once, but continously— is even harder than the first part. Continuous analytics requires keeping track of what you have done, what others have done, how and where the business is growing, and validating and monitoring implementations.

## 5. We have all the tools we need
There has been a tremendous growth in the (web) analytics space over the last 10 years, and hypergrowth during COVID. We now have all the tools we need, except for the tools that integrate all the other tools. We have both too many tools and too little. Of course every good analytics implementation needs a tool to**:
- Keep track of your tracking plan (Avo)
- Track your tracking implementation (Cypress, Observe point)
- Keep track of your campaign tracking (Airtable)
- Keep track of your campaign tracking's tracking (e.g. utm.io)
- Manage consent (Onetrust, Cookiebot, Cookiehub, Usercentrics, Klaro, Quantcast, Trustarc, Cookiefirst, Osano, Cookieyes, ...)
- Manage your event tracking (GTM, Tealium, Segment)
- Extract business data (Fivetran, Stitch, Airbyte, Funnel, Talend, Supermetrics, ...)
- Transform your data (dbt, Dataform, LookML)
- Store your data (BigQuery, Snowflake, Databricks, Redshift, Synapse, Postgres, Athena, DuckDb, Clickhouse, Trino)
- Orchestrate the whole process (Airflow, Prefect, Dagster, Astronomer)
- Observe and monitor data going in and out (Masthead, SODA, BigEye, MonteCarlo)
- Reverse ETL (Census, Hightouch)
- Visualise your data (Metabase, Hex, Looker, Tableau, Mode, Power BI, Superset)

Oh, and did we mention you actually need an analytics tool? (Google Analytics, Snowplow, Adobe, Segment, Amplitude, Mixpanel, Heap, Fathom, Plausible, Matomo, PiwikPRO, Cloudflare Analytics, Simple Analytics)

So yes, there is probably a tool that solves your problem, but it will likely only solve part of your problem and will thus require integration with other tools and add complexity. On top of that, some problems are not always interesting enough for others to solve. The list of tools is minimal when it comes to, for example, tracking plans, data validation, naming consistency. Managing a stack of tools and the effects of their (non-)interaction will likely be part of your job.

_**NB: Mention of a tool does not mean endorsement of the tool_

## 6. Web analytics is important for organisations
I recently worked for a client that didn't even have a website. Their business was doing better than ever. It was very refreshing. Your website is part of your business and its importance will depend on the type of your business. Usually if you work with web analytics data you will overestimate the importance of that data. 
This doesn't mean web analytics can't be valueable and insightful. It means that web analytics is mostly a cost-center, and therefore might not receive the attention and funds that you would like it to have to do your work in the best way possible.
On top of that, the web analytics team can sometimes be a hot potato within an organisation: does it belong to marketing, communications or IT? What if all departments use it in different ways and have different priorities? Like [Conways law](https://en.wikipedia.org/wiki/Conway%27s_law), a web analytics tracking implementation will be the reflection of the owner of the implementation. Nonetheless, if done well, your analytics team will be a feedback loop for the development of your organisation. Whether you are developing or selling a product or service, the key to improve is to get insight into the effect of your actions in both qualitative (talk to your customers) and quantative ways.

## Final thoughts
So yes, web analytics is still a mess. But we are starting to see that most of that mess stems from the fact that web analytics is tightly coupled to the organisation and most organisations are living organisms that require you to adapt and change as they adapt and change to their environment. Mapping the behaviour of people to the functions of an organisation is in my humble opinion one of the most interesting  puzzles to solve and not one that is easily replaced by AI either.
