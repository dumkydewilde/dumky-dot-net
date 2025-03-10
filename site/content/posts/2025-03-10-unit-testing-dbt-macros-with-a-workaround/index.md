---
title: "Unit Testing dbt Macros: A workaround for dbt's unit testing limitations"
date: "2025-03-10"
tags:
  - dbt
  - data-engineering
  - data-quality
  - sql
  - testing

description: "Ever wished you could catch that broken SQL logic before it wrecks your dashboards? With dbt 1.8's new unit testing capabilities, you can finally sleep at night! However, support for testing macros is still limited. Let's explore how to test both models and macros with a workaround."
---


If you come from an analytics background you might still know what it was like to have folders full of notes and SQL files to keep track of your transformation queries for dashboards and analyses for business questions. Dbt solved a lot of the problems with that approach by keeping track of changing queries in a team setting through version control, splitting long queries into smaller building blocks, even testing the resulting tables or views. But one key piece was still missing: unit testing. 

A unit test is type of test in concept that checks the output of a function given specified inputs. If my function is:
```python
def hammer_times(a):
	return a * "🔨"
```
Then the input of `3` should always result in `🔨🔨`🔨.. This makes it easy to write a test that checks if I'm making changes to the code the output of the function is still the same given my input of `3`. 

Now, since dbt version 1.8 it is possible to do this with a model. Let's take a look at a real world example.

## Unit Testing a dbt Model
Let's say we want to better understand search traffic in our website data. To be able to analyse it we want to distinguish direct traffic, organic search and paid search. Our model could look something like this:
```sql
-- fact_website_campaigns.sql
SELECT
	session_id,
	session_start_time,
	source,
	medium,
	CASE 
		WHEN source IS NULL 
			OR source = '' 
			OR LOWER(source) = 'direct') AND (medium IS NULL 
			OR medium = '' 
			OR LOWER(medium) = '(none)') 
		THEN 'direct' 
	
		WHEN LOWER(medium) IN ('organic', 'organic search') 
			OR (source IN ('google', 'bing', 'duckduckgo', 'yahoo') AND medium IS NULL) 
		THEN 'organic_search' 
	
		WHEN LOWER(medium) IN ('cpc', 'ppc', 'paidsearch', 'paid search') 
			OR (LOWER(source) LIKE '%adwords%' 
			OR LOWER(source) LIKE '%google ads%') 
		THEN 'paid_search'
	END AS standardized_campaign

FROM {{ ref('stg_website__sessions') }}
```

With a normal data test we would be able to test for things like the uniqueness of values in the session_id column, but with the unit test we are actually able to test our business logic. We can say that given a specific combination of values in the `source` and `medium` columns we know what output to expect. When our business logic changes over time being able to test this becomes even more valueable. In dbt we are now able to do this as follows.

```yaml
unit_tests:
  - name: test_fact_website_campaigns__direct
    model: fact_website_campaigns
    given:
      - input: ref('stg_website__sessions)
        rows:
	      - {session_id: 1, source: none, medium: none}
	      - {session_id: 2, source: "", medium: ""}
	      - {session_id: 2, source: "", medium: "(none)"}

    expect:
      rows:
	    - {session_id: 1, standardized_campaign: "direct"}
		- {session_id: 2, standardized_campaign: "direct"}
		- {session_id: 3, standardized_campaign: "direct"}

```
With some simple steps we can say that given our dummy input, we expect a certain output. Now we can safely go get a drink when deploying the changes to our campaign model on Friday afternoon (don't try this at home...). 

There is a big catch though. Let's say I want to standardize my logic to be able to use it in multiple models at the same time using a macro. Great, because I'm sure I only need to update my logic in one place to serve all my models. Surely this is the best use case to apply unit tests. Having the correct logic is even more valuable if it's used in multiple places. Well, hold your horses. 

## Unit Testing dbt Macros
Well, lucky for you I have a solution. It's not as good as having native support for macro testing, but it's the next best thing. Instead of giving the input in a YAML configuration, I can pass the contents of an entire model as my input. Meaning I can have a dummy model, apply the macro and expect an output. Say what? Well, let's look at another example. We've transformed our business logic into a macro.

```sql
{% macro standardize_campaign_data(source_column, medium_column, campaign_column) %} 
CASE 
	WHEN ({{ source_column }} IS NULL 
		OR {{ source_column }} = '' 
		OR LOWER({{ source_column }}) = 'direct') AND ({{ medium_column }} IS NULL 
		OR {{ medium_column }} = '' 
		OR LOWER({{ medium_column }}) = '(none)') 
	THEN 'direct' 

	WHEN LOWER({{ medium_column }}) IN ('organic', 'organic search') 
		OR ({{ source_column }} IN ('google', 'bing', 'duckduckgo', 'yahoo') AND {{ medium_column }} IS NULL) 
	THEN 'organic_search' 

	WHEN LOWER({{ medium_column }}) IN ('cpc', 'ppc', 'paidsearch', 'paid search') 
		OR (LOWER({{ source_column }}) LIKE '%adwords%' 
		OR LOWER({{ source_column }}) LIKE '%google ads%') 
	THEN 'paid_search'
END AS {{ campaign_column }}
{% endmacro %}
```

In the next step we create a new model.

```sql
-- models/unit_tests/macros/test_macro_standardize_campaign_data__direct.sql
WITH input AS (
    SELECT 1 AS session_id, NULL AS source, NULL AS medium
    UNION ALL
    SELECT 2 AS session_id, '' AS source, '' AS medium
    UNION ALL
    SELECT 3 AS session_id, '' AS source, '(none)' AS medium
)

SELECT
    {{ standardize_campaign_data("source", "medium", "output_column") }},
    *
FROM input
```

Now instead of running the unit test on our macro or our model with the business logic, we run it on our dummy model. To make this work, we have to tell the unit test it should take itself, i.e. `this` as the input.

```yaml
unit_tests:
  - name: test_macro_standardize_campaign_data__direct
    model: test_macro_standardize_campaign_data__direct
    given: []

    expect:
      rows:
        - {session_id: 1, output_column: "direct"}
        - {session_id: 2, output_column: "direct"}
        - {session_id: 3, output_column: "direct"}

```

We can now run this by first initializing our unit test dummy models as empty models:
```
dbt run -s unit_test.macros --empty
```
And then test them:
```
dbt test -s unit_test.macros
```

## A Generalised Approach
We can take this approach a little bit further and generalize it, although we will soon run into the limits of dbt as a tool. Instead of defining input like `SELECT 1 AS session_id, NULL AS source, NULL AS medium` in our macro testing model, we can create a general purpose 'input' model. The model will be empty and materialized as ephemeral:
```sql
-- /models/unit_tests/macros/macro_input.sql
{{ config(materialized='ephemeral') }}
```

We will then replace our macro testing model as follows:
```sql
-- models/unit_tests/macros/test_macro_standardize_campaign_data__direct.sql
SELECT
    {{ standardize_campaign_data("source", "medium", "output_column") }},
    *
FROM {{ ref('macro_input') }}
```

You might ask "why?" And that's always a good question. Unfortunately we cannot add a YAML dict of rows as we could for the original model we were unit testing. Using a YAML dict or even overriding a macro does not work as dbt will not be able to compile the correct SQL due to internal limitations. But adding in the ephemeral model does allow us to pass SQL as the input. That means we can at least achieve a notation where we show both the input and output in our configuration.

```yaml
  - name: test_macro_standardize_campaign_data__direct_with_ref
    model: test_macro_standardize_campaign_data__direct
    given:
      - input: ref('macro_input')
        format: sql
        rows: |
          SELECT 1 AS session_id, NULL AS source, NULL AS medium
          UNION ALL SELECT 2 AS session_id, '' AS source, '' AS medium
          UNION ALL SELECT 3 AS session_id, '' AS source, '(none)' AS medium

    expect:
      rows:
        - {session_id: 1, output_column: "direct"}
        - {session_id: 2, output_column: "direct"}
        - {session_id: 3, output_column: "direct"}
```
