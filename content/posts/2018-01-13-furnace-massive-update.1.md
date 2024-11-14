+++
author = "hannibal"
categories = ["AWS", "Go", "Furnace", "GCP"]
date = "2018-01-13T22:34:00+01:00"
title = "Huge Furnace Update"
url = "/2018/01/13/furnace-massive-update"
description = ""
featured = "furnace.png"
featuredalt = "Furnace"
featuredpath = "date"
linktitle = ""
+++

# Intro

Hi folks.

In the past couple of months I've been slowly updating [Furnace](https://github.com/Skarlso/go-furnace).

There are three major changes that happened. Let's take a look at them, shall we?

## Google Cloud Platform

Furnace now supports [Google Cloud Platform (GCP)](https://cloud.google.com). It provides the same API to handle GCP resource as with AWS. Namely, `create`, `delete`, `status`, `update`. I opted to leave out `push` because Google mostly works with git based repositories, meaning a push is literary just a push, than Google handles distributing the new code by itself.

All the rest of the commands should work the same way as AWS.

### Deployment Manager

GCP has a similar service to AWS CloudFormations called [Deployment Manager](https://cloud.google.com/deployment-manager/docs/). The documentation is fairly detailed with a Bookshelf example app to deploy. Code and Templates can be found in their Git repositroy here: [Deployment Manager Git Repository](https://github.com/GoogleCloudPlatform/deploymentmanager-samples).

### Setting up GCP

As the README of Furnace outlines...

> Please carefully read and follow the instruction outlined in this document: [Google Cloud Getting Started](https://cloud.google.com/sdk/#Quick_Start). It will describe how to download and install the SDK and initialize cloud to a Project ID.

> Take special attention to these documents:

> [Initializing GCloud Tools](https://cloud.google.com/sdk/docs/initializing)
> [Authorizing Tools](https://cloud.google.com/sdk/docs/authorizing)

> Furnace uses a Google Key-File to authenticate with your Google Cloud Account and Project.
>In the future, Furnace assumes these things are properly set up and in working order.

To initialize the client, it uses the following code:

~~~go
  ctx := context.Background()
  client, err := google.DefaultClient(ctx, dm.NdevCloudmanScope)
~~~

The DefaultClient in turn, does the following:

~~~go
// FindDefaultCredentials searches for "Application Default Credentials".
//
// It looks for credentials in the following places,
// preferring the first location found:
//
//   1. A JSON file whose path is specified by the
//      GOOGLE_APPLICATION_CREDENTIALS environment variable.
//   2. A JSON file in a location known to the gcloud command-line tool.
//      On Windows, this is %APPDATA%/gcloud/application_default_credentials.json.
//      On other systems, $HOME/.config/gcloud/application_default_credentials.json.
//   3. On Google App Engine it uses the appengine.AccessToken function.
//   4. On Google Compute Engine and Google App Engine Managed VMs, it fetches
//      credentials from the metadata server.
//      (In this final case any provided scopes are ignored.)
func FindDefaultCredentials(ctx context.Context, scope ...string) (*DefaultCredentials, error) {
~~~

Take note on the order. This is how Google will authenticate your requests.

### Running GCP

Running gcp is largely similar to AWS. First, you create the necessary templates to your infrastructure. This is done via the Deployment Manager and it's templating engine. The GCP templates are Python [JINJA](http://jinja.pocoo.org/) files. Examples are provided in the `template` directory. It's a bit more complicated than the CloudFormation templates in that it uses outside templates plus schema files to configure dynamic details.

It's all explained in these documents: [Creating a Template Step-by-step](https://cloud.google.com/deployment-manager/docs/step-by-step-guide/create-a-template) and [Creating a Basic Template](https://cloud.google.com/deployment-manager/docs/configuration/templates/create-basic-template).

It's not trivial however. And using the API can also be confusing. The Google Code is just a generated Go code file using gRPC. But studying it may provide valuable insigth into how the API is structured. I'm also providing some basic samples that I gathered together and the readme does a bit more explaining on how to use them.

### Your First Stack

Once you have everything set-up you'll need a configuration file for Furnace. The usage is outlined more here [YAML Configuration](#YAML-Configuration). The configuration file for GCP looks like this:

~~~yaml
main:
  project_name: testplatform-1234
  spinner: 1
gcp:
  template_name: google_template.yaml
  stack_name: test-stack

~~~

Where `project_name` is the name you generate for your first billable Google Cloud Platform project. Template lives next to this yaml file and stack name must be DNS complient.

Once you have a project and a template setup, it's as simple as calling `./furnace-gcp create` or `./furnace-gcp create mycustomstack`.

### Deleting

Deleting happens with `./furnace-gcp delete` or `./furnace-gcp delete mycustomstack`. Luckily, as with AWS, this means that every resource created with the DeploymentManager will be deleted leaving no need for search and cleanup.

### Project Name vs. Project ID

Unlike with AWS Google requires your stack name and project id to be DNS complient. This is most likely because all API calls and such contain that information.

## Separate Binaries

In order to mitigate some of Furnace's size, I'm providing separate binaries for each service it supports.

The AWS binaries can be found in `aws` folder, and respectively, the Google Cloud Platform is located in `gcp`. Both are build-able by running `make`.

If you would like to run both with a single command, a top level make file is provided for your convinience. Just run `make` from the root. That will build all binaries. Later on, Digital Oceans will join the ranks.

## YAML Configuration

Last but not least, Furnace now employs YAML files for configuration. However, it isn't JUST using YAML files. It also employs a smart configuration pattern which works as follows.

Since Furnace is a distributed binary file which could be running from any given location at any time. Because of that, at first I opted for a global configuration directory.

Now, however, furnace uses a furnace configuration file named with the following pattern: `.stackalias.furnace`. Where stackname, or stack is the name of a custom stack you would like to create for a project. The content of this file is a single entry, which is the location, relative to this file, of the YAML configuration files for the given stack. For example:

~~~bash
stacks/mydatabasestack.yaml
~~~

This means, that in the directory called `stacks` there will a yaml configuration file for your database stack. The AWS config file looks like this:

~~~YAML
main:
  stackname: FurnaceStack
  spinner: 1
aws:
  code_deploy_role: CodeDeployServiceRole
  region: us-east-1
  enable_plugin_system: false
  template_name: cloud_formation.template
  app_name: furnace_app
  code_deploy:
    # Only needed in case S3 is used for code deployment
    code_deploy_s3_bucket: furnace_code_bucket
    # The name of the zip file in case it's on a bucket
    code_deploy_s3_key: furnace_deploy_app
    # In case a Git Repository is used for the application, define these two settings
    git_account: Skarlso/furnace-codedeploy-app
    git_revision: b89451234...

~~~

The important part is the `template_name`. The template has to be next to this yaml file. To use this file, you simply call any of the AWS or GCP commands with an extra, optional parameter like this:

~~~bash
./furnace-aws create mydatabase
~~~

Note that mydatabase will translate to `.mydatabase.furnace`.

The intelligent part is, that this file could be placed anywhere in the project folder structure; because furnace, when looking for a config file, traverses backwards from the current execution directory up until `/`. Where root is not included in the search.

Consider the following directory tree:

├── docs
│   ├── `furnace-aws status mydatabase`
├── stacks
│   ├── mystack.template
│   └── mystack.yaml
└── .mydatabase.furnace

You are currently in your `docs` directory and would like to ask for the status of your database. You don't have to move to the location of the setting file, just simply run the command from where you are. This only works if you are above the location of the file. If you would be below, furnace would say it can't find the file. Because it only traverses upwards.

`.mydatabase.furnace` here contains only a single entry `stacks/mystack.yaml`. And that's it. This way, you could have multiple furnace files, for example a `.database.furnace`, `.front-end.furnace` and a `.backend.furnace`. All three would work in unison, and if want needs updating, simply run `./furnace-aws update backend`. And done!

# Closing words

As always, contributions are welcomed in the form of issues or pull requests. Questions anything, I tend to answer as soon as I can.

Always run the tests before submitting.

Thank you for reading.
Gergely.
