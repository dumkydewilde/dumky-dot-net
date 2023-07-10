---
title: "Dbt In a Box: Using Google Cloud Run and BigQuery to run your dbt SQL models from a Docker container"
date: "2022-12-31"
tags: 
  - "dbt"
  - "snowplow"
  - "BigQuery"
  - "Cloud Run"
  - "Docker"
  - "gcp"
  - "cloudbuild"
description: "Dbt is a great tool for data transformation. Snowplow is great for collecting web analytics data. What if you could harvest the power of both for just a few cents a day by running dbt in a Docker container on Google Cloud Run Jobs?" 
---
{{< box important >}}
*Update June 2023*: I have created a [massive blog post](/posts/own-your-web-analytics-pipeline-for-0.02-per-day-snowplow-terraform-dbt-bigquery-and-docker/) on how to get started with Snowplow, Terraform, Docker and dbt that has some more insights on how to best run dbt in a container as well. This post is still valid, but know that there's more if you are interested.
{{< /box >}}
[Dbt](https://docs.getdbt.com/docs/introduction) is a great tool to transform data in your data warehouse. It allows you to version control the rules and business logic you want to apply to your raw data. However, it can be tricky to set up if you're new to it. For our use case today we'll transform raw web analytics data in our BigQuery warehouse to dashboard-ready tables with session and user data. To be specific, we'll use this website's analytics data tracked with [Snowplow](https://snowplow.io) and process it with one of [Snowplow's dbt packages](https://hub.getdbt.com/snowplow/). The dbt packages are great because we can leverage the standardised models that the Snowplow team has created and minimise our own efforts. In other words: we'll have more time for coffee! And not only time, but also money, as our solution will be extremely efficient with resources costing at most a few cents a day: perfect for a small business or blog.

## Setup
Let's start with analysing how our data flows to better understand where dbt can add value. 
1. Raw data (pageviews) are collected with Snowplow and stored in a BigQuery table. This data is nothing more than e.g. a pageview or event potentially already enriched with data about e.g. a user's browser, geography or other context.
2. On a schedule raw data is transformed and aggregated into tables in an incremental fashion using version controlled SQL models. This will create tables for e.g.:
	- Sessions: a collection of pageviews and events for a specific user in a specific timeframe
	- Users: a unique identifier or identifiers (across websites or apps) containing information about the origin, engagement and marketing campaigns for an individual.
	- Pageviews: details of specific pageviews and how users interact with them (previous pages, engagement time, events, etc.)
3. From these tables we can build a dashboard to visualise data and trends over time.

For this post we will focus on step 2. Which has three important parts that we need to tie together.
1. Version controlled SQL models. In other words: a repository or folder with our dbt models.
2. A 'computer' to run those models and apply the transformations in BigQuery
3. A scheduler to trigger the 'computer' to run the models.

For part 1 we will not be using our own version controlled models, instead we'll be using Snowplow's standardised models and use some variables to apply them to our own data. We don't need to reinvent the wheel here and the Snowplow dbt package has plenty of flexibility to fit our needs. Similarly part 3 is a matter of setting Google's Cloud Scheduler to run every 3 hours between 6.00-21.00 on weekdays. This gives us near-realtime results without the costs of realtime. I've found this to be one of the biggest savers in data pipelines as most business decisions are not made in realtime nor on weekends.

To leverage our efficient scheduler however, our 'computer' also has to be able to fit this schedule. I often see people spinning up large virtual machines and leaving them on forever just to run a few models now and them. The actual computation to get the results table is done in BigQuery, so our 'computer' should be a lightweight instance that can compile our dbt models and orchestrate the runs. I've found the new [Cloud Run Jobs](https://cloud.google.com/run/docs/overview/what-is-cloud-run#jobs) to be perfect for just that. Cloud Run Jobs are serverless containers that can spin up in seconds when triggered to perform their task. I have a preference for GCP, but [Fargate on AWS](https://aws.amazon.com/fargate/) will do the trick as well

## Costs
I am already using Cloud Run for collecting the raw web analytics events as my traffic is very intermittent and sometimes non-existent. For the last 90 days this has cost me €2.86 or €0.03 cents a day (I've been long out of credits on GCP unfortunately). I expect this setup to cost even less since the whole job runs only for about 60 seconds on a single thread. I'm pointing this out because setting up a custom web analytics pipeline can be daunting and costly and while the default setup from Snowplow (and similar providers) is great and robust, it is not necessarily the most efficient for smaller businesses and blogs. While you should definitely think about redundancy (i.e. multiple VMs) across regions for an international e-commerce site, you should make sure to fit your infrastructure to your business needs not the other way around.

## A Shoppinglist: Docker, gcloud, dbt
Enough about money. Let's get building. To get started with [Cloud Run Jobs](https://cloud.google.com/run/docs/quickstarts/jobs/build-create-python) we have a small shopping list.
- We need **Docker** to run our dbt script in a 'container'. Docker is an easy and standardised way to create small computational workloads across platforms. That means you can easily run your container locally, then ship it to your cloud provider without fearing the problem of *it-works-on-my-machine*. If you are new to Docker try to [build a simple service](https://docs.docker.com/samples/) for something that you're familiar with like your own Wordpress instance or Database.
- We need a folder with a **dbt** project that runs locally. We'll work with Snowplow data in this example, but it could also be your own project. We'll bundle the folder with the dbt models in our Docker container.
- We'll be using the **`gcloud`** command line interface (CLI). If you don't have that installed yet, you will [need to install](https://cloud.google.com/sdk/gcloud) it first and authenticate with `gcloud init` or `gcloud auth login`.
- With the `gcloud` CLI we can then easily use Google **Cloudbuild** to build our Docker container, upload it to GCP and run it as a Cloud Run Job.
- We will also need a **service account** with BigQuery Editor and BigQuery Job User permissions so that our Docker container can access our data in BigQuery. There are other a bit more secure ways to do this, but a service account will make it more portable. Do make sure to handle your account key with care as it gives full access to your data.
- Finally we try to decouple variables that are specific to our project (project name, database name, deployment location, etc.) from the tool we are building so it becomes reusable. I like to store those variables as **environment variables** so they are easily replaced in different environments. Instead of manually doing `export PROJECT_ID=my_project` all the time, I like to use [direnv](https://direnv.net/) which creates a `.envrc` file in your project folder in which your project specific variables are stored. A simple `direnv allow .` then allows access to those variables within the scope of the folder.
- Final final v2: You might need to enable a few APIs in GCP for Cloud Run, Scheduler, etc. You will usually get notifications for this along the way, but keep it in mind.

## Building the dbt project
I'm assuming you will have some familiarity with dbt already. If not, you can follow [their getting started guide](https://docs.getdbt.com/docs/get-started/getting-started-dbt-core) to get dbt Core setup on your machine. For our dbt project there are four important parts we need to manage.

First up, we're using the [snowplow web package](https://github.com/snowplow/dbt-snowplow-web/tree/0.12.2/). So we need to define that as a dependency in `packages.yml`.
```yaml
packages:
  - package: snowplow/snowplow_web
    version: 0.12.2
```

Next we need to declare this package in our `dbt_project.yml` and define some variables so the package understands how to handle our data. Besides your own project you will also have the `snowplow_web` entity. I like to adjust the schema so it creates all standardised tables in a seperate schema from the raw data. Apart from that you will also need to set some variables to define the name of your 'schema' or dataset as it's called in BigQuery. The 'database' is your GCP project ID where the raw data is stored and finally the package needs to understand from which date on it should start processing data. As you can see I've set all that as environment variables. On top of that the `enable_yauaa` means browser enrichment is enabled in my Snowplow pipeline and `tstamp_partitioned` means my BigQuery data will be partitioned by date based on the collection timestamp (i.e. less dataprocessing and faster queries)

```yaml
models:
  my_project:
    ...
  snowplow_web:
    +schema: "sp_dbt"

vars:
  snowplow_web:
    snowplow__atomic_schema: "{{ env_var('BQ_DATASET') }}"
    snowplow__database: "{{ env_var('BQ_PROJECT_ID') }}"
    snowplow__events: "{{ source(env_var('BQ_DATASET'), 'web_events') }}"
    snowplow__enable_yauaa: true
    snowplow__start_date: "{{ env_var('WEB_EVENTS_START_DATE') }}"
    snowplow__derived_tstamp_partitioned: true
```

Thirdly, to easily run all the Snowplow models from the package as well as anything we ourselves may want to add on top we'll add the Snowplow selector to our `selectors.yml`.
```yaml
selectors:
  - name: snowplow_web
    description: >
      Suggested node selection when running the Snowplow Web package. 
      Runs:
        - All Snowplow Web models.
        - All custom models in your dbt project, tagged with `snowplow_web_incremental`.
    definition:
      union:
        - method: package
          value: snowplow_web
        - method: tag
          value: snowplow_web_incremental
```

Finally we can set the same environment variables for our `profiles.yml` to access our data including the location of our data in BigQuery (I usually choose `europe-west1` because it's close to me and has comparetively low CO2 output).
```yaml
dumky_net:
  target: dev
  outputs:
    dev:
      schema: dev
      type: bigquery
      method: oauth
      project: "{{ env_var('BQ_PROJECT_ID') }}"
      dataset: "{{ env_var('BQ_DATASET') }}"
      location: "{{ env_var('BQ_LOCATION') }}"
      threads: 4
    
    prod:
      type: bigquery
      method: oauth
      project: "{{ env_var('BQ_PROJECT_ID') }}"
      dataset: "{{ env_var('BQ_DATASET') }}"
      location: "{{ env_var('BQ_LOCATION') }}"
      threads: 1
      timeout_seconds: 1200
      retries: 1
```

If you want to see if your dbt project runs you can try to run `dbt debug --profiles-dir .` to test the connection with the profile in this folder. If you want to test the full setup you can run `dbt deps && dbt run --selector snowplow_web --target=dev --profiles-dir .`

## Packaging the package with Docker
Now we can build a Dockerfile to package up our dbt project, project specific settings and service account key so it can run from anywhere in the world (*🎵 If you can make it there, you can make it anywhere 🎵)*. Our path is mostly paved because we can start with a dbt-bigquery base image: `FROM ghcr.io/dbt-labs/dbt-bigquery:1.3.latest`

We will then capture some variables in our Docker build command by using the following arguments (that you've already seen in our dbt project). Those arguments are then set as environment variables in the container environment alongside some other environment variables that are optionally set in the build command.
```Dockerfile
# define in docker build
ARG BQ_PROJECT_ID
ARG BQ_DATASET
ARG DBT_PROJECT_DIR
ARG WEB_EVENTS_START_DATE

ENV DBT_PROJECT_DIR=$DBT_PROJECT_DIR
ENV BQ_PROJECT_ID=$BQ_PROJECT_ID
ENV BQ_DATASET=$BQ_DATASET
ENV WEB_EVENTS_START_DATE=$WEB_EVENTS_START_DATE

# default env values, can be overridden
ENV BQ_LOCATION="europe-west1"
ENV DBT_PROFILES_DIR=/usr/app/
ENV GOOGLE_APPLICATION_CREDENTIALS=/usr/app/auth/gcp-service-account.json
ENV TARGET=prod
```

We then perform some copying and test the connection
```Dockerfile
USER root

# Copy dbt project in the docker image to build
COPY $DBT_PROJECT_DIR /usr/app/
COPY auth /usr/app/auth/

# Use root to avoid permission issues
USER root

RUN dbt debug --target=$TARGET
```

Finally we set the entrypoint for our container (i.e. what it will run when triggered).
```Dockerfile
ENTRYPOINT dbt deps && dbt run --selector snowplow_web --target=$TARGET
```

If you want to run this container locally you can build it and run it with Docker. Otherwise we will use Google Cloudbuild to build it for us and upload it to GCP.
`docker build --build-arg DBT_PROJECT_DIR=my_project --build-arg BQ_PROJECT_ID --build-arg BQ_DATASET -t snowplow-dbt .`

## Cloud Running it
Now it's time for the final step: add our Docker container to a Cloud Run Job. We will use a `cloudbuild.yml` file so we can easily submit our build with the `gcloud` CLI. The Cloudbuild file consists of four steps:
- build the docker container
- upload/push it to the GCP container registry for your GCP project
- Create/update a new Cloud Run Job with our uploaded container image
- create a schedule to trigger our job 

Let's look at that first part of building the container. Our cloudbuild file takes in a few arguments called 'substitutions' that we can fill with our own environment variables to again decouple our specific configuration from the setup/code itself. Then in the first step we run the `docker` command with our build arguments and tag the container with a location/project-id name that is unique and reusable.

```yaml
substitutions:
    _BQ_PROJECT_ID: ${PROJECT_ID}
    _BQ_DATASET: ""
    _DBT_PROJECT_DIR: "my_project"
    _WEB_EVENTS_START_DATE: "2023-01-01"
options:
    dynamic_substitutions: true
    substitution_option: 'ALLOW_LOOSE'

steps:
- name: 'gcr.io/cloud-builders/docker'
  entrypoint: 'docker'
  args: ['build', 
          '--build-arg', 'DBT_PROJECT_DIR=${_DBT_PROJECT_DIR}',
          '--build-arg', 'BQ_PROJECT_ID=${_BQ_PROJECT_ID}', 
          '--build-arg', 'BQ_DATASET=${_BQ_DATASET}',
          '--build-arg', 'WEB_EVENTS_START_DATE=${_WEB_EVENTS_START_DATE}',
          '--tag=$LOCATION-docker.pkg.dev/$PROJECT_ID/snowplow/snowplow-dbt:$BUILD_ID',
          '.']
```

Step two is way easier luckily. We take the name (tag) of the container we just created and use that to upload it.
```yaml
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', '$LOCATION-docker.pkg.dev/$PROJECT_ID/snowplow/snowplow-dbt:$BUILD_ID']
```

Now for the juicy part. Setting up the actual job. We use the `gcloud` command. The jobs functionality is still in beta and we're naming it `snowplow-dbt` here. Also using the same location for our container image as where the job should run (not always necessary or possible). If you want to adjust the container image after creation, use `update` instead of `create`.
```yaml
- name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
  entrypoint: gcloud
  args: [
    'beta', 'run', 'jobs', 'create',
    'snowplow-dbt', '--image', '$LOCATION-docker.pkg.dev/$PROJECT_ID/snowplow/snowplow-dbt:$BUILD_ID',
    '--region', '$LOCATION']
```

Finally we use the same `gcloud` CLI to create a new schedule for our trigger. Note that the cron job schedule is `23 6-21/3 * * *`, i.e. the 23rd minute of every third hour between 6-21 (6:23, 9:23, etc.). Feel free to ask ChatGPT for any adjustments to your needs... Note also that the Cloud Run Job name is used here in the `uri` which you'll need to change if you have changed it in the previous step.
```yaml
- name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
  entrypoint: gcloud
  args: [
    'scheduler', 'jobs', 'create', 'http',
    'snowplow-dbt-schedule', 
    '--schedule', '23 6-21/3 * * *',
    '--location', '$LOCATION',
    '--uri', 'https://$LOCATION-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/$PROJECT_ID/jobs/snowplow-dbt:run',   
    '--http-method', 'POST',
    '--oauth-service-account-email', '$PROJECT_NUMBER-compute@developer.gserviceaccount.com'
    ]
```

With our `cloudbuild.yaml` file done we can run `builds submit` and hopefully watch it magically put itself together.

```
gcloud builds submit --substitutions=_BQ_DATASET="$BQ_DATASET" --region=europe-west1
```

And that's it. Unless you want to change anything to your dbt models or packages, you don't have to run the cloud build again. So just sit back and watch your dashboard update!

## Notes
* You can [find the full code as used on my site on Github](https://github.com/dumkydewilde/snowplow-minimal/tree/main/snowplow-dbt)
* Make sure to use Python 3.10 or lower when running dbt...
* Make sure the Snowplow package matches the dbt version e.g. pair dbt 1.3 with 12.2, but dbt 1.2 with 11.0
* I've had some issues downloading the Snowplow package before the Docker entrypoint, so it's been added in the entrypoint command instead.
* Make sure you don't accidentally copy your local packages to your Docker container as it will double-install them and throw errors. You can use .dockerignore to exclude `dbt_packages/**/*`
* Cloudbuild substitutions have to start with an underscore `_`
* dbt seems to have a preference for `.yml` files while GCP has a preference for `.yaml` they are usually interchangeable but not always...
* [This article by Christophe Oudar](https://medium.com/teads-engineering/setup-a-slim-ci-for-dbt-with-bigquery-and-docker-ce8e0a1a38f) expands the concept to a more advanced setup in a CI/CD pipeline using state comparison in dbt.
* You might also [find some more inspiration on the dbt forums](https://discourse.getdbt.com/t/publishing-dbt-docs-from-a-docker-container/141) and [in the dbt docs](https://docs.getdbt.com/docs/get-started/docker-install).
