---
title: "Language Detection in SQL with BigQuery Remote Functions"
date: "2022-06-30"
tags: 
  - "NLP"
  - "BigQuery"
  - "Cloud Function"
  - "language"
  - "gcp"
description: "Over the last few years SQL has really started embracing its second adolescence. That's cool, but what if you could easily extend your queries beyond the SQL domain and add in Python and Javascript based serverless functions to get real time stock information, enrich location data or: build a language detection function!? That's what we'll do." 
---

Over the last few years SQL has really started embracing its second adolescence —it's nearly half a century old after all. It's been riding the fast sportsbike —dbt— and now the Google BigQuery SQL dialect is trying out something more extreme: native integration with serverless functions, called [BigQuery Remote Functions](https://cloud.google.com/bigquery/docs/reference/standard-sql/remote-functions). Ok, maybe that doesn't sound as exciting as a sportsbike, but hear me out. 

A SQL query has always been limited by the constraints of the language and the database it lives in. What if, within that same language, you could take any set of data (rows, columns) and let it take a quick, easy, scalable step outside of that ecosystem and back. That's basically what BigQuery's integration with serverless functions means. You can enrich any kind of data from your database through a Python, NodeJS, Java, or whatever-runtime-you-prefer function and pipe it straight back. I can think of a few use cases like adding in realtime stock prices or currency conversions, enrich company or location data from outside APIs, but today I want to start with something simple and language related.

Recently I was looking at a dataset of around 300.000 short user comments and reviews of which 30% was in English and the rest in one of 26 other languages which had to be translated to English for use. Of course we don't want to call a translation API for that 30% but we do for the other 70%. Hence the need to quickly detect the language. That itself is simple (and accurate) enough with a small, pre-trained fast text model. However getting the 300.000 outside a database, through a Python model and back in felt more cumbersome than needed. It wasn't available at the time, but this would've been a great use case for BigQuery remote functions.

## Creating a SQL serverless function
Although, it takes a few steps to set up, the whole process is not overly complex. Let's start with the basic script. It requires [our pretrained model](https://fasttext.cc/docs/en/language-identification.html) bundled with the function, the fasttext library and a simple request/response framework (Flask) to respond to the input from BigQuery. 

We start with some basic dependencies to install  `pip install fasttext functions-framework` and then create a function that takes the "calls" object from BigQuery, which could be one or multiple rows of data, and returns a value —the predicted language— for each element in the object.

```Python
import functions_framework
import fasttext
import json

ft_model_path = './lid.176.ftz' #pre-trained model
ft_model = fasttext.load_model(ft_model_path)

@functions_framework.http
def lang_detect(request):
    """Responds to any HTTP request.
    Args:
        request (flask.Request): HTTP request object.
    Returns:
        The response text or any set of values that can be turned into a
        Response object using
        `make_response <https://flask.palletsprojects.com/en/1.1.x/api/#flask.Flask.make_response>`.
    """
    try:    
        request_json = request.get_json()
        calls = request_json['calls']

        lang_predictions = ft_model.predict([call[0] for call in calls])[0]

        replies = [p[0][9:] for p in lang_predictions]  # remove __label__

        return_json = json.dumps( { "replies" :  replies} )
        return return_json
    except Exception as inst:
        return json.dumps( { "errorMessage": 'Unexpected value in input' }, 400 )
```

Now, assuming you have set up [a Google Cloud project](console.cloud.google.com),  we can use [the gcloud CLI](https://cloud.google.com/sdk/docs/install) from the folder with our script on our local computer we can deploy the function with:
`gcloud functions deploy language-detect --entry-point lang_detect --runtime python37 --trigger-http --region=europe-west1 `

Next we create a new BigQuery connection resource that allows for external connections from our database (don't forget to add in your project ID in the right place):
`bq mk --connection --display_name='cloud functions connection' --connection_type=CLOUD_RESOURCE --project_id=<<PROJECT ID>> --location=europe-west1 cloud-functions-connection`

The connection resource then needs to have the right permissions to access our cloud function. Luckily BigQuery has already set up a service account that allows the two services to interact with eachother without the need for those pesky humans. Show the associated service account with `bq show --location=europe-west1 --connection cloud-functions-connection`

This will give you the email address for the service account that you can add in the projects IAM (access) manager: `https://console.cloud.google.com/iam-admin/iam?project=<<PROJECT ID>>` . Add the email address as a principle and give it the role of 'Cloud Functions Invoker'.

Now all that is left is to create a custom SQL function (UDF) that we call from our queries. Enter the following through the BigQuery UI to create it.
```SQL
CREATE FUNCTION <<DATASET>>.lang_detect(x STRING) RETURNS STRING

REMOTE WITH CONNECTION `<<PROJECT ID>>.europe-west1.cloud-functions-connection`

OPTIONS (endpoint = 'https://europe-west1-<<PROJECT ID>>.cloudfunctions.net/language-detect ')
```
Make sure that the data processing locations are the same across the board. 

Now it's time to have some fun and try out our new remote function with some of that European goodness (sorry, I only know four languages 🤓):

```SQL
CREATE OR REPLACE TABLE `<<PROJECT>>.<<DATASET>>.language_data` as (
  select
    "Ich liebe schwarzwälder kirsch" as comment,
    5 as score,
    union all select "J'aime des pains au chocolat", 3
    union all select "I hate deep fried Mars bars", 1
    union all select "Een lekker stukje kaas!", 5
)
```
