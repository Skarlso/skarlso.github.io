+++
author = "hannibal"
categories = ["Golang", "AWS"]
date = "2017-03-22T12:03:00+01:00"
title = "Furnace - The building of an AWS CLI Tool for CloudFormation and CodeDeploy - Part 3"
url = "/2017/03/22/building-furnace-part-3"

+++

# Intro

Hi folks.

Previously on this blog: [Part 1](http://skarlso.github.io/2017/03/16/building-furnace-part-1/). [Part 2](https://skarlso.github.io/2017/03/19/building-furnace-part-2/). [Part 4](https://skarlso.github.io/2017/04/16/building-furnace-part-4/).


In this part, I'm going to talk about the experimental plugin system of Furnace.

# Go Experimental Plugins

Since Go 1.8 was released, an exciting and new feature was introduced called a Plug-in system. This system works with dynamic
libraries built with a special switch to `go build`. These libraries, `.so` or `.dylib` (later), are than loaded and once that
succeeds, specific functions can be called from them (symbol resolution).

We will see how this works. For package information, visit the plugin packages Go doc page
[here](https://tip.golang.org/pkg/plugin/).

# Furnace Plugins

So, what does furnace use plugins for? Furnace uses plugins to execute arbitery code in, currently, four given locations / events.

These are: `pre_create, post_create, pre_delete, post_delete`. These events are called, as their name suggests, before and after
the creation and deletion of the CloudFormation stack. It allows the user to execute some code without having to rebuild the whole
project. It does that by defining a single entry point for the custom code called `RunPlugin`. Any number of functions can be
implemented, but the plugin MUST provide this single, exported function. Otherwise it will fail and ignore that plugin.

## Using Plugins

It's really easy to implement, and use these plugins. I'm not going into the detail of how to load them, because that is done by
Furnace, but only how to write and use them.

To use a plugin, create a go file called: `0001_mailer.go`. The `0001` before it will define WHEN it's executed.
Having multiple plugins is completely okay. Execution of order however, depends on the names of the files.

Now, in 0001_mailer.post_create we would have something like this:

~~~go
package main

import "log"

// RunPlugin runs the plugin.
func RunPlugin() {
	log.Println("My Awesome Pre Create Plugin.")
}
~~~

Next step is the build this file to be a plugin library. Note: Right now, this only works on Linux!

To build this file run the following:
~~~
go build -buildmode=plugin -o 0001_mailer.pre_create 0001_mailer.go
~~~

The important part here is the extension of the file specified with `-o`. It's important because that's how Furnace identifies
what plugins it has to run.

Finally, copy this file to `~/.config/go-furnace/plugins` and you are all set.

## Slack notification Plugin

To demonstrate how a plugin could be used is if you need some kind of notification once a Stack is completed. For example, you
might want to send a message to a Slack room. To do this, your plugin would look something like this:

~~~go
package main

import (
	"fmt"
	"os"

	"github.com/nlopes/slack"
)

func RunPlugin() {
	stackname := os.Getenv("FURNACE_STACKNAME")
	api := slack.New("YOUR_TOKEN_HERE")
	params := slack.PostMessageParameters{}
	channelID, timestamp, err := api.PostMessage("#general", fmt.Sprintf("Stack with name '%s' is Done.", stackname), params)
	if err != nil {
		fmt.Printf("%s\n", err)
		return
	}
	fmt.Printf("Message successfully sent to channel %s at %s", channelID, timestamp)
}
~~~

Currently, Furnace has no ability to share information of the stack with an outside plugin. Thus 'Done' could be anything from
Rollback to Failed to CreateComplete.

# Closing Words

That's it for plugins. Thanks very much for reading!
Gergely.

