+++
author = "hannibal"
categories = ["golang", "functional-options"]
date = "2024-07-01T01:01:00+01:00"
title = "Discoverable functional options pattern"
url = "/2024/07/01/discoverable-functional-options"
comments = true
+++

Hello.

Today's will be a quick post. Everyone knows and loves/hates functional options[^1] in Go.

The biggest gripe people get with it is, that the options aren't discoverable and that there
is no IDE support for nicely auto-completing options.

My thought about this was that, what if we would just hang it on a struct? Let's see how that
looks.

Consider this normal server builder with options:

```go
type Server struct {
	Name    string
	Address string
	Port    int
}

func WithName(name string) ServerOptFn {
	return func(s *Server) {
		s.Name = name
	}
}

func WithAddress(address string) ServerOptFn {
	return func(s *Server) {
		s.Address = address
	}
}

func WithPort(port int) ServerOptFn {
	return func(s *Server) {
		s.Port = port
	}
}

type ServerOptFn func(*Server)

func NewServer(opts ...ServerOptFn) *Server {
	s := &Server{}

	for _, o := range opts {
		o(s)
	}

	return s
}
```

Now, what if you would like to retain the niceness of the clean options pattern where you
don't have to specify and empty struct but still could use a struct to gather the options
together?

```go
type Server struct {
	Name    string
	Address string
	Port    int
}

type ServerOpts struct{}

func (ServerOpts) WithName(name string) ServerOptFn {
	return func(s *Server) {
		s.Name = name
	}
}

func (ServerOpts) WithAddress(address string) ServerOptFn {
	return func(s *Server) {
		s.Address = address
	}
}

func (ServerOpts) WithPort(port int) ServerOptFn {
	return func(s *Server) {
		s.Port = port
	}
}

type ServerOptFn func(*Server)
```

Now, all the options are on the `ServerOpts` struct.

And then calling the builder would be something like this:

```go
	server := pkg.NewServer(
		pkg.ServerOpts{}.WithAddress("address"),
		pkg.ServerOpts{}.WithName("name"),
		pkg.ServerOpts{}.WithPort(9998))

	server.Start()
```

To make it nicer, create a variable:

```go
	serverOpts := pkg.ServerOpts{}
	server := pkg.NewServer(
		serverOpts.WithAddress("address"),
		serverOpts.WithName("name"),
		serverOpts.WithPort(9998))

	server.Start()
```

That's not half that bad, I recon. And looking at the docs, we can see that the options are nicely
put on the [ServerOpts](Skarlso/discoverable-functional-options) struct therefore they are discoverable.

# Conclusion

What do you think? Yay or nay?

Thanks for reading!


[^1]: https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis