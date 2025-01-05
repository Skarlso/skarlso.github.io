+++
author = "hannibal"
categories = ["oras", "golang", "oci"]
date = "2025-01-06T01:01:00+01:00"
title = "Using ORAS as a library to interact with OCI repositories"
url = "/2025/01/06/using-oras-as-a-library-to-interact-with-oci-repositories"
comments = true
+++

# Introducing ORAS into a Library

This post talks about using [ORAS](https://github.com/oras-project/oras-go/) as a library to interact with OCI repositories.

First and foremost I like to keep things simple. I was seeing basic usages around the project I'm working in and some
common behaviour that started to emerge. Looking at that behaviour I drawn a preliminary interface. This interface is
also something similar that docker remote implementation has in containerd.

Looks something like this:

```go
type Resolver interface {
	// Resolve attempts to resolve the reference into a name and descriptor.
	//
	// The argument `ref` should be a scheme-less URI representing the remote.
	// Structurally, it has a host and path. The "host" can be used to directly
	// reference a specific host or be matched against a specific handler.
	//
	// The returned name should be used to identify the referenced entity.
	// Dependending on the remote namespace, this may be immutable or mutable.
	// While the name may differ from ref, it should itself be a valid ref.
	//
	// If the resolution fails, an error will be returned.
	Resolve(ctx context.Context, ref string) (name string, desc ocispec.Descriptor, err error)

	// Fetcher returns a new fetcher for the provided reference.
	// All content fetched from the returned fetcher will be
	// from the namespace referred to by ref.
	Fetcher(ctx context.Context, ref string) (Fetcher, error)

	// Pusher returns a new pusher for the provided reference
	// The returned Pusher should satisfy content.Ingester and concurrent attempts
	// to push the same blob using the Ingester API should result in ErrUnavailable.
	Pusher(ctx context.Context, ref string) (Pusher, error)

	Lister(ctx context.Context, ref string) (Lister, error)
}
```

The way this works is that the `*ers` are created during the phase where we are introducing interactions.

A Fetcher for example, looks like this:

```go
// Fetcher fetches content.
type Fetcher interface {
	// Fetch the resource identified by the descriptor.
	Fetch(ctx context.Context, desc ocispec.Descriptor) (io.ReadCloser, error)
}
```

The `ref` will be part of the Fetcher so from here, Fetch, has to only deal with the ocispec.Descriptor object. It already
knows where to fetch it from. It just needs some details.

Here is the Fetcher's struct:

```go
type OrasFetcher struct {
	client    *auth.Client
	ref       string
	plainHTTP bool
	mu        sync.Mutex
}
```

## Client

There are a number of things here. First, it's the `auth.Client`. This client is added to the fetcher when it's created.
There is the ref, there is a setting to use plainHTTP and a mutex.

Creating it is simple:
```go
func (c *Client) Fetcher(ctx context.Context, ref string) (Fetcher, error) {
	return &OrasFetcher{client: c.client, ref: ref, plainHTTP: c.plainHTTP}, nil
}
```

How is the `Client` created? Here is the entire code:
```go
	authCreds := auth.Credential{}
	if creds != nil {
		pass := creds.GetProperty(credentials.ATTR_IDENTITY_TOKEN)
		if pass == "" {
			pass = creds.GetProperty(credentials.ATTR_PASSWORD)
		}
		authCreds.Username = creds.GetProperty(credentials.ATTR_USERNAME)
		authCreds.Password = pass
	}

	client := retry.DefaultClient
	client.Transport = ocmlog.NewRoundTripper(retry.DefaultClient.Transport, logger)
	if r.info.Scheme == "https" {
		// set up TLS
		//nolint:gosec // used like the default, there are OCI servers (quay.io) not working with min version.
		conf := &tls.Config{
			// MinVersion: tls.VersionTLS13,
			RootCAs: func() *x509.CertPool {
				var rootCAs *x509.CertPool
				if creds != nil {
					c := creds.GetProperty(credentials.ATTR_CERTIFICATE_AUTHORITY)
					if c != "" {
						rootCAs = x509.NewCertPool()
						rootCAs.AppendCertsFromPEM([]byte(c))
					}
				}
				if rootCAs == nil {
					rootCAs = rootcertsattr.Get(r.GetContext()).GetRootCertPool(true)
				}
				return rootCAs
			}(),
		}
		client.Transport = ocmlog.NewRoundTripper(retry.NewTransport(&http.Transport{
			TLSClientConfig: conf,
		}), logger)
	}

	authClient := &auth.Client{
		Client: client,
		Cache:  auth.NewCache(),
		Credential: auth.CredentialFunc(func(ctx context.Context, hostport string) (auth.Credential, error) {
			if strings.Contains(hostport, r.info.HostPort()) {
				return authCreds, nil
			}
			logger.Warn("no credentials for host", "host", hostport)
			return auth.EmptyCredential, nil
		}),
	}

	return oras.New(oras.ClientOptions{
		Client:    authClient,
		PlainHTTP: r.info.Scheme == "http",
		Logger:    logger,
		Lock:      locker.New(),
	}), nil
```

Basically, we create an auth client. The `retry.DefaultClient` has logic to retry on failure. In case we have https, we
also add any custom certificates to the client provided by the user. To make sure we can call internal https registries that
require certificates.

Then, we set up an auth cache ( this is important, otherwise it's authenticating a surprising amount of times ).

Let's look at the usage.

## Usage

### Resolve

The first thing we need to do when we receive a descriptor is `Resolve` it. In the project I'm working in all I get
usually is a digest. We know where it is, but we need to gather some more information on it. Things like, the size and
the mediatype. These two are the most important bits. The mediatype tells us if it's a Manifest or a Blob. The size is
used for security purposes. If the fetched content size is different, we'll get an error.

This is all we need to do a Resolve:

```go
func (c *Client) Resolve(ctx context.Context, ref string) (string, ociv1.Descriptor, error) {
	src, err := createRepository(ref, c.client, c.plainHTTP)
	if err != nil {
		return "", ociv1.Descriptor{}, err
	}

	// We try to first resolve a manifest.
	// _Note_: If there is an error like not found, but we know that the digest exists
	// we can add src.Blobs().Resolve in here. If we do that, note that
	// for Blobs().Resolve `not found` is actually `invalid checksum digest format`.
	// Meaning it will throw that error instead of not found.
	desc, err := src.Resolve(ctx, ref)
	if err != nil {
		if errors.Is(err, oraserr.ErrNotFound) {
			return "", ociv1.Descriptor{}, errdefs.ErrNotFound
		}

		return "", ociv1.Descriptor{}, fmt.Errorf("failed to resolve manifest %q: %w", ref, err)
	}

	return "", desc, nil

```

This is quite simple. If we can't find the thing we are looking for, we throw a notFound. The library is using that error
to decide how to proceed. If it was fetching something, it will try and resolve a blob instead. If that doesn't exist
either it will tell the user that.

The resolve here, resolves a Manifest, because when we are resolving something 99% of the cases we are looking at a Manifest.
We, then, go from there to find any blobs we are looking for based on the mediatype.

### Fetch

Fetching turned out to be really interesting. Previously, we used containerd code verbatim. It's a long story and something
worth telling on it's own, but this is not the place. I wanted to replace that code with ORAS for a long time now, and
finally, I did it. So, the old code ran an integration test that lasted 50 seconds to run.

Once, I introduced ORAS, the runtime suddenly increased to 11 minutes. Here is the old code before the fix:

```go
func (c *OrasFetcher) Fetch(ctx context.Context, desc ociv1.Descriptor) (io.ReadCloser, error) {
	src, err := createRepository(c.ref, c.client, c.plainHTTP)
	if err != nil {
		return nil, fmt.Errorf("failed to resolve ref %q: %w", c.ref, err)
	}

	// oras requires a Resolve to happen before a fetch because
	// -1 or 0 are invalid sizes and result in a content-length mismatch error by design.
	// This is a security consideration on ORAS' side.
	// For more information (https://github.com/oras-project/oras-go/issues/822#issuecomment-2325622324)
	// We explicitly call resolve on manifest first because it might be
	// that the mediatype is not set at this point so we don't want ORAS to try to
	// select the wrong layer to fetch from.
	if desc.Size < 1 || desc.Digest == "" {
		rdesc, err := src.Manifests().Resolve(ctx, desc.Digest.String())
		if err != nil {
			var berr error
			rdesc, berr = src.Blobs().Resolve(ctx, desc.Digest.String())
			if berr != nil {
				// also display the first manifest resolve error
				err = errors.Join(err, berr)

				return nil, fmt.Errorf("failed to resolve fetch blob %q: %w", desc.Digest.String(), err)
			}

			reader, err := src.Blobs().Fetch(ctx, rdesc)
			if err != nil {
				return nil, fmt.Errorf("failed to fetch blob: %w", err)
			}

			return reader, nil
		}

		// no error
		desc = rdesc
	}

	// manifest resolve succeeded return the reader directly
	// mediatype of the descriptor should now be set to the correct type.
	fetch, err := src.Fetch(ctx, desc)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch manifest: %w", err)
	}

	return fetch, nil
}
```

Pretty simple. Because the library I was using wasn't always giving me a mediatype ( it wasn't always resolving... )
I had to basically "guess" where to fetch the digest from or what it was. So this looked harmless. After three days,
finally, I managed to understand that for _some_ repositories that `src.Manifests().Resolve()` if it wasn't there, took
7-9 seconds. It's a plain REST call. But somehow, it took a LOT of time for _some_ types of repositories. For ghcr this
issue was not apparent.

I made a bit more clever.

```go

func (c *OrasFetcher) Fetch(ctx context.Context, desc ociv1.Descriptor) (io.ReadCloser, error) {
	c.mu.Lock()
	defer c.mu.Unlock()

	src, err := createRepository(c.ref, c.client, c.plainHTTP)
	if err != nil {
		return nil, fmt.Errorf("failed to resolve ref %q: %w", c.ref, err)
	}

	// oras requires a Resolve to happen in some cases before a fetch because
	// -1 or 0 are invalid sizes and result in a content-length mismatch error by design.
	// This is a security consideration on ORAS' side.
	// For more information (https://github.com/oras-project/oras-go/issues/822#issuecomment-2325622324)
	//
	// To workaround, we resolve the correct size
	if desc.Size < 1 {
		if desc, err = c.resolveDescriptor(ctx, desc, src); err != nil {
			return nil, err
		}
	}

	// manifest resolve succeeded return the reader directly
	// mediatype of the descriptor should now be set to the correct type.
	reader, err := src.Fetch(ctx, desc)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch blob: %w", err)
	}

	return reader, nil
}

// resolveDescriptor resolves the descriptor by fetching the blob or manifest based on the digest as a reference.
// If the descriptor has a media type, it will be resolved directly.
// If the descriptor has no media type, it will first try to resolve the blob, then the manifest as a fallback.
func (c *OrasFetcher) resolveDescriptor(ctx context.Context, desc ociv1.Descriptor, src *remote.Repository) (ociv1.Descriptor, error) {
	if desc.MediaType != "" {
		var err error
		// if there is a media type, resolve the descriptor directly
		if isManifest(src.ManifestMediaTypes, desc) {
			desc, err = src.Manifests().Resolve(ctx, desc.Digest.String())
		} else {
			desc, err = src.Blobs().Resolve(ctx, desc.Digest.String())
		}
		if err != nil {
			return ociv1.Descriptor{}, fmt.Errorf("failed to resolve descriptor %q: %w", desc.Digest.String(), err)
		}

		return desc, nil
	}

	// if there is no media type, first try the blob, then the manifest
	// To reader: DO NOT fetch manifest first, this can result in high latency calls
	bdesc, err := src.Blobs().Resolve(ctx, desc.Digest.String())
	if err != nil {
		mdesc, merr := src.Manifests().Resolve(ctx, desc.Digest.String())
		if merr != nil {
			// also display the first manifest resolve error
			err = errors.Join(err, merr)

			return ociv1.Descriptor{}, fmt.Errorf("failed to resolve manifest %q: %w", desc.Digest.String(), err)
		}

		return mdesc, nil
	}

	return bdesc, err
}
```

With this change, the e2e test runtime went down to 40s. :Great Success:.

### Push

Push was interesting as well. Basically, if the reference contained a `Tag` we wanted to push a reference only. Not the
entire thing twice again. So, we parse the tag, figure out if it has a tag, then push ref, otherwise, push blob.

```go
...
	vers, err := ociutils.ParseVersion(ref.Reference)
	if err != nil {
		return fmt.Errorf("failed to parse version %q: %w", ref.Reference, err)
	}

	if vers.IsTagged() {
		// Once we get a reference that contains a tag, we need to re-push that
		// layer with the reference included. PushReference then will tag
		// that layer resulting in the created tag pointing to the right
		// blob data.
		if err := repository.PushReference(ctx, d, reader, c.ref); err != nil {
			return fmt.Errorf("failed to push tag: %w", err)
		}

		return nil
	}

	ok, err := repository.Exists(ctx, d)
	if err != nil {
		return fmt.Errorf("failed to check if repository %q exists: %w", ref.Repository, err)
	}

	if ok {
		return errdefs.ErrAlreadyExists // special error handling and cache usage
	}

	if err := repository.Push(ctx, d, reader); err != nil {
		return fmt.Errorf("failed to push: %w, %s", err, c.ref)
	}
...
```

There are certain blob caches all around the code as well. But I spare you the details of that.

### List


Finally, tag list is pretty simple:

```go
func (c *OrasLister) List(ctx context.Context) ([]string, error) {
	src, err := createRepository(c.ref, c.client, c.plainHTTP)
	if err != nil {
		return nil, fmt.Errorf("failed to resolve ref %q: %w", c.ref, err)
	}

	var result []string
	if err := src.Tags(ctx, "", func(tags []string) error {
		result = append(result, tags...)

        return nil
	}); err != nil {
        return nil, fmt.Errorf("failed to list tags: %w", err)
	}

	return result, nil
}
```

And that's it.

# Finally

And with that, we can interact with OCI repositories using ORAS. Push and Fetch blobs and Manifests. Return them, read them
and so on. The entire code is here: https://github.com/open-component-model/ocm/tree/main/api/tech/oras It's barely 300
lines of code or so. I really like it.

That's it. Thanks for reading.
