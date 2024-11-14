+++
author = "hannibal"
categories = ["Golang", "AWS"]
date = "2017-03-19T12:03:00+01:00"
title = "Furnace - The building of an AWS CLI Tool for CloudFormation and CodeDeploy - Part 2"
url = "/2017/03/19/building-furnace-part-2"

+++

# Intro

Hi folks.

Previously on this blog: [Part 1](https://skarlso.github.io/2017/03/16/building-furnace-part-1/), [Part 3](https://skarlso.github.io/2017/03/22/building-furnace-part-3/), [Part 4](https://skarlso.github.io/2017/04/16/building-furnace-part-4/)

In this part, I'm going to talk about the AWS Go SDK and begin do dissect the intricacies of Furnace.

# AWS SDK

Fortunately, the Go SDK for AWS is quiet verbose and littered with examples of all sorts. But that doesn't make it less complex
and less cryptic at times. I'm here to lift some of the early confusions, in hopes that I can help someone to avoid wasting time.

## Getting Started and Developers Guide

As always, and common from AWS, the documentation is top notch. There is a 141 pages long developer's guide on the SDK containing
a getting started section and an API reference. Go check it out. I'll wait. [AWS Go SDK DG PDF](http://docs.aws.amazon.com/sdk-for-go/v1/developer-guide/aws-sdk-go-dg.pdf). I will only talk about some gotchas and things I encountered, not the basics of the SDK.

## aws.String and other types

Something which is immediately visible once we take a look at the API is that everything is a pointer. Now, there are a
tremendous amount of discussions about this, but I'm with Amazon. There are various reasons for it, but to list the most prominent
ones:
    - Type completion and compile time type safety.
    - Values for AWS API calls have valid zero values, in addition to being optional, i.e. not being provided at all.
    - Other option, like, empty interfaces with maps, or using zero values, or struct wrappers around every type, made life much
       harder rather than easier or not possible at all.
    - The AWS API is volatile. You never know when something gets to be optional, or required. Pointers made that decision easy.

There are good number of other discussions around this topic, for example: [AWS Go GitHub #363](https://github.com/aws/aws-sdk-go/issues/363).

In order to use primitives, AWS has helper functions like `aws.String`. Because &"asdf" is not allowed, you would have to create a
variable and use its address in situations where a string pointer is needed, for example, name of the stack. These primitive helpers will
make in-lining possible. We'll see later that they are used to a great extent. Pointers, however, make life a bit difficult when
constructing Input structs and make for poor aesthetics.

This is something I'm returning in a test for stubbing a client call:
~~~go
		return &cloudformation.ListStackResourcesOutput{
			StackResourceSummaries: []*cloudformation.StackResourceSummary{
				{
					ResourceType:       aws.String("NoASG"),
					PhysicalResourceId: aws.String("arn::whatever"),
				},
			},
		}
~~~

This doesn't look so appealing, but one gets used to it quickly.

## Error handling

Errors also have their own types. An AWS error looks like this:

~~~go
if err != nil {
    if awsErr, ok := err.(awserr.Error); ok {
    }
}
~~~

First, we check if error is nil, than we type check if the error is an AWS error or something different. In the wild, this will
look something like this:

~~~go
	if err != nil {
		if awsErr, ok := err.(awserr.Error); ok {
			if awsErr.Code() != codedeploy.ErrCodeDeploymentGroupAlreadyExistsException {
				log.Println(awsErr.Code())
				return err
			}
			log.Println("DeploymentGroup already exists. Nothing to do.")
			return nil
		}
		return err
	}
~~~

If it's an AWS error, we can check further for the error code that it returns in order to identify what to handle, or what to throw
on to the caller to a potential fatal. Here, I'm ignoring the AlreadyExistsException because, if it does, we just go on to a next
action.

## Examples

Luckily the API doc is very mature. In most of the cases, they provide an example to an API call. These examples, however, from
time to time provide more confusion than clarity. Take CloudFormation. For me, when I first glanced upon the
description of the API it wasn't immediately clear that the `TemplateBody` was supposed to be the whole template, and that
the rest of the fields were almost all optional settings. Or provided overrides in special cases.

And since the template is not an ordinary JAML or JSON file, I was looking for something that parses it into that the Struct I
was going to use. After some time, and digging, I realized that I didn't need that, and that I just need to read in the template,
define some extra parameters, and give the TemplateBody the whole of the template. The parameters defined by the CloudFormation
template where extracted for me by `ValidateTemplate` API call which returned all of them in a convenient
`[]*cloudformation.Parameter` slice. These things are not described in the document or visible from the examples. I mainly found
them through playing with the API and focused experimentation.

## Waiters

From other SDK implementations, we got used to Waiters. These handy methods wait for a service to become available or for certain
situations to take in effect, like a Stage being `CREATE_COMPLETE`. The Go waiters, however, don't allow for callback to be fired,
or for running blocks, like the ruby SDK does. For this, I wrote a handy little waiter for myself, which outputs a spinner to see
that we are currently waiting for something and not frozen in time. This waiter looks like this:

~~~go
// WaitForFunctionWithStatusOutput waits for a function to complete its action.
func WaitForFunctionWithStatusOutput(state string, freq int, f func()) {
	var wg sync.WaitGroup
	wg.Add(1)
	done := make(chan bool)
	go func() {
		defer wg.Done()
		f()
		done <- true
	}()
	go func() {
		counter := 0
		for {
			counter = (counter + 1) % len(Spinners[config.SPINNER])
			fmt.Printf("\r[%s] Waiting for state: %s", yellow(string(Spinners[config.SPINNER][counter])), red(state))
			time.Sleep(time.Duration(freq) * time.Second)
			select {
			case <-done:
				fmt.Println()
				break
			default:
			}
		}
	}()

	wg.Wait()
}
~~~

And I'm calling it with the following method:

~~~go
	utils.WaitForFunctionWithStatusOutput("DELETE_COMPLETE", config.WAITFREQUENCY, func() {
		cfClient.Client.WaitUntilStackDeleteComplete(describeStackInput)
	})
~~~

This would output these lines to the console:

~~~bash
[\] Waiting for state: DELETE_COMPLETE
~~~

The spinner can be configured to be one of the following types:

~~~go
var Spinners = []string{`←↖↑↗→↘↓↙`,
	`▁▃▄▅▆▇█▇▆▅▄▃`,
	`┤┘┴└├┌┬┐`,
	`◰◳◲◱`,
	`◴◷◶◵`,
	`◐◓◑◒`,
	`⣾⣽⣻⢿⡿⣟⣯⣷`,
	`|/-\`}
~~~

Handy.

And with that, let's dive into the basics of Furnace.

# Furnace

## Directory Structure and Packages

Furnace is divided into three main packages.

### commands

Commands package is where the gist of Furnace lies. These commands represent the commands which are used through the CLI. Each
file has the implementation for one command. The structure is devised by this library: [Yitsushi's Command Library](https://github.com/Yitsushi/go-commander).
As of the writing of this post, the following commands are available:
- create - Creates a stack using the CloudFormation template file under ~/.config/go-furnace
- delete - Deletes the created Stack. Doesn't do anything if the stack doesn't exist
- push - Pushes an application to a stack
- status - Displays information about the stack
- delete-application - Deletes the CodeDeploy application and deployment group created by `push`

These commands represent the heart of furnace. I would like to keep these to a minimum, but I do plan on adding more, like
`update` and `rollout`. Further details and help messages on these commands can be obtained by running: `./furnace help` or
`./furnace help create`.

~~~bash
❯ ./furnace help push
Usage: furnace push appName [-s3]

Push a version of the application to a stack

Examples:
  furnace push
  furnace push appName
  furnace push appName -s3
  furnace push -s3
~~~

### config

Contains the configuration loader and some project wide defaults which are as follows:
- Events for the plugin system - `pre-create`, `post-create`, `pre-delete`, `post-delete`.
- CodeDeploy role name - `CodeDeployServiceRole`. This is used if none is provided to locate the CodeDeploy IAM role.
- Wait frequency - Is the setting which controls how long the waiter should sleep in between status updates. Default is `1s`.
- Spinner - Is just the number of the spinner to use.
- Plugin registry - Is a map of functions to run for the above events.

Further more, config loads the CloudFormation template and checks if some necessary settings are present in the environment, exp:
the configuration folder under `~/.config/go-furnace`.

### utils

These are some helper functions which are used throughout the project. To list them:
- error_handler - Is a simple error handler. I'm thinking of refactoring this one to some saner version.
- spinner - Sets up which spinner to use in the waiter function.
- waiter - Contains the verbose waiter introduced above under [Waiters](##Waiters).

## Configuration and Environment variables

Furnace is a Go application, thus it doesn't have the luxury of Ruby or Python where the configuration files are usually bundled
with the app. But, it does have a standard for it. Usually, configurations reside in either of these two locations. Environment
Properties or|and configuration files under a fixed location ( i.e. HOME/.config/app-name ). Furnace employs both.

Settings like, region, stack name, enable plugin system, are under environment properties ( though this can change ), while the
CloudFormation template lives under `~/.config/go-furnace/`. Lastly it assumes some things, like the Deployment IAM role just
exists under the used AWS account. All these are loaded and handled by the config package described above.

## Usage

A typical scenario for Furnace would be the following:

- Setup your CloudFormation template or use the one provided. The one provided sets up a highly available and self healing setting
  using Auto-Scaling and Load-Balancing with a single application instance. Edit this template to your liking than copy it to
`~/.config/go-furnace`.
- Create the configured stack with `./furnace create`.
- Create will ask for the parameters defined in the template. If defaults are setup, simply hitting enter will use these defaults.
  Take note, that the provided template sets up SSH access via a provided key. If that key is not present in CF, you won't be able
to SSH into the created instance.
- Once the stack is completed, the application is ready to be pushed. To do this, run: `./furnace push`. This will locate the
  appropriate version of the app from S3 or GitHub and push that version to the instances in the Auto-Scaling group. To all of
them.

## General Practices Applied to the Project

### Commands

For each command the main entry point is the `execute` function. These functions are usually calling out the small chunks of
distributed methods. Logic was kept to a bare minimum ( probably could be simplified even further ) in the execute functions
mostly for testability and the likes. We will see that in a followup post.

### Errors

Errors are handled immediately and usually through a fatal. If any error occurs than the application is halted. In followup
versions this might become more granular. I.e. don't immediately stop the world, maybe try to recover, or create a Poller or
Re-Tryer, which tries a call again for a configured amount of times.

### Output colors

Not that important, but still... Aesthetics. Displaying data to the console in a nice way gives it some extra flare.

### Makefile

This project works with a Makefile for various reasons. Later on, once the project might become more complex, a Makefile makes it
really easy to handle different ways of packaging the application. Currently, for example, it provides a `linux` target which will
make Go build the project for Linux architecture on any other Architecture i.e. cross-compiling.

It also provides an easy way to run unit tests with `make test` and installing with `make && make install`.

# Closing Words

That is all for Part 2. Join me in Part 3 where I will talk about the experimental Plugin system that Furnace employs.

Thank you for reading!
Gergely.
