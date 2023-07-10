---
title: "Mastering Time in dbt: Incremental Merging of Estimates and Actuals for large datasets"
date: "2023-02-21"
tags: 
  - dbt
  - data modelling
  - bigquery
  - incrementality
description: "Managing incrementality (change over time) in a large database is hard. 
Dbt can help us alleviate some of the pain by making the selection of incremental strategies we have easier 
to choose from. Lets look at updating an example sales table with actuals and estimates over time." 
---
Incrementality is hard. Updating larges table with new information while also updating existing entries is not for the faint-hearted, but of course you are here because you've risen to the challenge. And so we shall conquer time and (storage) space together! The challenge ahead of us today is that of overwriting estimated data points with actual data points when they arrive (like the cavalry). 

Let's first take a step back and reiterate what incrementality is, why it can be problematic when we have data or rows that need to be updated at a later point in time from the original entry (also called late arriving facts). Have a look at the following table from a spreadsheet that contains sales data from Peppa and George. 

{{<table "table table-striped table-borderless table-hover">}}
| date      | name | sales | status   |
| --------- | ---- | ----- | -------- |
| Jan. 2023 | Peppa | 4     | actual   |
| Jan. 2023 | George | 4     | actual   |
| Feb. 2023 | Peppa | 8     | actual   |
| Feb. 2023 | George | 2     | actual   |
| Mar. 2023 | Peppa | 6     | estimate |
| Mar. 2023 | George | 3     | estimate |
{{</table>}}

As you can see we have validated information for January and February, but for March we only have estimates. Maybe that is because March is not complete yet or because it takes a few weeks for sales that originated in March to fully close. Now imagine that end of March we receive new information (e.g. a spreadsheet file):

{{<table "table table-striped table-borderless table-hover">}}
| date      | name   | sales | status   |
| --------- | ------ | ----- | -------- |
| Mar. 2023 | Peppa  | 7     | actual   |
| Mar. 2023 | George | 4     | actual   |
| Apr. 2023 | Peppa  | 6     | estimate |
| Apr. 2023 | George | 3     | estimate |
{{</table>}}
 
We have a couple of strategies to integrate this data:
- Reload all spreadsheets all the time
- Overwrite a specific time period
- Insert new entries and update existing ones

Let's go over all of them and how they work in dbt. If you are not familiar with [dbt](https://www.getdbt.com) yet, it is a tool that allows you to modularise and version control your SQL queries.

## Reloading everything everytime
Our first strategy is to reload all the data all the time. This means we don't have to consider any difference between a full refresh and incremental loads. This can work for small datasets, but of course when you cross a threshold of 100+ MBs or files this doesn't make sense anymore.  At the same time we still need to reconcile our data. It can be interesting to compare the actuals to the original estimates, but in this case we are just looking for the most accurate data.

In dbt we could roughly do something like the following.
```sql
SELECT
	date,
	name,
	sales,
	status
FROM (
	{{ dbt_utils.deduplicate(source('data', 'sales'), "date, name", "status") }}
	)
```

Let's first assume that our spreadsheets are just loaded altogether in one big table. In this case we select all columns, but we use the the deduplication macro to get one record per date and name using those in the "partition by" argument. For each of those date-name partitions we order by status in ascending order (default). Since `actual` comes before `estimate` we will select the actual first if it's available. We could use a `WHERE` filter on status, but the deduplication also gives us the guarantee that if there is an overlap in data (e.g. the same row occurs in two different files that are loaded) we will only take one row.

The `dbt_utils.deduplicate` macro is a nice touch because it will automatically optimise for you warehouse, but in essence it is very similar to the strategy of adding a row number to each partition and selecting the first row:

```sql
with counting_rows AS (
	SELECT
	date,
	name,
	sales,
	status,
	ROW_NUMBER() OVER(PARTITION BY date, name ORDER BY status) AS rn
FROM {{ source('data', 'sales') }}
)

SELECT * EXCEPT(rn) 
FROM counting_rows
WHERE rn = 1
```


## Incrementality: Overwriting (parts of) the existing table
Our previous approach was scanning all the data all the time. That's fine for a minimal set of data, but what if you have for example 1000+ days of web analytics data? That will quickly get you over 2TB of data in total so you'd definitely want to prevent scanning all of that. If not for performance reasons, then for cost —or the enviroment if you prefer. Now appending is not necessarily something we do with dbt, it is something that is done to our underlying source data. A data warehouse like Snowflake will allow you to use an append only incremental strategy, but BigQuery for example, does not. 
Since our source table has data appended to it continuously every month, we still need to deal with getting the right facts out of our source table and into our target table that we can use for reporting. Instead of scanning our entire source table and update our entire target table, we want to use some sort of lookback window where every month we want to get the most recent data and update/insert only that data into our target table.

```sql
{{
    config(
        materialized='incremental',
        incremental_strategy = 'insert_overwrite',
        partition_by = {
	        'field': 'date', 
	        'data_type': 'date',
	        'granularity': 'month'
	    }
    )
}}

SELECT
	date,
	name,
	sales,
	status
FROM {{ source('data', 'sales') }}
WHERE TRUE

{% if is_incremental() %}
AND date BETWEEN
    DATE_SUB(CURRENT_DATE(), INTERVAL 2 DAY)
    AND DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
{% endif %}
```

There's a lot more happening in this statement than in our previous one. First of all, it is incremental. We materialise it as incremental, which means that dbt will apply an [incremental strategy](https://docs.getdbt.com/docs/build/incremental-models#about-incremental_strategy). That still doesn't say anything if you are just getting started with incrementality but what it comes down to is you can either:
- `append` data without changing existing data (default on Spark, optional on Snowflake, not available for other data warehouses)
- `insert_overwrite`, that is overwrite partitions. You can think of a partition as a folder in a filesystem. In this case every month is a folder containing data of that month. Overwriting a partition means fully replacing the information in the folder with the new information.
- `merge`, that is, use a unique (combination) of keys (column names) to determine which specific rows to update. Or if the key doesn't exist yet, insert a new row.

In this case we are using the `insert_overwrite` strategy, which is [usually the fastest ](https://discourse.getdbt.com/t/benchmarking-incremental-strategies-on-bigquery/981) . `insert_overwrite` has two potential drawbacks:
- It might need a bigger lookback window than necessary (i.e. consume more data) if your partitions span a large timeframe. But more importantly,  
- You need to have all the information of the partition, not just the new information since you are fully overwriting the partition.

In our use case this is perfect, because our sheet contains both estimates and actuals for a specific time period. Since our partitions are month based, we can select data between two months ago and this month if we are on an incremental run. Only the first time and when we do a `--full-refresh` in dbt will the date selection in the `WHERE` clause be ignored and in that case we will scan the full set of data. Of course all of this assumes our source data is also partitioned by date, or —as is common in BigQuery— sharded, that is, you would use a `_table_suffix` like `20230101` to differentiate between different time periods. 

## Incrementality: Updating the existing table

Funnily enough our last strategy is not much different in terms of code from our previous strategy. However, the actual behaviour is very different. Where our `insert_overwrite` strategy was overwriting the entire "folder" of the partition, a `merge` strategy will look for existing rows and either update them if they match the unique key you have defined or insert them if they are new. It is usually a little bit slower than overwrites, very effective and precise.

```sql
{{
    config(
        materialized='incremental',
        incremental_strategy = 'merge', -- default
        unique_key=["date", "name"]
    )
}}

SELECT
	date,
	name,
	sales,
	status
FROM {{ source('data', 'sales') }}
WHERE TRUE

{% if is_incremental() %}
AND status = "actual"
AND date BETWEEN
    DATE_SUB(CURRENT_DATE(), INTERVAL 2 DAY)
    AND DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
{% endif %}
```


## Playing with time in dbt
Regardless of whether the table is partitioned or sharded by date, we have so far been defining our range dynamically. We take the `CURRENT_DATE()` and subtract a number of days from the current date to get to a start date. This is a good start, but selecting the right time frame for large tables can get quite complex. Think of the following situations:
- You want to update 3 missing days or period with errors from 90 days ago
- You develop new models locally on only the last 30 days of data instead of 2TB of historical data to speed up `dbt run`
- The CI/CD pipeline runs quick tests on only a week of data while also being able to test the incremental models on the last 2 days of data.

So let's go back to our original date range selection using a `BETWEEN` statement that takes a start and end date.
```sql
AND date BETWEEN
    DATE_SUB(CURRENT_DATE(), INTERVAL 2 DAY)
    AND DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
```

We can adjust our time range by changing the interval depending on the context or target by using a variable in dbt.
```sql
AND date BETWEEN
	DATE_SUB(
	CURRENT_DATE(), 
	INTERVAL {{ var('lookback_days', 2 if is_incremental() else 7) }} DAY
	)
	AND DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
```

With this little trick we do three things at the same time:
- We can pass a variable called `lookback_days` to our `dbt run` statement to, for example take the last 30 days of data. 
- If that variable is not defined the default will be 7 days the first time it runs
- On subsequent (incremental) runs the default will be 2 days

This is an easy start for incrementality, however it does not yet support our use case of backfilling a specific period of dates. To do that we will need to pass not just a number of days to lookback, but a specific start and end date. If both are passed we will use them, otherwise we will fallback to our earlier date selection mechanism

```sql
AND date BETWEEN
{% if var('start_date', false) != false and var('end_date', false) != false %}
	DATE({{ var('start_date') }})
	AND DATE({{ var('end_date') }})

{% else %}
	DATE_SUB(
	CURRENT_DATE(), 
	INTERVAL {{ var('lookback_days', 2 if is_incremental() else 7) }} DAY
	)
	AND DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
{% endif %}
```

And that's all. Now you too, can be a master of time with dbt.
