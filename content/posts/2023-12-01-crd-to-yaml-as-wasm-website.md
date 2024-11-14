+++
author = "hannibal"
categories = ["golang", "wasm"]
date = "2023-12-01T01:01:00+01:00"
title = "CRD to YAML as WASM website"
url = "/2023/12/01/crd-to-yaml-as-wasm-website"
comments = true
+++

# CRD to YAML as WASM website

![logo](/img/2023/12/01/crdtoyamllogo.png)

A while ago, I wrote about [Generating Sample YAML files from CRDs](https://skarlso.github.io/2022/10/19/crd-to-yaml/).

It's a tool I created that lives [here](https://github.com/skarlso/crd-to-sample-yaml). It has a front-end service as
well for convenience.

I wrote it in a traditional client-server manner. It's running from a Docker Swarm container.

But, as I was thinking about it, nothing in this service requires interaction with a server. It gets some user input,
processes it, and has some output. I could have written it in plain Javascript. But, since I don't know JavaScript, or
don't know it well enough, and I do know GO, and I wanted to become more familiar with WASM, this was the perfect
learning opportunity.

The bonus is that if I pull it off, it will be a static website I can serve using GitHub Pages instead of my server.

## WASM

WASM has been available for a while now. You can use it with TinyGo or with plain Go or GopherJS. Each option has a
significant drawback of not being able to pass in higher-order values like strings. WASM will only accept pointers to
more complex variables, and that's just a pain in the butt to deal with.

Instead, I've gone with _another_ option called [go-app](https://go-app.dev/).

## Go-App

Go-App is a fantastic resource for building WASM-based SPAs. Getting started with it is a bit difficult. It's not
something that you can just straight dive into. You need to understand its flow and not force it to use a native flow
for a client-server setup.

First off, it's a SPA. The second, and maybe the most crucial part, is that it has its own update cycle of its components.
That means that instead of trying to update an innerHTML or something silly like that for a SPA, we do a conditional
rendering of components.

The update loop will update the components based on events and exported fields of the components. How does that look
like?

Let's take a look at a high-level `index` component that is the entry point to my site:

```go
func (i *index) Render() app.UI {
	// Prevent double rendering components.
  if !i.isMounted {
    return app.Main()
  }

  return app.Main().Body(app.Div().Class("container").Body(func() app.UI {
    if i.err != nil {
      return app.Div().Class("container").Body(&header{}, i.buildError())
    }

    if i.content != nil {
      return &crdView{content: i.content}
    }

    return app.Div().Class("container").Body(&header{}, &form{formHandler: i.OnClick})
  }()))
}

```

Let's break it down a bit.

First, the mount is a check, so it doesn't double-render my components. Rendering happens when entering `OnMount` event
AND when leaving. It doesn't bother my other components, but for `main` there seems to be a bug that causes some
double rending when refreshing. I solved it by adding a rendered check in my index component. Only render it once.

Second are the `if` statements. I update an internal error field if there is any error during rendering or fetching
content. If that error field isn't empty, I render an error message instead of the rest of the components. The error
message can be dismissed with an `OnClick` event like this:

```go

func (i *index) dismissError(ctx app.Context, e app.Event) {
	i.err = nil
}

```

Basically, just clear the internal error field. This function is called in the `buildError` above like this:

```go
func (i *index) buildError() app.UI {
	return app.Div().Class("alert alert-danger").Role("alert").Body(
		app.Span().Class("closebtn").OnClick(i.dismissError).Body(app.Text("×")),
		app.H4().Class("alert-heading").Text("Oops!"),
		app.Text(i.err.Error()),
	)
}
```

The event will ensure the `Render` fires; thus, the main component is re-rendered.

Where is the WASM thing, though, that I keep talking about? That's the best part. The `go-app` uses Go's own WASM file
building, and the wasm-js does all the heavy lifting. Internally, it will do all the transformation so you don't have
to care about it.

## Running it

To start the wasm local server, run:
```shell
make run
```

... from the [wasm folder](https://github.com/Skarlso/crd-to-sample-yaml/tree/147b4d31bba323ae2887e4e85242d8cea10c448c/wasm).

_Note_: I might move this folder around or rename it after writing this post.

This command will do two things. First, it will build the wasm binary. Second, it will build the main server.

The server will serve the binary and the static contents defined in the handler app.

## Generating static content

It's all nice and good, but I want statically servable content. And not _another_ server. `go-app` has our back.

To generate the static content, simply call the following function

```go
func generateGitHubPages(h *app.Handler) {
	if err := app.GenerateStaticWebsite(".", h); err != nil {
		panic(err)
	}
}
```

instead of firing off `ListenAndServer`. That's all there is to it. It will generate the following files:

```
tree
.
├── Makefile
├── app-worker.js
├── app.css
├── app.go
├── app.js
├── index.go
├── index.html
├── main.go
├── manifest.webmanifest
├── wasm
├── wasm_exec.js
└── web
    ├── app.wasm
    ├── css
    │   ├── alert.css
    │   ├── halfmoon-variables.min.css
    │   ├── main.css
    │   ├── prism-okaidia.css
    │   ├── prism.css
    │   └── root.css
    ├── img
    │   └── logo.png
    └── js
        ├── clipboard.min.js
        ├── prism.js
        └── wasm_exec.js

5 directories, 22 files
```

The additions are app* files, `index.html`, the manifest file and the `wasm_exec.js` file. This folder will be the root
from which we can serve the site from GitHub pages.

## GitHub Pages

All we need to do is set up some actions to serve the site data from the desired folder. And that's it. GitHub
will pick it up and serve the site correctly. I gave it a temporary home here: https://crdtoyaml.github.io/.

Pending some trouble with my DNS provider, the right location will be https://crdtoyaml.com.

## Conclusion

And that's all. My site is now static and hosted on GitHub. One less thing I need to maintain. It's far from perfect and
there are more things I want to improve in using the components correctly. And acquiring the value from the input box or
the textarea could be done better.

But I'm rather proud of it anyways. And I learned a lot for future projects I would like to work on.

Thanks for reading!
Gergely.
