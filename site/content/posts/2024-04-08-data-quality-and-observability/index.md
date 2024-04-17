---
title: "Data Observability is not a tool: understanding data quality at the source, in transformations and in governance"
date: "2024-04-08"
tags: 
  - observability
  - data-quality
  - data-governance
  - lineage
  - ETL
  - analytics
  
description: "Have you ever wasted time or money because you made a decision based on incorrect data? Then you'll appreciate good data quality. Buying an observability, however, might not be the solution to your data quality issues. Let's explore how data quality issues arise at the source, in transformations and in data governance and find the appropriate solutions to those problems." 
---


Have you ever wasted time or money because you made a decision based on incorrect data? Then you'll appreciate the quality of your data. Fixing data quality issues, however, is a whole different beast. There are many different tools and technologies claiming to fix your data quality: data observability tools, data lineage, semantic layers, data catalogues, but to find the right solution to your problems, you need to understand why data quality is an issue in your organisation in the first place. 

I define three broad areas where data quality issues can occur:

- **In the source system**: Here, data quality issues can arise because, by definition, this system is
separated from your central data warehouse. That means that any decision and assumption
made by the developers of the source system when collecting or designing the data model of
the system might not be communicated properly to any consumers of this data.

- **When moving data from a source system to a target system**: An example of this might
be moving data from your sales system to your central data warehouse. In that process, you
can create issues around the integrity or completeness of the data or make errors in the
transformations applied to the data.

- **In the governance of data quality**: Here, additional issues can arise, for example, a misunderstanding
of documentation, a lack of metadata, such as the time of extraction, or unclear ownership are
all data governance-related issues that can impact data quality down the line.

{{< box important >}}
This post is a summary of a chapter in my book [The Fundamentals of Analytics Engineering](https://amzn.to/43KxIik), a perfect handbook if you are looking to learn more about 
data modeling, data quality, data warehousing and data ingestion.
{{< /box >}}

# Data quality issues in source systems
Whether it is your CRM system, an third-party API or an independent report. Wherever data is collected assumptions are made about that data, maybe it is about the definition of a user or a customer, maybe it is about a time or a timezone, or it might be a more complex or implicit assumption about uniqueness or granularity. In other words: in every system you use as a source for your data platform, someone has made a decision about the concepts and entities that are relevant to your data platform and directly impact data quality. If you want to stay ahead of data quality issues coming from a source system in your data platform you have to consider 5 key aspects: completeness, consistency, reliability, integrity, and relevance.

## Completeness
A simple question like, “when was customer X last contacted?” is surprisingly hard to answer for many companies. They will use many different systems and sources that are not connected. So any analysis done on this data might mean parts of the picture are missing. The missing data may even introduce some bias. Often, you do not know what you do not know, and if there is no visible change in your data, you might have incomplete data for years. Countering incompleteness will sometimes mean accepting you cannot have perfect data quality and documenting uncertainty, especially when you have limited control over the data collection. In other cases, you may still be able to test your source
data or algorithmically detect anomalies.

## Consistency
Imagine a smart humidity monitor in an underground parking garage using a mobile connection. It might be able to send data every now and then, but a device like this that sends data  intermittently will probably be inconsistent over time. This will result in: bias, uncertainty, and, eventually, less accuracy in decision-making. The consistency of data, however, can be checked in different ways. First, a data source can be checked against itself over different dimensions. Is it consistent over time or when comparing different groups, such as countries or states? Second, some data is available from different sources allowing you to create benchmarks for your data. 

## Reliability
Sometimes, you have little or no access to the data collection process, or it might be so complicated it requires a PhD to understand it. In that scenario, you will have to ensure the source is reliable. There are many different ways to determine reliability, and unfortunately humans are surprisingly bad at this, but here are some simple heuristics
you can use. For starters, you can test your assumptions about the source system. Simple tests on uniqueness, NULL values, or data freshness may suffice. Secondly, as the famous saying goes, “Follow the money.” If there is an incentive for your source to over or under-report (for example, Facebook did this multiple times with ad impressions over the years), this will likely happen. So, if you do not have that PhD to understand the data collection system, it could help to hire an independent consultant
who assesses the system.

## Integrity
The fourth aspect when looking at data sources is integrity. Even when your data collection process is
bulletproof, and your source is reliable and consistent, you may still run into trouble. For one thing,
if you just did an analysis on that perfect dataset but the users from whom that data was collected
did not consent to its use for analytics purposes, you might have to throw out your data altogether.
Legislation and compliance are important factors regarding data integrity, and they can definitely
affect your process if you do not consider them.

Integrity also refers to whether or not your data may be compromised. In the worst case, malicious
actors might mess with your data; in other cases, bots, automation, or spam might ruin your data.
Some organizations unknowingly open the gates to data hell by asking the public for input. When
the data collection system is poorly designed, some people will not use it as intended. One such case
was the naming of a new British research ship for which the public was asked for input and a vote.
The result of that vote was the name Boaty McBoatFace, which was not exactly the intended outcome.

## Relevance
When you ask a stakeholder in your organization what they would like to measure, the most annoying
answer you can get is “everything.” It is often appealing to capture as much data as possible, but that is
a recipe for disaster. Adding more and more data to your data lake will end up creating a data swamp. Data without context and meaning is just noise. 
The availability or absence of clear business goals will also make or break your data definitions. It
is common to see definitions and metrics based on the source data instead of the target metric or
indicator. That leads to a wild growth in definitions for the same metric or indicator. As a consequence,
these definitions will have to be aligned later in the process, leading to additional business logic
and complexity.

# Data quality issues in data infrastructure and data pipelines
Of course, even if your data source is reliable you might still introduce a lot of issues when moving data from a source to a target system.
There are five important aspects to consider: timeliness, integrity, completeness, precision, cleaning.

- **Timeliness**: Is the data you need a representation of the current state of affairs and actually there when you need it? Or are you making decisions on outdated and inaccurate data?

- **Integrity**: Data integrity requires the assurance that data is not corrupted over its entire lifecycle. The integrity of the data processing infrastructure (that is, the 'pipeline') means that each part of the pipeline has to contribute to keeping the integrity of the data. That's easier said then done with dropping connections, retries, incremental loads, duplicates, mismatching environments, etc.

- **Completeness**: Is all the required data present? Was there a problem in the source system at some point, causing missing data? Is missing data contributing to bias and is it a known unknown or an unknown unknown.

- **Precision**: The precision or granularity of your data can be one of the trickiest parts of getting data quality right; Sometimes it is hard to get data at the lowest granularity either because it is too much like with sensor data, time series data or behavioural data. Or sometimes the data requires interpretation in the first place (for example: entity recognition, topic extraction, OCR). Compare event-level vs. daily, weekly or monthly. But also dimensions can have different granularities (marketing campaign, ads, ad text/image; organisation, business unit, location, employee)

- **Cleaning**: Cleaning data means you are making assumptions about what needs to be cleaned and removed. Lack of proper cleaning means you'll have 'dirty' data which can be harder to work with and consolidate later on, but of course any assumption you make might introduce a new error or bias.

# How data governance impacts data quality

## Documentation
One of the most obvious yet underappreciated areas where data governance impacts data quality is
documentation. It is common to hear horror stories from new employees starting in a data swamp of
hours-long ETL jobs and 1,000-line SQL queries after all previous staff have gone without leaving any
documentation behind. This often reflects the inability of an organization to balance the short-term
time investment in documentation with the long-term cost of debugging data pipelines. However,
borrowing time from the future in this way is usually a terrible investment. 

It doesn't even have to be this bad. Sometimes not writing down a clear definition of a metric can wreak havoc on reports.
Imagine a hypothetical online dating platform where someone has implemented the metric 'active users' as 'logged in users'. 
Now someone changes the session duration from 1 to 30 days and all of a sudden the active users are growing even though
this might not match the desired definition of what an 'active user' is.

## Consensus
Have you ever encountered a situation where you present a report with some numbers, and someone
from a different team says, “But my Excel sheet here shows a different number”? Getting a single source
of truth, or consensus, in your organization is very hard to achieve, as different teams rely on different
operational tools with different types of data collection and definitions. Sometimes, even getting a
consensus for one data source is hard. 

## Accountability
A crucial part of data governance is the ownership and accountability of data. We have already seen
how dependencies on source data or other teams can create data quality issues and
confusion, and those dependencies become even harder to manage when there is no clear owner of
the underlying data. When you clearly see issues with the data that you need to work with but no one
is willing to address them, or it is unclear who should take accountability for addressing the issues,
data quality will suffer.

## Metadata management
Metadata is nothing more than data about data, and gives context to your datasets. Metadata can
detail when, how, and by whom data was collected, transformed, or accessed. Without robust metadata
management, teams can struggle to understand data relevance and quality or even have trouble discovering data in the first place.
This can lead to potential misuse or misinterpretations during analysis.

## Cost management
As you ingest and process more and more data, your pipelines might account for a substantial chunk of your total cloud costs.
For some reducing cloud spend is a religion, but for most it is up to you to decide when it makes sense to spend time and
money on reducing your cloud costs. You could write an entire book on cost management for data, but often the biggest overspending
comes from a few simple issues:

- **Ingesting and storing data you do not need or use**: For example, when you miss visibility or
knowledge on legacy data ingestion processes that are still running.

- **Ingesting and storing more often than you need the data**: If your analysts are not in the office
on the weekend, you might not need to run that expensive pipeline on the weekend. If they
look at the data once a day, you might not have to run it every 15 minutes.

- **Over-engineering your data pipelines**: Just because Netflix, AirBnB, or Google wrote a blog
about it, it does not mean you need to implement their solution for a problem you do not have.
Keeping things as simple as possible will take you a long way in saving costs.

- **Under-engineering your data pipelines**: The biggest benefit of having data or software engineers
on your team is that they can turn business rules and logic into code. Instead of clicking your way
through the tool that’s in vogue today, you can spend a bit more time and money engineering
a solution that leverages programming and data languages that have been around for over 30
years, and that will likely be around for the next 30 years.

## Data lineage
Considering the many places where issues can arise in moving data from one place to another, data
lineage is a crucial aspect of data quality. In essence, it describes the journey of your data. However,
the lack of proper implementation or even the absence of data lineage can cause severe issues or, at the
very least, time-consuming debugging sessions. That makes data lineage both a problem area and a
potential solution for data quality. Understanding data lineage can vastly increase the speed at which
you can identify and solve data quality issues.

# Finding solutions to data quality issues: observability, data catalogs, and semantic layers
Making sure the quality of your data matches your business needs is crucial for making better
decisions and creating trust in your data. Luckily, we have a set of tools and techniques at
our disposal to overcome issues around data quality. 

## Using observability to improve your data quality
Data observability is a growing trend that borrows principles from software engineering and operational
monitoring, adapting them to the nuanced needs of data management. Observability is a concept from
the software engineering and DevOps fields, where being able to quickly and consistently observe
issues is helpful to minimize or even prevent the downtime of application systems. When translated
into the data world, this means having visual insights into the data dependencies of a given table or
dashboard, as well as any errors, anomalies, or data quality issues, and being able to monitor where
and when they appear.

I quite like the definition from Yuliia Tkachova from the data observability company [Masthead](https://mastheadata.com).
She defines data observability as: 
> “the ability of an organization to track and control its data landscape and multilayer data dependencies (pipelines, 
  infrastructure, applications) at all times […] with the goal of detecting, preventing, controlling, and remediating 
  data outages or any other issues disrupting the work of the data system or spoiling data quality and reliability.”

Any observability solution, whether consisting of one or multiple tools, will need the following features:

- **Lineage**: Having a (preferably visual) understanding of how datasets, tables, and transformations
depend on each other is the first step to identifying data quality issues. The most common
way to do this is with a directed acyclic graph (DAG), as seen in Figure 9.4, which is a type
of visualization that shows the linear flow of data from source to destination and, sometimes,
even from column to column in a dataset. If the linear character does not work for your use
case, you might be better off using something as simple as a heatmap or an entity-relationship
diagram where relationship types, such as one-to-one or one-to-many between entities (the
objects your business works with), are mapped out.

- **Testing**: Though never enough on its own, testing data against assumptions, such as uniqueness
or non-null values, is great for quickly identifying issues as they appear. Many tools provide
extensive testing solutions, sometimes even with machine learning and anomaly detection on
top of them. While testing is great for providing guarantees about your data, it will not help you
discover unknown problems with your data. Some examples of this are an orphaned pipeline
where some data are still moved around in a valid way, but it is never used by a downstream
dependency; a rogue dashboard that is created by another team in an environment that is not
observed by the data team; or a legacy database query outside of the current system, created
by an employee that no longer works at the company and that takes a long time to run.

- **Monitoring**: With your tests running and your pipelines turning, you need a way to see problems
at a glance. Since this is a shared and common problem for many organizations, you can easily
use the metrics and visualizations that tools provide. It could be as simple as the duration of
a data ingestion pipeline, the number of rows that do not match a specified schema, or even
whether a data loading job failed or succeeded.

- **Alerting**: While your monitoring solution might be running perfectly and detecting all your
data pipeline problems, if no notification ever reaches you, it might all be in vain. An alerting
solution, whether it is an email or a message on Slack, can alert you to act on data quality issues.
The crucial part of any alerting solution is not just that it reaches you but that it also strikes the
right balance between when to alert and when not to alert to prevent alerting fatigue.

- **Tracing**: Where monitoring and lineage give you a high-level overview of what went wrong
and where the dependencies are, tracing can be a way to identify the errors or issues in specific
transformations and functions that are applied to your data. It could be an issue with performance
or timing, understanding when and where the error or culprit was introduced, or pinpointing
the exact data transformation or function that is not operating as intended.

- **Profiling**: When working with data, not all errors are as easy to spot as failures in your program
or function. Sometimes, a data transformation works well, but the outcome is not what is
intended. There could be duplicates or missing values where none are expected. Profiling is
a way to easily understand the makeup of your dataset by visualizing, often on a per-column
basis, the number of NULL values, averages, maximum and minimum values, distributions,
formatting, and similar insights to understand if your data makes sense in a single glance.

- **History capture**: A big problem with data transformations is that, over time, the data or the
transformation might change with the business or the pipeline. Often, it is important to keep
track of changes to the data for audit and compliance purposes, but similarly, you might want
to keep track of data quality for debugging purposes.

- **Anomaly detection**: We have already considered how testing can help you with guaranteeing
certain assumptions you have about the data. You can also try to detect anomalies in the
unknown parts of known data. For example, when you ingest way more or fewer rows than
you usually do or when certain values are out of normal bounds, this could be an indication
of a data quality issue.

## The benefits of data catalogs for data quality
For a long time, the hardest part of analytics was the discovery and availability of data. We are at a
turning point where the discovery and gathering of data is getting easier, but with increasing data
volumes, understanding the quality of that data is getting harder. Data catalogs are intended to solve
a big part of the problem around discovering data, including the quality and lineage of that data.
A catalog is not just valuable for data consumers. With the increasing availability of
data, managing data quality is more important for data producers as well. Having a catalog can provide
a clear overview of all the data assets that a data producer is responsible for. That can significantly
assist in compliance and governance, as a data producer is able to more easily manage permissions
and access and understand the lineage, dependencies, quality, and potentially, the costs of their data.

In the mean time, big parts of implementing a data catalog are becoming easier and easier. With the rise in machine
learning and large language models, writing documentation, creating descriptions, and generating
metadata have become more manageable tasks. While computer-generated descriptions might
not be perfect, they are a great first step and can save a human a lot of time when improving those
descriptions. Tools such as [Select Star](https://www.selectstar.com/), [Atlan](https://www.atlan.com/), and [Collibra](https://www.collibra.com/) can automate a lot of the
work that goes into creating a data catalog.

## Improving data quality with a semantic layer
While a data catalog and good documentation help to overcome many challenges around creating
shared definitions of data, this often lacks the practical implementation of going from a data model to
a dashboard. A semantic layer, metrics layer or metric store is a translation layer
that turns the complexity of data models into answers to business questions. In essence, it handles access controls, 
data modeling, metrics, aggregations, and caching so that drag-and-drop queries from business users can be translated 
to database queries regarding the raw data. It is not a replacement for your visualization tool, nor does it replace your transformation
layer. Instead, it serves as an abstraction layer to centralize metrics within your organization.