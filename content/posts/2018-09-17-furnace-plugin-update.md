+++
author = "hannibal"
categories = ["Golang", "Furnace"]
date = "2018-09-17T07:01:00+01:00"
title = "Furnace with a new Plugin System"
url = "/2018/09/17/furnace-plugin-update"
description = "Furnace plugins"
featured = "furnace.png"
featuredalt = "furnace"
featuredpath = "date"
linktitle = ""
draft = false
+++

Hi.

A quick update, but a very important and interesting one hopefully. Furnace just got a massive boost to its plugin system.

I'm using [HashiCorp's Go-Plugins](https://github.com/hashicorp/go-plugin) system now to handle plugins. This means one of
two things that are interesting to the plugin author.

One, plugins can be written in any language which is supported by Furnace and supports GRPC. Currently this means that
plugins can be written in the following languages:

* Go

* Python

* Ruby

Adding new plugins is easy and I'm open for suggestions in which language to provide next if the need arrises.

To find out more, please read the README on Furnace about plugins located here: [Furnace Plugin System](https://github.com/go-furnace/go-furnace/blob/master/README.md#plugins).

I hope to see a bunch of nice plugins pop up here and there if please are interested in writing them. I'm listing a couple of
possibilities like, notification after create, or resource cleanup or even preventing the stack from creating in the first place
with a pre-create check for permissions / resource availability / funds constraints.

Have fun writing plugins and making Furnace more powerful then ever.

I'm planning on providing some basic plugins that could be used out of the box. Those will probably be in Go though.

Thanks,
Gergely.
