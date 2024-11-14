+++
author = "hannibal"
categories = ["golang"]
date = "2020-02-17T21:01:00+01:00"
title = "How to Make SPA refresh work with a Go backend"
url = "/2020/02/17/making-spa-refresh-work-with-go-backend"
comments = true
+++

# Intro

Hi folks.

Today I would like to share a quick "fix" for a problem I've seen popping up here and there.

That is, if you have a react frontend which is a SPA app but you still want refresh to work.
What do I mean by that? Consider the following...

# The problem

You have a SPA app with a react router which navigates the user around. The app calls to a backend
api which serves content of some kind. You have the following routes.... login, signup, reset, archive.

If your app is compiled with your backend, as it usually is, then something like: https://app.com/login
will not work unless it's also defined on the backend serving some content.

So but what should the content be in this case?

# The structre

For that, let's first look at the strucute of the app. Consider the following directory tree:

~~~
.
├── Dockerfile
├── LICENSE
├── Makefile
├── README.md
├── build
├── cmd
│   └── root.go
├── frontend
│   ├── LICENSE
│   ├── README.md
│   ├── build
│   ├── package-lock.json
│   ├── package.json
│   ├── public
│   ├── src
│   └── yarn.lock
├── go.mod
├── go.sum
├── img
├── internal
└── pkg
~~~

For this, the frontend contains a build dir in which the generated react frontend static files plus
compiled JavaScript libraries are. In this directory there also is a index.html file which does the actual
heavy lifting in terms of routing.

The Go backend therefor must only route to index.html on certain endpoints.

In Go to build and deploy a single binary containing the static assets here in, you can use something like
[go.rice](https://github.com/GeertJohan/go.rice) or [assetfs](https://github.com/elazarl/go-bindata-assetfs) which
generate a Go file for you which contains all the data in an easily accessible way.

I'll be using go.rice.

# The solution

To summarize, all you have to do is route every route in your router.js file to index.html in Go. But how? Well, like this...

Consider this appliction: [Staple](https://github.com/staple-org/staple). This is a react frontend go backend application
which builds a frontend asset then packages it up with go.rice, builds a Docker container and deploys the whole thing to
a Kubernetes cluster. But this is the interesting part which handles the index routing:

In routes.go (contains the mapped routes from under Router.js):

~~~go
package pkg

// These routes must match the routes under frontend/Routes.js
var routes = []string{
	"/login",
	"/archive",
	"/staples/new",
	"/reset",
	"/signup",
	"/settings",
}
~~~

Once we have a list of routes to map...

In server.go (which is starting up the server and generates the handlers...)

~~~go
    // ... code which sets up the api routes... after every handler has been estabilished...
	// Setup front-end if not in production mode.
	if !config.Opts.DevMode {
        // This path needs to be relative from this files package's location.
		staticAssets, err := rice.FindBox("../frontend/build")
		if err != nil {
			log.Fatal("Cannot find assets in production")
			return err
		}

		// Register handler for static assets
        assetHandler := http.FileServer(staticAssets.HTTPBox())
        // Open the index.html file as a *File for reading the content out of it.
        // This is a virtual file handled by go.rice.
		index, err := staticAssets.Open("index.html")
		if err != nil {
			config.Opts.Logger.Error().Err(err).Msg("Failed to find index.html content.")
			return err
		}

        // Set up the main point as a static file server
		e.GET("/", echo.WrapHandler(assetHandler))
		// Set up routes to index.html for all routes under Routes.js. Index.html will handle the routing any further.
		for _, r := range routes {
			e.GET(r, indexServer(r, index))
		}
		e.GET("/favicon.ico", echo.WrapHandler(assetHandler))
		e.GET("/site.webmanifest", echo.WrapHandler(assetHandler))
		e.GET("/static/css/*", echo.WrapHandler(http.StripPrefix("/", assetHandler)))
		e.GET("/static/js/*", echo.WrapHandler(http.StripPrefix("/", assetHandler)))
		e.GET("/static/media/*", echo.WrapHandler(http.StripPrefix("/", assetHandler)))
    }
~~~

What is `indexServer` in this you might ask? Well, fret no longer, I shall show you:

~~~go
// indexServer takes a name and the contents of the virtual file index.html gathered up by go.rice
// and serves its content via http.ServeContent under the given name.
func indexServer(name string, file *rice.File) echo.HandlerFunc {
	return func(c echo.Context) error {
		stat, _ := file.Stat()
		http.ServeContent(c.Response().Writer, c.Request(), name, stat.ModTime(), file)
		return nil
	}
}
~~~

The key points are the name, which will be the route and the file which is the index.html content which contains
the logic to route based on the request. All that will be handled. And if a new route comes along,
simple add it to the list, recompile and you are done!

# Conclusion

In summary, you let your index.html file handle the routing as you would normally do. Just you need to make your
backend aware of that fact. Now refreshing the page will work as you'd expect.

Thank you for reading,
Gergely.