+++
author = "hannibal"
categories = ["books"]
date = "2021-12-21T01:01:00+01:00"
title = "Providers Pattern"
url = "/2021/12/21/providers-pattern"
comments = true
+++

Hello Dear readers.

Today, I would like to write about a project design pattern I've been using successfully over the past years for
various projects. It has many variations and it has some design patterns that are commonly found in the wild, so there
is nothing really special about it.

Let's begin.

# Providers Pattern

What is this pattern anyways? It's a pattern I learned while working at [ArangoDB](https://www.arangodb.com/). It's
quite nice and defines package abstractions wonderfully. It somewhat resembles the Repository pattern from DDD and also
uses Chain of Responsibility to setup multiple providers for a given functionality. Like a fallback, in case a Provider
does not understand the current thing it got. In that case, it will delegate to `Next`.

I'm going to demonstrate all of the pattern's capabilities through a sample project which is hopefully a sensible thing
and not just dummy functionality.

## TL;DR

A Provider is like the [Repository pattern](https://martinfowler.com/eaaCatalog/repository.html) from DDD combined with
[Chain Of Responsibility](https://en.wikipedia.org/wiki/Chain-of-responsibility_pattern). Basically, set up a Provider
using an `interface` as a definition. Give that as dependency to another provider and call it's function. If the
Provider doesn't understand the type its supposed to work on, it will call `Next` in the chain delegating the function
to the following Provider. For a detailed use of this pattern, check out
[Providers Example](https://github.com/Skarlso/providers-example) project on GitHub.

## The Project

[Providers Example](https://github.com/Skarlso/providers-example) is the project I'll be using to demonstrate this
pattern. It's pretty simple; yet, I hope, it presents a useful function to be show off this pattern's capabilities.

In essence, this is a plugin executor. We have a CLI which can register plugins to execute. These plugins can either be
bare metal (as in a simple executable living somewhere), or a docker container, in which case it will use Docker to
execute the plugin. It will forward all possible parameters and display any outputs.

Simple, yet there are a couple things that we can extract into Providers such as:

1. ~~Dealing with the archive ( so we can test the Tar function )~~ (sorry, I scrapped this, to keep things simple)
2. Selecting the executing environment ( bare metal, container ) which we can chain
3. Output formatting ( possibly, thing like, JSON, Table, etc. )
4. Saving things into a Database ( we will save what kind of plugins exist using sqlite )
    4.1. We'll just save the name and the type for simplicity

## Basics

Let's take a look at the folder structure here.

```
➜  providers-example git:(main) tree
.
├── LICENSE
├── Makefile
├── README.md
├── bin
├── cmd
│   ├── add.go
│   ├── list.go
│   ├── remove.go
│   ├── root.go
│   └── run.go
├── example
│   └── echo_plugin
│       └── Dockerfile
├── go.mod
├── go.sum
├── main.go
├── pkg
│   ├── commands
│   │   └── add.go
│   ├── models
│   │   └── plugin.go
│   └── providers
│       ├── archiver.go
│       ├── bare
│       │   └── bare_runner.go
│       ├── container
│       │   ├── container_runner.go
│       │   └── container_runner_test.go
│       ├── fakes
│       │   └── fake_storer_client.go
│       ├── runner.go
│       ├── storage.go
│       ├── storer
│       │   └── storer.go
│       └── tar
│           ├── tar.go
│           ├── tar_test.go
│           └── testdata
│               └── test.tar.gz
└── tests
    └── integration
        ├── init_test.go
        └── plugin_store_test.go

18 directories, 29 files
```

What are we looking at? We have some standard packages, like `pkg`, `cmd` and `tests`. What's "new" is `pkg/providers`.
I ended up not using archiver, but I left it in for prosperity. There are 3 interfaces under `providers`. Each support
a simple functionality. Let's go over them.

```go
// Archiver can extract files from an archive.
type Archiver interface {
	Untar(content []byte) ([]byte, error)
}
```

My initial plan was to implement the bare metal as an ability to download an archive and then untar that. But I scrapped
that idea for a simpler one. This provider would have been an `untarer`.

Next up is storage.

```go
// ListOpts defines options for listing plugins.
type ListOpts struct {
	TypeFilter string
}

// Storer can store information about the plugins that were created.
//go:generate go run github.com/maxbrunsfeld/counterfeiter/v6 -generate
//counterfeiter:generate -o fakes/fake_storer_client.go . Storer
type Storer interface {
	Init() error
	Create(ctx context.Context, plugin *models.Plugin) error
	Get(ctx context.Context, name string) (*models.Plugin, error)
	Delete(ctx context.Context, name string) error
	List(ctx context.Context, opts ListOpts) ([]*models.Plugin, error)
}
```

A simple CRUD interface, with an Init action. Whatever that init is. Currently, I'm using sqlite3, so the live store init
is simply creating the database file and bootstrapping it with the `plugins` table. I'm using counterfeiter to generate
a mock which I'll be using in tests to mock this provider where I don't care about it.

And the runner...

```go
// Runner runs a plugin.
type Runner interface {
	Run(ctx context.Context, name string, args []string) error
}
```

This is the single method a runner needs to be able to provide / do. A runner simply takes a name and some arguments
and based on the plugin's type, runs it.

These interfaces are then implemented by real providers. For example, the storage has its implementation under `storer`.
Which has code for sqlite. The runner has two implementations. One for container and one for bare metal. And now, let's
see how the chain works.

## Dependency Injection and Usage

Take a closer look at the container implementation for example.

```go
// Config defines parameters for the Runner.
type Config struct {
	DefaultMaximumCommandRuntime int
}

// Dependencies defines the provider dependencies this provider has.
type Dependencies struct {
	Next   providers.Runner
	Storer providers.Storer
    Logger zerolog.Logger
}

// Runner implements the Run interface for container based runtimes.
type Runner struct {
	Config
	Dependencies

	cli    client.APIClient
}

// NewRunner creates a new container based runtime.
func NewRunner(cfg Config, deps Dependencies) (*Runner, error) {
	cli, err := client.NewClientWithOpts(client.FromEnv)
	if err != nil {
		deps.Logger.Debug().Err(err).Msg("Failed to create docker client.")
		return nil, err
	}
	return &Runner{
		Config:       cfg,
		Dependencies: deps,
		cli:          cli,
	}, nil
}
```

This is the main structure of a provider. It gets Configuration options via the `Config` struct. Things like,
`DefaultMaximumCommandRuntime` which is a configuration value for maximum wait time. And it gets dependencies, other
providers and third party things, like logger, through the `Dependencies` struct. We also wire things up that we are
going to use later on, like a client for Docker. Later on, in the tests, I can inject a mock for it.

Notice the field in the `Dependencies` struct called `Next`. This is the chain's first stop. In `root.go` we wired up
next to be the bare metal type plugin like this:

```go
	barePlugin := bare.NewBareRunner(bare.Config{}, bare.Dependencies{
		Logger: log,
		Storer: store,
	})
	containerPlugin, err := container.NewRunner(container.Config{
		DefaultMaximumCommandRuntime: 15,
	}, container.Dependencies{
		Storer: store,
		Next:   barePlugin,
		Logger: log,
	})
```

And then, when calling `Run` we check the type of the plugin we got, and if we know it, we execute it, if not, we call
the `Next` thing in the chain. Simple.

```go
// Run implements the container based runtime details, using Docker as an engine.
func (cr *Runner) Run(ctx context.Context, name string, args []string) error {
	// Find the plugin, get the location, find the type, if it's not container, call next.
	cmd, err := cr.Storer.Get(ctx, name)
	if err != nil {
		return fmt.Errorf("plugin not found: %w", err)
	}
	if cmd.Type != models.Container {
		cr.Logger.Info().Msg("Unknown plugin type, calling next in line.")
		if cr.Next == nil {
			return fmt.Errorf("no next provider configured")
		}
		return cr.Next.Run(ctx, name, args)
	}
	if err := cr.runCommand(cmd.Name, cmd.Container.Image, args); err != nil {
		return fmt.Errorf("failed to run command: %w", err)
	}
	return nil
}
```

And that's it! But wait a second. How is this different from just calling the thing? Well, this makes it so that in the
actual code, we only have to call the top level element in the chain and call `Run` on it. A default if you will.
By default, our plugins are of `container` type. This way, you can avoid a type switch for example. Or avoid having
implementation choosing logic in the code when deciding what runner to use.

```go
	if err := containerPlugin.Run(context.Background(), runArgs.name, runArgs.args); err != nil {
		log.Error().Err(err).Msg("Failed to run plugin")
		os.Exit(1)
	}
```

I'm just calling `containerPlugin`'s Run and the rest will just work. With generics coming in 1.18 this will be even
more easier to use.

## Testability

The great thing about these providers are that they enclose ( you might say, encapsulate ) a single functionality. They
provide some kind of behavior to another behavior. Because that behavior is defined by an interface, and the dependency
requires an interface, it's trivial to unit test the actual provider. For example, the storage. You see above, that the
storage is defined as `Storer providers.Storer`. And we already have a fake. So in the tests we can set it up such as:

```go
    fakeStorer := &fakes.FakeStorer{}
    fakeStorer.GetReturns(&models.Plugin{
        ID:   1,
        Name: "test",
        Type: models.Container,
        Container: &models.ContainerPlugin{
            Image: "test-image",
        },
    }, nil)
...
	r := Runner{
		Dependencies: Dependencies{
			Storer: fakeStorer,
			Logger: logger,
		},
		Config: Config{
			DefaultMaximumCommandRuntime: 15,
		},
		cli: apiClient,
	}
```

After that, we can use it to return errors and various bits and pieces to test our actual plugin runner and not care
about storage.

## Pros and Cons

The pros are obvious. Separation, testability, chaining, fallback logic, encapsulated behavior, etc.

But what about the cons? The downside could be the following things:

- naming gets hard and packages can get prolific
- interfaces will be binding, changing them might result in changing a bunch of implementations
- some implementations might not use some parameters from the interface if they don't provide that functionality ( this
might be mitigated by a very restrictive interface design )
- it's a bit verbose ( this can be mitigated by not calling it dependencies or passing in a single struct instead of two)
- wiring up all the dependencies can lead to a long set up function if there are several providers ( this can be
mitigated by having defaults and smaller setup functions )

Still, I think the benefits easily out weight these problems. And many other package designs have similar issues.

## Other providers

This is just an example. Another couple of examples include:

- fallback authentication; start with Auth0, then SSH, username/password, finally localhost and fail
- output parsing; what does the struct provide, use that, otherwise, fallback to table
- github, gitlab, bitbucket handlers chained up and chosen based on url types

## Conclusions

This is it. That's the provider pattern. It's a useful little tool in the box to have. With generics, the choosing type
will change, but the pattern can adapt to that too. It's a nice little thing to have. Of course it's not a solution to
all the problems ever. But I've been using it for a while now, to good effect.

Thanks for reading,

Gergely.
