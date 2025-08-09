+++
author = "hannibal"
categories = ["go", "programming"]
date = "2025-06-25T01:01:00+01:00"
title = "REST based, interface focused Go plugin system using UDS or TCP"
url = "/2025/06/25/rest-based-interface-focused-go-plugin-system"
comments = false
draft = true
+++

# REST based, interface focused Go plugin system using UDS or TCP

Hello everyone.

Today, I would like to show a plugin system I've been working on for Go that's intuitive to use
and implement from the Plugin author's side and intuitive to use from the Library's side.

The communication is REST based with the connection being either UDS ( Unix Domain Socket ) or TCP
based. Depending on availability but defaulting / starting with the socket approach. The plugin
is inherently local, so there are little or no concerns for security at this point. Also, both connection
types allow for multiplexing therefore reusing a socket or a connection allows for fast communication
between the server and the client.

Let's get into it.

## Basics

Why write a plugin system? Well, Go doesn't really have a stable, usable plugin system. It's own plugins
are basically abandoned; HashiCorp's go-plugin is overly complex with its GRPC contracts; Lua, and other
script language based plugins are difficult to handle, but provide a "good enough" alternative; binaries
that you basically just blindly execute ( the kubectl approach ).

What all of these are missing is that the plugin's write has little to no assurance of how to write the plugin
and what to implement and what to expect. ( Okay, not counting GRPC because the protobuf plugin is really nice. )



### Setup

### Capabilities

### Endpoints

### Wrappers

## Client SDK

### Startup

### Idle Checker

### Shutdown

## Usage

### Implementation

### Caller side

### Library

# Conclusion
