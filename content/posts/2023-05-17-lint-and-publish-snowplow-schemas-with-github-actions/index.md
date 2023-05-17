---
title: "Automatically Lint and Publish your Snowplow Schemas with Github Actions"
date: "2023-05-17"
tags: 
  - Snowplow
  - json-schema
  - github-actions
  - yaml
description: "Snowplow schemas are a great way to codify expected data in JSON format. Using Github actions you can 
make them eevn more powerful by automatically checking for typos, validity, and other errors as well as directly
publishing them to your production environment with no manual action." 
---

Imagine this: your website team has developed a great new feature. The developers are proud, the product owner is happy, and now they come to you to hear how it's impacting sessions and conversions. You look at the data and realise they messed up the tracking implementation.

Snowplow fixes a lot of these tracking problems by giving you the ability to work with schemas, a codified description of expected data. For example the schema with the expected values of a booking appointment event could be described in a JSON format similar to this:

```json
{
	"description" : "A booking interaction event",
	  "self": {
	    "name": "booking_event",                                                                                  
	    "version": "1-0-1"
	  },
	"properties": {
		"booking_interaction": {
		  "type": "string",
		  "maxLength": 128,
		  "description": "The type of booking (e.g. appointment, cancellation)."
		},
		"booking_id": {
		  "type": ["string", "null"],
		  "maxLength": 128,
		  "description": "The ID of the booking."
		}
	},
	"required": [
      "booking_interaction"
    ],
}
```

As you can see our schema describes that a `booking_event` must have at least a `booking_interaction` and optionally a `booking_id`. This schema gives us incredible power because we have a version controlled definition of expected data. Our developers can use this to test the event tracking they are implementing, our collection pipeline can use this to split out good and bad events as they come in, and in our analytics tool we can use this to define tables and models.

Of course all of this hinges on the quality and availability of your schemas which is why it is essential to remove any errors as early as possible and to make sure they are available in to all actors and components in your data pipeline through the schema server as soon as possible. Let me show you how to do that with Github Actions.

## Checking the validity of schemas in your repository
Github, of course, allows you to store code and files in a version controlled repository, and like many git providers, they also allow you to create workflows based on changes to that repository. So if we create a pull request to integrate an updated version of our schema into our main branch, we can kick off a workflow to do check if the new schema is actually valid and in the right format, a process called 'linting'. 
When we then merge the pull request and update our main branch —the single source of truth for our schemas—, we can kick off another workflow to update our schema server, where all the production schemas can be accessed.

First off, let's look at the pull request workflow. You will see that it actually does 4 things.
1. It checks that the pull request is on the main branch, and not another branch
2. It checks out the files in the repository so the script can access them
3. It installs `igluctl` the schema tool from snowplow
4. It lints using `igluctl`

```yaml
name: Lint schema
on:
  pull_request:
    branches: [ "main" ]

jobs:
  lint-schemas:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install igluctl
        run: |
	        wget https://github.com/snowplow/igluctl/releases/download/0.11.0/igluctl_0.11.0.zip 
	        unzip igluctl_0.11.0.zip
	        chmod +x ./igluctl 

      - name: Lint all schemas
        run: ./igluctl lint --skip-checks stringLength,numericMinMax schemas

```

Let's call this file `lint-schema.yml` so we can add this to our repository as `.github/lint-schema.yml`. Now the next time we add a pull request to the main branch the steps will be executed.

## Publishing schemas to our production Iglu Server
If you want to be able to sleep at night and don't worry about your work on the weekends, you should try to get the amount of manual interactions with your production environment to zero. In other words, any time there is a manual intervention on a production environment, you run the chance that either an error is introduced or discrepencies between different environments (dev and prod) accumulate over time eventually causing errors or outages. 
Using automations like Github Actions allows you to fully keep your hands of a production environment and make sure any changes are always deployed in the exact same way. For our schema server that means the write API key that we use for deploying new schemas should only available to the Github Action. We can achieve this by adding a 'secret' called `IGLU_API_WRITE_KEY` in our settings that we can reference in our YAML file by calling `${{ secrets.IGLU_API_WRITE_KEY }}`. And while we're at it, we'll also add in our Iglu server URL as a variable, so we can easily update it if needed and reference it as `${{ vars.IGLU_SERVER }}`
If you're wondering how to get the write key. You can also use `igluctl` locally to generate a new set of read and write keys by running the following command:
`./igluctl server keygen --vendor-prefix custom.vendor.name my.iglu.server/server-path <MASTER_API_KEY>`

For our Github Action we now have the following configuration:

```yaml
name: Publish schema
on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

jobs:
  publish-schemas:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install igluctl
        run: |
	        wget https://github.com/snowplow/igluctl/releases/download/0.11.0/igluctl_0.11.0.zip 
	        unzip igluctl_0.11.0.zip
	        chmod +x ./igluctl 

      - name: Lint all schemas
        run: ./igluctl lint --skip-checks stringLength,numericMinMax schemas

      - name: Push schemas to PROD on merge
        if: ${{ github.ref == 'refs/heads/main' }}
        run: ./igluctl static push --public schemas ${{ vars.IGLU_SERVER }} ${{ secrets.IGLU_API_WRITE_KEY }}

```

You can see we've changed the first `on` part from `pull_request` to `push`, and we check `github.ref` to make sure we're referencing the correct branch. Wait, but why then are we linting again? Well, first of all it is just good practice to double check, but secondly, not every change might go through a pull request. Imagine you just did a pull request where an error still made it through. You now might want to deploy a hotfix and commit directly on the main branch. This minor step of linting will make sure you don't accidentally deploy something with a typo or wrong indentation. 

## Final thoughts
It may seem simple, but having basic automation in your pipeline will make your life a lot easier down the line. 
You can obviously adjust this setup both to add in more steps to this pipeline —maybe you also want to add in a development environment, 
or add more checks on the JSON files and the general repo structure with [pre-commit](https://pre-commit.com/)— or you can
use a similar setup for other tools and repositories that you are using, like dbt or Terraform.