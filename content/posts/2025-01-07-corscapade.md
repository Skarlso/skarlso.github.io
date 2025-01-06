+++
author = "hannibal"
categories = ["cty", "golang", "cors"]
date = "2025-01-07T01:01:00+01:00"
title = "CORScapade; the story why cty doesn't support git flow on web"
url = "/2025/01/07/corscapade"
comments = true
+++

# CORScapade; the story why cty doesn't support git flow on web

Recently, I implemented git based discovery for [cty](https://github.com/Skarlso/crd-to-sample-yaml). It means, that the user can provide
a git repo URL and cty will clone the content and look for any valid CRDs and discover them.

I wanted ot provide this through the front-end as well. However, I ran into some issues...

## CORS

Plain HTTP requests are working fine only if `raw.githubusercontent.com` is being used. That service doesn't have CORS.

However, sadly, the rest of the endpoints and locations DO require CORS to be negotiated. Which means any complex
operations won't work.

There are a couple of solutions to this out there already, sadly, none of them fit.

## Static Site

One solution is a plain CORS server running besides your own application. It's super simple. Looks like this:

```go
package main

import (
	"net/http"
	"net/http/httputil"
	"net/url"
	"time"
)

const (
	headerTimeout = 5 * time.Second
	readTimeout   = 20 * time.Second
)

func createReverseProxyServer(baseURL string) *http.Server {
	proxy := &httputil.ReverseProxy{
		Director: func(req *http.Request) {
			targetURL := req.URL.Query().Get("url")
			if targetURL == "" {
				return
			}

			target, err := url.Parse(targetURL)
			if err != nil {
				return
			}

			req.URL = target
			req.Host = target.Host
		},
	}

	handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Access-Control-Allow-Origin", baseURL)
		w.Header().Set("Access-Control-Allow-Methods", "POST, GET, OPTIONS, PUT, DELETE")
		w.Header().Set("Access-Control-Allow-Headers", "Accept, Content-Type, Content-Length, Accept-Encoding")
		w.Header().Set("Access-Control-Allow-Credentials", "true")

		if r.Method == "OPTIONS" {
			w.WriteHeader(http.StatusOK)
			return
		}

		proxy.ServeHTTP(w, r)
	})

	server := &http.Server{
		Addr:              ":8999",
		Handler:           handler,
		ReadHeaderTimeout: headerTimeout,
		ReadTimeout:       readTimeout,
	}

	return server
}
```

Then in main:

```go
	server := createReverseProxyServer(baseURL)
	go func() { _ = server.ListenAndServe() }()
	defer func() { _ = server.Shutdown(context.Background()) }()
```

And finally, when I get the URL I pass it to my service:

```go
		tag := app.Window().GetElementByID("url_tag").Get("value")
		u := "http://localhost:8999?url=" + v
```

Simple. However, I'm running a WASM app. It's a static site. It can't run a service because it's not technically running.

## No Remote CORS Service

Another options would be to use something like https://corsproxy.io/ which has a reasonable pricing as well. However, the
reasonable pricing includes only 100.000 requests / months and we all know what would happen with that. It would be
siphoned away with a bot on day one.

## No Javascript with React proxy

Finally, if it would be an angular or a react app, they have integration CORS proxies and such that alter the request
and add a CORS header before sending it out. I don't have access to that either because my app is fully in Go.

I could somehow alter the request in flight, or use a fork of go-git and alter that somehow, but that would be insanity.

Therefore, with sad heart, I'm going to drop this support from the web frontend.

However, the CLI is happily providing this option and now I also added it to the configuration.

```yaml
apiGroups:
  - name: "com.aws.services"
    description: "Resources related to AWS services"
    files: # files and folders can be defined together or on their own
      - sample-crd/infrastructure.cluster.x-k8s.io_awsclusters.yaml
      - sample-crd/delivery.krok.app_krokcommands
  - name: "com.azure.services"
    description: "Resources related to Azure services"
    folders:
      - azure-crds
  - name: "git-repositories"
    urls:
      - url: <some-url>
    gitUrls:
      - url: git@github.com:Skarlso/crd-bootstrap
      - url: git@github.com:crossplane/crossplane
```
